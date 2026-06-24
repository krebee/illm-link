# AGENTS.md

## Project Purpose

iLLM Link は、Codex App と Codex CLI 向けのローカル LLM ルーティング層です。

localhost 上に OpenAI Responses API 互換のエンドポイントを公開し、Codex からのリクエストを、要求されたモデル名に基づいてローカルモデルやサードパーティ製 LLM プロバイダーへ転送します。

このプロジェクトの主な目的は、Codex でモデルを切り替えるたびに `~/.codex/config.toml` を頻繁に編集しなくても済むようにすることです。Codex 側からは iLLM Link を単一のカスタムプロバイダーとして扱い、iLLM Link 側で OpenAI / Codex 互換モデル、Z.ai GLM、Ollama、LM Studio などへ振り分けます。

現時点では設計フェーズです。初期実装では、`POST /v1/responses`、テキスト入出力、ストリーミング / SSE、モデルプレフィックスによるプロバイダー振り分け、ヘッダー分離、基本的なエラー変換、localhost のみでバインドするサーバーを主な対象とします。


## Directory Notes

- `docs/`: Directory for design documents.

### docs directory structure

Only the structure under docs/ is listed below.

```text
docs/
└── design/
    └── illm-link_design_v4.md
```

## Status

Design phase.
