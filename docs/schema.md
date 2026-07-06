# Annotation JSON Schema v1.0

This document defines the source-of-truth data format produced by `annotation-tool.html` and consumed by `result-viewer.html` and downstream analysis tools.

## Design principles

1. **Structured event stream is the source of truth.** Images and text reports are derived views.
2. **Error is the atomic unit of analysis**, not strokes or images. Each error has exactly one bounding box.
3. **Coordinates live in image-pixel space** (`naturalWidth × naturalHeight`), not screen space.
4. **`badcase_count` is auto-derived** from `errors.length`. Never stored as an independent override.
5. **`paper` is the granularity of identification** (v2+). A 卷 (paper) = one subfolder = one YAML = 1+ images sharing the same `paper_id`. Question lists and per-question judgments live paper-level, not image-level.

## File layout in exported ZIP

```
批阅标注结果_<timestamp>.zip
├── _session.json              # session index (annotator-level)
├── _stats.jsonl               # derived: one row per error, for analysis
├── taxonomy.json              # snapshot of taxonomy used during this session
└── <paper_id>/                # paper_id = first task_id from the YAML
    ├── paper.json             # paper-level truth: questions, judgments, image refs
    ├── page_1/
    │   ├── annotations/
    │   │   └── default.json   # image-level errors + paper_id reference (NO judgments)
    │   └── source.<ext>
    └── page_2/
        ├── annotations/
        │   └── default.json
        └── source.<ext>
```

For multi-annotator (v2+), `annotations/` contains `<annotator_id>.json` per annotator and optionally `review.json` for adjudication.

### Legacy v1 layout (still readable by viewer)

```
批阅标注结果_<timestamp>.zip
└── <task_id>/
    ├── annotations/default.json
    └── source.<ext>
```

The viewer detects `paper.json` at the task-folder root. If absent, it falls back to the legacy single-image path.

## `paper.json` (v2+ — paper-level truth source)

```json
{
  "schema_version": "1.0",
  "paper_id": "db45c6d8-04ed-4a71-a7d2-2179957bd9b4",
  "image_count": 2,
  "questions": [
    { "question_no": "1" },
    { "question_no": "1(1)" },
    { "question_no": "1(2)" }
  ],
  "judgments": [
    { "question_no": "1(1)", "status": "correct" },
    { "question_no": "1(2)", "status": "wrong" }
  ],
  "identified_at": "2026-07-06T10:00:00.000Z",
  "vlm_model_id": "doubao-1.5-vision-pro-32k-250115",
  "images": [
    {
      "page_index": 0,
      "task_id": "db45c6d8-04ed-4a71-a7d2-2179957bd9b4",
      "source_path": "未匹配/1/1.jpg",
      "source_hash": "sha256:9f2c1a...",
      "status": "annotated",
      "error_count": 2,
      "annotation_file": "db45c6d8-.../page_1/annotations/default.json"
    },
    {
      "page_index": 1,
      "task_id": "db45c6d8-04ed-4a71-a7d2-2179957bd9b4",
      "source_path": "未匹配/1/2.jpg",
      "source_hash": "sha256:8e1b0b...",
      "status": "no_badcase",
      "error_count": 0,
      "annotation_file": "db45c6d8-.../page_2/annotations/default.json"
    }
  ]
}
```

### Field semantics

| Field | Type | Required | Notes |
|---|---|---|---|
| `paper_id` | string | ✓ | UUID from YAML `task_ids[0]`. Same as `task_id` for v1. |
| `image_count` | int | ✓ | Number of pages belonging to this paper (1+). |
| `questions` | array | ✓ | Sub-question numbers identified by VLM. Empty if VLM not configured or returned nothing. Each item: `{question_no: string}`. |
| `judgments` | array | ✓ | User-per-question verdicts. Empty if no verdicts recorded. Each item: `{question_no, status}` where status ∈ `correct|wrong|unmarked`. |
| `identified_at` | ISO 8601 \| null | ✓ | When VLM succeeded for this paper. `null` if VLM not configured. |
| `vlm_model_id` | string \| null | ✓ | Model ID used for identification. `null` if VLM not configured. |
| `images` | array | ✓ | One entry per page (matches `image_count`). Sorted by `page_index`. |

## `<paper_id>/page_N/annotations/<annotator_id>.json`

```json
{
  "schema_version": "1.0",
  "image": {
    "task_id": "db45c6d8-04ed-4a71-a7d2-2179957bd9b4",
    "paper_id": "db45c6d8-04ed-4a71-a7d2-2179957bd9b4",
    "page_index": 0,
    "source_path": "未匹配/1/1.jpg",
    "source_hash": "sha256:9f2c1a...",
    "width": 2480,
    "height": 3508,
    "metadata": {
      "task_ids": ["db45c6d8-04ed-4a71-a7d2-2179957bd9b4"]
    }
  },
  "annotation": {
    "status": "annotated",
    "errors": [
      {
        "error_id": "err_01",
        "error_type": "ocr",
        "error_subtype": "char_wrong",
        "severity": null,
        "comment": "把 7 识别成 1",
        "marks": [
          {
            "mark_id": "m_01",
            "role": "primary",
            "type": "bbox",
            "geometry": {
              "bbox": [120, 338, 80, 42],
              "points": null
            },
            "color": "#3498db",
            "width": 2
          }
        ],
        "annotator_id": "default",
        "created_at": "2026-06-29T10:00:00.123Z",
        "updated_at": "2026-06-29T10:00:04.456Z",
        "duration_ms": 3200
      }
    ],
    "session_id": "sess_20260629_001",
    "annotator_id": "default",
    "started_at": "2026-06-29T09:58:00.000Z",
    "saved_at": "2026-06-29T10:02:00.000Z",
    "total_duration_ms": 240000,
    "client": {
      "tool_version": "2.0.0",
      "browser": "Mozilla/5.0..."
    }
  }
}
```

### Field semantics

| Field | Type | Required | Notes |
|---|---|---|---|
| `schema_version` | string | ✓ | `"1.0"`. Bump on breaking changes. |
| `image.task_id` | string | ✓ | UUID from `metadata.yaml`. Same as `paper_id` for v2+ data (kept for legacy compat). |
| `image.paper_id` | string | ✓ (v2+) | The paper (subfolder) this image belongs to. Equals the YAML's first task_id. New in v2; viewer backfills with `task_id` if absent. |
| `image.page_index` | int | ✓ (v2+) | 0-based position within the paper. New in v2; viewer backfills with 0 if absent. |
| `image.source_path` | string | ✓ | Relative path within original folder. |
| `image.source_hash` | string \| null | ✓ | `sha256:<hex>`. `null` if hash computation failed (file:// + SubtleCrypto unavailable). |
| `image.width`, `image.height` | int \| null | ✓ | Natural pixel dimensions of source image. `null` when the tool serialized before the image finished loading (e.g. file:// + Safari slow load). Consumers should handle null gracefully. |
| `image.metadata` | object | ✓ | Pass-through from `metadata.yaml`. Always includes `task_ids`. |
| `annotation.status` | enum | ✓ | One of: `pending`, `annotated`, `no_badcase`, `skipped`. `no_badcase` = reviewed, no errors found (added v1.0, replaces most uses of `skipped`). `skipped` = legacy "user clicked skip", kept for backward compat. v2 adds `reviewed`, `adjudicated`. |
| `annotation.errors` | array | ✓ | Empty array if status is `no_badcase`, `skipped`, or `pending`. |
| ~~`annotation.judgments`~~ | ~~array~~ | ✗ | **Deprecated in v2.** Per-sub-question judgments have moved to paper-level (`paper.json`). Old v1 files may still have this field; the viewer ignores it. New exports do NOT write it. |
| `errors[].error_id` | string | ✓ | Unique within this file. Format: `err_NN`. |
| `errors[].error_type` | string | ✓ | References `taxonomy.categories[].id`. |
| `errors[].error_subtype` | string \| null | | References `taxonomy.categories[].subtypes[].id`. |
| `errors[].severity` | int \| null | | 1-3. v1 not exposed in UI; always `null` for new data. |
| `errors[].comment` | string | | Free text. May be empty string. |
| `errors[].marks` | array | ✓ | May be empty for comment-only errors added via the viewer (no bbox drawn). When non-empty, exactly one mark must have `role: "primary"`. |
| `marks[].role` | enum | ✓ | `primary` (counted) or `note` (decorative, not counted). |
| `marks[].type` | enum | ✓ | `bbox` (v1 only). Reserved: `point`, `arrow`, `stroke`. |
| `marks[].geometry.bbox` | [x,y,w,h] \| null | | Required when `type === "bbox"` AND mark exists. `null` only for migrated historical data where location was rasterized. Image-pixel coords. |
| `marks[].color` | string | | Hex color from taxonomy category. |
| `errors[].annotator_id` | string | ✓ | `"default"` in v1. |
| `errors[].created_at`, `updated_at` | ISO 8601 | ✓ | UTC. |
| `errors[].duration_ms` | int | | Time spent on this error. v1 may be 0 if not tracked. |
| `annotation.session_id` | string | ✓ | Groups annotations from one work session. |
| `annotation.annotator_id` | string | ✓ | Matches each error's `annotator_id`. |
| `annotation.started_at`, `saved_at` | ISO 8601 | ✓ | When this image's annotation session began/ended. |
| `annotation.total_duration_ms` | int | | Wall-clock time on this image. |

### Invariants

- `badcase_count` is never stored; consumers compute as `errors.length`.
- An error with `marks: []` is a comment-only error (no spatial location). This is allowed when errors are added via the viewer (which has no bbox drawing UI) or for migrated data with unrecoverable location.
- When `marks` is non-empty, exactly one mark has `role: "primary"`.
- `marks[].geometry.bbox` uses `[x, y, w, h]` in image-pixel space. `x, y` is top-left corner.
- For migrated historical data, a mark with `type: "bbox"` and `bbox: null` is permitted; `comment` notes the data loss.
- All images of the same paper share the **same** `paper.json` (questions + judgments). Image-level files only carry errors, never judgments.
- A paper with one image is valid (single-page paper); `image_count === 1`, `page_index === 0`.

## `_session.json`

```json
{
  "schema_version": "1.0",
  "session_id": "sess_20260629_001",
  "annotator_id": "default",
  "annotator_name": null,
  "source_folder": "未匹配/",
  "started_at": "2026-06-29T09:55:00Z",
  "ended_at": "2026-06-29T11:20:00Z",
  "total_images": 50,
  "status_count": { "annotated": 48, "no_badcase": 2, "skipped": 0, "pending": 0 },
  "error_type_count": { "ocr": 12, "topic": 8, "solution": 5, "judgment": 3 },
  "papers": [
    { "paper_id": "db45c6d8-...", "image_count": 2 }
  ],
  "images": [
    {
      "task_id": "db45c6d8-...",
      "paper_id": "db45c6d8-...",
      "page_index": 0,
      "source_path": "未匹配/1/1.jpg",
      "status": "annotated",
      "error_count": 2,
      "annotation_file": "db45c6d8-.../page_1/annotations/default.json"
    }
  ]
}
```

`status_count`, `error_type_count`, and `papers` are derived; consumers may recompute.

## `_stats.jsonl`

One JSON object per line, flat row per error. Suitable for `pd.read_json(lines=True)`:

```json
{"task_id":"db45c6d8-...","paper_id":"db45c6d8-...","page_index":0,"image":"1.jpg","error_id":"err_01","error_type":"ocr","subtype":"char_wrong","bbox":[120,338,80,42],"comment":"把 7 识别成 1","annotator_id":"default","severity":null,"duration_ms":3200,"saved_at":"2026-06-29T10:02:00Z"}
```

## Backward compatibility

- v2 viewer detects `paper.json` at the task-folder root. If present, reads paper-level path; otherwise falls back to v1 (single-image `<task_id>/annotations/default.json`); otherwise legacy `error_info.txt`.
- Legacy data migrated via `migrate_legacy_zip.py` produces `annotations/default.json` with `bbox: null` (location unrecoverable).
- v1 ZIPs with `annotation.judgments`: viewer ignores this field; judgments are read from `paper.json` only. If a v1 ZIP is opened, judgments card simply doesn't render (no paper.json).
- Tooling that consumes `_stats.jsonl` is unaffected: judgments were never in `_stats.jsonl` (only errors are). The new `paper_id` / `page_index` columns are additive.
