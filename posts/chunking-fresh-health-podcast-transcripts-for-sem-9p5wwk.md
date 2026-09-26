# Chunking Fresh Health Podcast Transcripts for Semantic Search (After Page Changes)

Use timestamped transcript cues as the durable source, build chunks on cue boundaries, and reindex only after the watched page points to changed audio. That rule keeps freshness work proportional to a real content change instead of every cosmetic page edit.

| Choice | Chunk boundary | Freshness signal | Best fit | Main cost |
|---|---|---|---|---|
| Timestamped cues | Speaker or time cue | Audio identity changed | Search results that must jump to evidence | More metadata to preserve |
| Fixed token windows | Token count | Any transcript changed | Fast prototypes and uniform embedding batches | Boundaries can split an answer |
| Topic segmentation | Semantic shift | Transcript changed | Long, loosely structured discussions | More tuning and unstable boundaries |

**TL;DR:** For a healthtech podcast watcher, choose timestamped cues, merge them into bounded chunks, and assign IDs from normalized text plus the episode ID. Store the page validator, audio fingerprint, transcript version, chunk hash, and timestamps separately. A changed page should trigger inspection; only changed audio should trigger transcription. Only changed chunk hashes should trigger embedding and index writes.

This is the boring path.

Good.

A one-person SaaS earns nothing from retranscribing an unchanged 48-minute episode because a producer edited its show notes. I would outsource speech recognition and embedding generation behind small interfaces, then own the change detection and chunk identity. Those two decisions determine freshness, citation quality, and most of the avoidable compute.

## Why doesn't every page change require a new transcript?

A podcast page is a container, not the audio. Its title, sponsor copy, publication date, transcript link, and media URL can change independently. Treating the HTML body as one freshness key couples all of those fields and creates needless downstream work.

HTTP already provides conditional request semantics. A watcher can retain an `ETag` and `Last-Modified` value, send `If-None-Match` or `If-Modified-Since` on the next request, and accept `304 Not Modified` as evidence that the selected representation has not changed [1][2]. These validators reduce transfers. They do not prove that the linked audio is the same, because the page and media object are separate resources.

So the page fetch is a gate. After a changed response, parse the canonical episode ID and audio URL, then compare the media identity with the last successful run. Prefer a strong media validator when the host supplies one. Otherwise, download the object and compute a cryptographic digest. URL equality alone is weak: a publisher may replace the bytes at the same path, or move identical bytes to a new path.

This distinction matters in health content. Consider an episode page whose show notes change twice on Tuesday: the first edit fixes a guest's job title, while the second points the existing media URL to corrected audio. The first observation should update page metadata and stop. The second must hash the fetched audio, create a transcript revision, validate its cues, embed changed chunks, and switch the active revision only after every write succeeds. A corrected dosage statement or newly inserted disclaimer deserves that visible version boundary. A CSS class rename does not. The watcher needs enough state to tell those cases apart without guessing from the page's update date.

Freshness is narrower than recency.

Keep an append-only observation record even when no indexing work follows:

| Field | Purpose |
|---|---|
| `observedAt` | Establish when the watcher checked the source |
| `pageEtag` and `pageLastModified` | Drive the next conditional request |
| `audioUrl` and `audioSha256` | Separate a media change from a page edit |
| `transcriptRevision` | Tie chunks to one transcription result |
| `chunkIds` | Make index deletion and replacement explicit |

Do not let a failed refresh erase the last searchable revision. Publish a new revision only after transcription, chunk validation, embedding, and index writes succeed. Until then, the prior revision remains current and the observation can carry a failure state for retry. Freshness without atomic publication is just a race with a nicer name.

## Chunking is a retrieval decision, not a text-cleaning step

Speech recognizers commonly return timestamped segments or words. Preserve them. WebVTT also models timed text as cues with start and end times, which makes it a useful interchange shape even if the final search index stores JSON [3]. Flattening the transcript into one string throws away the cheapest route from a search hit back to the audio.

The first chunk boundary should follow a cue, speaker turn, or sentence ending. The size limit comes second. For an initial implementation, I would target a range rather than a magic count: merge adjacent cues until the chunk is large enough to carry context, stop before it becomes a miniature episode, and retain a small overlap only when the previous chunk ends mid-thought. Measure tokens with the tokenizer used by the embedding model before sending the request. Character counts are merely a local guardrail.

Short chunks sharpen lexical focus but can omit the qualification that makes a medical statement accurate. Large chunks preserve context but return more irrelevant text and make timestamp links vague. That trade-off is why chunk size belongs in the index schema as a versioned policy, not as an unrecorded constant.

There is another trap: positional IDs such as `episode-17-chunk-9`. Insert one corrected sentence near the beginning and every later position can move. The index then sees a full replacement even when most text survived. A content-derived ID limits churn. Hash the normalized chunk text with a stable episode identifier, while keeping start and end time as mutable metadata. If duplicate refrains are possible, add the first cue timestamp or an occurrence counter to disambiguate them.

**The indexable unit is evidence:** text, episode revision, start time, end time, source URL, language, and a stable content hash. An embedding without those fields can retrieve a plausible passage but cannot support a precise result link or a clean refresh.

## How should a Python pipeline transcribe audio, chunk the transcript, and index it?

The example below leaves speech recognition, token counting, embedding, and vector storage behind interfaces. That is deliberate. The differentiated work is the revision contract and chunk policy, while model calls are replaceable infrastructure.

The same pipeline contract applies in Python; the copyable implementation is TypeScript so the types expose each boundary.

```ts
import { createHash } from "node:crypto";

type Cue = {
  startMs: number;
  endMs: number;
  text: string;
  speaker?: string;
};

type Chunk = {
  id: string;
  episodeId: string;
  transcriptRevision: string;
  startMs: number;
  endMs: number;
  text: string;
  sourceUrl: string;
};

interface Transcriber {
  transcribe(audio: Uint8Array): Promise<Cue[]>;
}

interface Embedder {
  embed(texts: string[]): Promise<number[][]>;
}

interface SearchIndex {
  replaceRevision(input: {
    episodeId: string;
    previousRevision?: string;
    chunks: Array<Chunk & { vector: number[] }>;
  }): Promise<void>;
}

const normalize = (text: string): string =>
  text.replace(/\s+/g, " ").trim();

const digest = (value: string): string =>
  createHash("sha256").update(value).digest("hex");

function chunkCues(
  episodeId: string,
  transcriptRevision: string,
  sourceUrl: string,
  cues: Cue[],
  maxCharacters = 1_800,
): Chunk[] {
  const chunks: Chunk[] = [];
  let pending: Cue[] = [];

  const flush = (): void => {
    if (pending.length === 0) return;

    const text = normalize(pending.map((cue) => cue.text).join(" "));
    const startMs = pending[0].startMs;
    const endMs = pending[pending.length - 1].endMs;
    const occurrenceKey = `${episodeId}:${startMs}:${text}`;

    chunks.push({
      id: digest(occurrenceKey),
      episodeId,
      transcriptRevision,
      startMs,
      endMs,
      text,
      sourceUrl,
    });
    pending = [];
  };

  for (const cue of cues) {
    const candidate = normalize(
      [...pending.map((item) => item.text), cue.text].join(" "),
    );
    if (pending.length > 0 && candidate.length > maxCharacters) flush();
    pending.push(cue);
  }
  flush();
  return chunks;
}

async function refreshEpisode(input: {
  episodeId: string;
  sourceUrl: string;
  audio: Uint8Array;
  previousRevision?: string;
  transcriber: Transcriber;
  embedder: Embedder;
  index: SearchIndex;
}): Promise<string> {
  const audioHash = createHash("sha256").update(input.audio).digest("hex");
  const revision = `sha256:${audioHash}`;
  if (revision === input.previousRevision) return revision;

  const cues = await input.transcriber.transcribe(input.audio);
  if (cues.length === 0) throw new Error("Transcription returned no cues");

  const chunks = chunkCues(
    input.episodeId,
    revision,
    input.sourceUrl,
    cues,
  );
  const vectors = await input.embedder.embed(chunks.map((chunk) => chunk.text));
  if (vectors.length !== chunks.length) {
    throw new Error("Embedding count does not match chunk count");
  }

  await input.index.replaceRevision({
    episodeId: input.episodeId,
    previousRevision: input.previousRevision,
    chunks: chunks.map((chunk, index) => ({ ...chunk, vector: vectors[index] })),
  });
  return revision;
}
```

The `1_800`-character ceiling is an example guardrail, not a universal optimum. It is intentionally visible so a test corpus can challenge it. In production, the chunker should also reject reversed timestamps, sort or reject out-of-order cues, normalize Unicode consistently, and enforce the embedding model's token limit. The index operation must behave atomically from the reader's perspective: upsert the new revision, switch the active revision, then remove stale records, or use an equivalent transaction or alias swap.

Ship the first version with a small fixture set. One episode should contain a correction near the start, one should repeat the same sentence at two timestamps, one should include a very long cue, and one should produce no speech. Those four cases catch more useful failures than a large happy-path transcript.

## Freshness needs observable states

A scheduled job should report decisions, not merely “success.” Record whether the page was unchanged, the page changed but audio did not, audio changed and indexing completed, or refresh failed while the previous revision stayed active. This creates a short path from an alert to the responsible stage.

Track counts and ages: last successful page check, active transcript age, cues produced, chunks produced, unchanged chunk hashes, embedding requests, and stale chunks removed. Avoid using transcript age alone as an alarm. A monthly show can be healthy with a month-old transcript; a recently changed media object with an old active revision is the actionable mismatch.

Retrieval evaluation belongs beside freshness monitoring. Keep a compact set of real questions with expected episode IDs and approximate timestamp ranges. Run it whenever the chunk policy, transcription model, or embedding model changes. Retrieval-augmented generation joins a retriever with downstream generation [4], but this system should evaluate retrieval before judging generated prose. If the correct evidence never enters the candidate set, generation cannot repair that omission reliably.

Privacy boundaries also affect the pipeline. Public podcast audio may still contain names, symptoms, or personal accounts. Store only the fields needed for search, define retention for raw audio and intermediate transcripts, and separate access to operational logs from access to transcript content. Do this before adding more feeds. Retrofitting deletion across blobs, queues, embeddings, and backups is expensive.

## When the runner-up choices are better

Fixed token windows are reasonable for a disposable proof of concept, especially when the source has no timestamps and the immediate question is whether semantic retrieval works at all. They are easy to batch and easy to reproduce. Add source offsets from day one, though, or migration to playable timestamps becomes a reconstruction project.

Topic segmentation earns its extra machinery for long panel discussions that shift subjects without clear speaker or chapter boundaries. It can keep one clinical subject together even when the conversation runs beyond a simple size target. The downside is operational: a model or threshold change may redraw many boundaries, invalidate IDs, and force broad re-embedding. Version that policy and compare it against a fixed evaluation set before publishing the new index.

For a weekly shipping cadence, I would start with cue-aware bounded chunks. The decision can change when measured retrieval misses show a boundary problem. It should not change because a more elaborate chunker sounds sophisticated.

The final rule is compact: inspect on page change, transcribe on media change, embed on chunk change, and publish only a complete revision. That gives a health podcast search system fresh evidence without turning every editorial tweak into a pipeline-wide bill.

## References

1. https://www.rfc-editor.org/rfc/rfc9110.html
2. https://developer.mozilla.org/en-US/docs/Web/HTTP/Conditional_requests
3. https://www.w3.org/TR/webvtt1/
4. https://arxiv.org/abs/2005.11401
