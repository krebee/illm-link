# AGENTS.md

## Project Purpose

iLLM Link は、Codex App と Codex CLI 向けのローカル LLM ルーティング層です。Codex からは単一のカスタムプロバイダーとして扱い、iLLM Link 側でローカルモデルやサードパーティ製 LLM プロバイダーへ振り分けます。

プロジェクトの詳細、目的、スコープはルート直下の [README.md](./README.md) を参照してください。


## Directory Notes

- `docs/`: Directory for design documents.
- `docs/design/`: Store all design documents here. Refer to and update these documents as appropriate when making design or implementation decisions.
- `docs/design/decisions/`: Store important design decision records in an ADR-like format. Add or update entries when a design choice has lasting impact or important trade-offs.

### docs directory structure

Only the structure under docs/ is listed below.

```text
docs/
└── design/
    ├── basic-design.md
    ├── technical-design.md
    └── decisions/
        └── template.md
```

## Status

Design phase.

## Commit Messages

コミットメッセージは Conventional Commits に準拠してください。

Format:

```text
<type>(<scope>): <summary>
```

`<summary>` は必ず日本語で簡潔に記載してください。詳細は `conventional-commit-messages` skill を参照してください。
