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

Use Conventional Commits based commit messages.

Format:

```text
<type>(<scope>): <summary>
```

Examples:

```text
docs(design): 基本ルーティング設計を追加
feat(router): プロバイダー選択インターフェースを追加
fix(config): 不正なプロバイダー名を拒否
test(router): フォールバックルーティングのケースを追加
chore: 開発用依存関係を更新
```

Recommended `type` values:

- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation only
- `test`: Test additions or updates
- `refactor`: Code changes without behavior changes
- `chore`: Maintenance tasks
- `build`: Build system or dependency changes
- `ci`: CI configuration changes
- `perf`: Performance improvement

`scope` is optional. Prefer concise scopes that match the affected area, such as `docs`, `design`, `router`, `provider`, `config`, `api`, `cli`, or `codex`.

`<summary>` は必ず日本語で記載してください。

Use `!` for breaking changes:

```text
feat(config)!: プロバイダー設定スキーマを置き換え
```
