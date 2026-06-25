---
name: conventional-commit-messages
description: Draft and validate project commit messages that follow Conventional Commits. Use when Codex creates a commit, suggests a commit message, reviews commit message format, or decides commit type, scope, Japanese summary, or breaking-change notation for this repository.
---

# Conventional Commit Messages

## Overview

Use Conventional Commits for all project commit messages. Keep the subject concise, make the summary Japanese, and choose the smallest accurate type and scope for the actual change.

## Format

```text
<type>(<scope>): <summary>
```

Omit `(scope)` only when a concise, meaningful scope is not available.

For breaking changes, add `!` before the colon:

```text
feat(config)!: プロバイダー設定スキーマを置き換え
```

## Types

- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation only
- `test`: Test additions or updates
- `refactor`: Code changes without behavior changes
- `chore`: Maintenance tasks
- `build`: Build system or dependency changes
- `ci`: CI configuration changes
- `perf`: Performance improvement

Choose the type by the user-visible or maintainer-visible intent of the change. Prefer `docs` for design document updates, `chore` for repository housekeeping, and `feat` only when behavior or capability is added.

## Scopes

Prefer short scopes that match the affected area:

- `docs`
- `design`
- `router`
- `provider`
- `config`
- `api`
- `cli`
- `codex`

Use another concise scope when it better matches the changed files. Do not force a scope when the change is broad or only repository-level maintenance.

## Summary

Write `<summary>` in Japanese. Keep it short, imperative or noun-ending, and specific to the change.

Avoid vague summaries such as `修正`, `更新`, or `作業内容を反映`.

## Examples

```text
docs(design): 基本ルーティング設計を追加
feat(router): プロバイダー選択インターフェースを追加
fix(config): 不正なプロバイダー名を拒否
test(router): フォールバックルーティングのケースを追加
chore: 開発用依存関係を更新
```
