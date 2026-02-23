# scratch-archive-spec v1

## 概要

この仕様は、複数人で共同して Scratch のプロジェクトをアーカイブ（収集・保存・共有）するためのディレクトリ構成とファイル形式を定める。

## アップロード

```bash
hf upload-dir pnsk-lab/scratch-archive .
```

### Hugging Face 上での確認

* 既存のアセット一覧は、Hugging Face API の tree を使って確認できる。

  * `https:### Hugging Face 上での確認

* 既存のアセット一覧は、Hugging Face API の tree を使って確認できる。

  * `https://huggingface.co/api/datasets/pnsk-lab/scratch-archive/tree/main/assets?expand=false`

* 特定スナップショットの `info.json` は raw から直接確認できる。

  * `https://huggingface.co/datasets/pnsk-lab/scratch-archive/raw/main/projects/{project_id}/{last_modified_utc}/info.json`

### 効率化（既存のものはスキップ）

共同で追加していく前提のため、アップロード前に既存データを参照して「すでにあるものは作らない・取らない」ことを推奨する。

* **アセットの重複回避**

  * `assets/` の tree から既存ファイル名（`{asset_id}.{ext}`）を取得し、存在するものは **ダウンロード/保存をスキップ**する。
  * 例: `.../tree/main/assets?expand=false`

* **スナップショットの重複回避**

  * `projects/{project_id}/{last_modified_utc}/info.json` が raw で取得できるなら、そのスナップショットは既存とみなし、 **同パスの生成/アップロードをスキップ**する。
  * 例: `.../raw/main/projects/{project_id}/{last_modified_utc}/info.json`

* **404 を記録する場合**

  * `info.json` が 404（プロジェクトが存在しない）なら、`metadata.json` に `notFound: true` を付けてスナップショットとして残してよい。
  * その場合、`project.json` / `thumbnail.{ext}` / `assets/` は作らなくてよい（取得できないため）。

## ディレクトリ構成

```
projects/
  {project_id}/{last_modified_utc}/
    project.json
    metadata.json
    info.json
    thumbnail.{ext}
assets/
  {asset_id}.{ext}
```

* `project_id`: Scratch の数値 ID。
* `last_modified_utc`: `info.json` の `history.modified` を使用（ISO 8601、小数秒を含める）。例: `2025-11-11T11-42-30.000Z`

  * ファイルシステム互換のため `:` は `-` に置換する。
* `asset_id`: Scratch の assetId（例: md5hash）。
* `{ext}`: Scratch が参照する拡張子（例: `png`, `wav`, `svg`, `json` など）。

### project.json

Scratch のプロジェクトデータ本体（Scratch 3 の project.json そのもの）。

* この `project.json` に含まれる **costumes / sounds の全アセット**をダウンロードして `assets/` に保存する。

### metadata.json

`projects/{project_id}/{last_modified_utc}/` スナップショットを取得した時刻だけを持つ。

例:

```json
{
  "scrapedAt": "2026-02-23T13:00:00Z"
}
```

* `scrapedAt`: ISO 8601（UTC 推奨）。
* `notFound`: （オプショナル）取得時にプロジェクトが 404 だった場合は `true` を入れる。

#### metadata.json の JSON Schema（draft 2020-12）

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.invalid/scratch-archive/metadata.schema.json",
  "title": "Scratch Archive Snapshot Metadata",
  "type": "object",
  "additionalProperties": false,
  "required": ["scrapedAt"],
  "properties": {
    "scrapedAt": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 timestamp when this snapshot was scraped (UTC recommended)."
    },
    "notFound": {
      "type": "boolean",
      "description": "Optional. True if the project fetch returned 404 (not found)."
    }
  }
}
```

### info.json

Scratch の `https://api.scratch.mit.edu/projects/{id}` などから取得できる「プロジェクト情報」をそのまま保存する（タイトル/作者/統計/履歴/サムネURLなど）。この API のデータを加工なしで保存。ただし、意味的な変更を加えないのであれば、JSON のフォーマットなどは許可する。

### thumbnail.{ext}

`info.json` の `image` フィールド（最大画質のサムネイル URL）を **HTTP fetch した生データ**を保存する。

* `{ext}` は `image` URL の拡張子に合わせる（例: `png`, `jpg`, `webp` など）。
* 取得に失敗した場合は `thumbnail.{ext}` を作らず、代わりに `info.json.image` を参照する。

#### info.json の JSON Schema（draft 2020-12）

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.invalid/scratch-archive/info.schema.json",
  "title": "Scratch Project Info",
  "type": "object",
  "additionalProperties": true,
  "required": ["id", "title", "author", "history"],
  "properties": {
    "id": {"type": "integer", "minimum": 0},
    "title": {"type": "string"},
    "description": {"type": "string"},
    "instructions": {"type": "string"},

    "visibility": {"type": "string"},
    "public": {"type": "boolean"},
    "comments_allowed": {"type": "boolean"},
    "is_published": {"type": "boolean"},

    "author": {"$ref": "#/$defs/author"},

    "image": {"type": "string", "format": "uri"},
    "images": {"$ref": "#/$defs/imageMap"},

    "history": {"$ref": "#/$defs/projectHistory"},
    "stats": {"$ref": "#/$defs/stats"},

    "remix": {
      "type": "object",
      "additionalProperties": true,
      "properties": {
        "parent": {"type": ["integer", "null"], "minimum": 0},
        "root": {"type": ["integer", "null"], "minimum": 0}
      }
    },

    "project_token": {"type": "string"}
  },

  "$defs": {
    "dateTime": {"type": "string", "format": "date-time"},

    "imageMap": {
      "type": "object",
      "additionalProperties": {"type": "string", "format": "uri"}
    },

    "projectHistory": {
      "type": "object",
      "additionalProperties": true,
      "required": ["created", "modified"],
      "properties": {
        "created": {"$ref": "#/$defs/dateTime"},
        "modified": {"$ref": "#/$defs/dateTime"},
        "shared": {"$ref": "#/$defs/dateTime"}
      }
    },

    "stats": {
      "type": "object",
      "additionalProperties": true,
      "properties": {
        "views": {"type": "integer", "minimum": 0},
        "loves": {"type": "integer", "minimum": 0},
        "favorites": {"type": "integer", "minimum": 0},
        "remixes": {"type": "integer", "minimum": 0}
      }
    },

    "author": {
      "type": "object",
      "additionalProperties": true,
      "required": ["id", "username"],
      "properties": {
        "id": {"type": "integer", "minimum": 0},
        "username": {"type": "string"},
        "scratchteam": {"type": "boolean"},

        "history": {
          "type": "object",
          "additionalProperties": true,
          "properties": {
            "joined": {"$ref": "#/$defs/dateTime"}
          }
        },

        "profile": {
          "type": "object",
          "additionalProperties": true,
          "properties": {
            "id": {"type": ["integer", "null"], "minimum": 0},
            "images": {
              "type": "object",
              "additionalProperties": {"type": "string", "format": "uri"}
            },
            "membership_avatar_badge": {"type": "integer", "minimum": 0},
            "membership_label": {"type": "integer", "minimum": 0}
          }
        }
      }
    }
  }
}
```
