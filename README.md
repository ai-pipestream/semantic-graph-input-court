# &lt;step&gt;-input-court

Per-step **input** fixtures for the CourtListener corpus. Each `<step>-input-court`
repo holds the 1000 court opinions **as they enter `<step>`** — i.e. the output of
the previous stage — one gzipped `ai.pipestream.data.v1.PipeDoc` per source doc.

These are the drop-in feed for **stress-testing and regression-testing `<step>`**:
push each doc through `<step>`'s in-VM `process-once` door at scale, or diff against
a known-good run.

| repo | captures (engine node) | content | feeds / stresses |
|------|------------------------|---------|------------------|
| `chunker-input-court`        | `chunker`         | raw text                | chunker |
| `embedder-input-court`       | `embedder`        | + chunks (no vectors)   | embedder |
| `semantic-graph-input-court` | `semantic-graph`  | + embedding vectors     | semantic-graph |
| `opensearch-sink-input-court`| `opensearch-sink` | + semantic graph data   | opensearch-sink |

## Layout

```
src/main/resources/fixtures/court/<step>/doc_NNNN.pb.gz   # 1000 files
src/main/resources/META-INF/pipestream-sample-data.json  # descriptor (count, crawl, pairing)
src/main/resources/fixtures/court/index_<step>.tsv       # doc_NNNN -> doc_id
```

- One file = one `PipeDoc` serialized with `writeTo(...)` then gzipped. Read with:
  ```java
  try (var in = getClass().getResourceAsStream("/fixtures/court/<step>/doc_0001.pb.gz");
       var gz = new java.util.zip.GZIPInputStream(in)) {
      PipeDoc doc = PipeDoc.parseFrom(gz.readAllBytes());
  }
  ```
- `doc_NNNN` is a stable index ordered by `doc_id`; the **same NNNN refers to the
  same source doc across every `<step>-input-court` repo** (pairing), so you can
  line up a doc's shape at each stage.

## Maven coordinate

`ai.pipestream:<repo-name>:<version>` (artifactId = repo name; version from git tag).

## How the data is (re)generated — not by hand

Always from a **from-scratch** full-persist e2e crawl. The control scripts live in
[`_template-input-court`](https://github.com/ai-pipestream/_template-input-court):

```bash
./refresh-all-court.sh           # clone+scaffold missing step repos, run e2e, harvest all
./refresh-all-court.sh --commit  # + commit each repo
SKIP_E2E=1 ./refresh-all-court.sh  # re-harvest the latest existing crawl (debug only)
```

`harvest-court.sh` pulls each step's captured PipeDocs out of repo-service/SeaweedFS
(grouped by the crawl's `graph_address_id`) and writes them here. This repo is a
**generated artifact** — don't hand-edit the fixtures.
