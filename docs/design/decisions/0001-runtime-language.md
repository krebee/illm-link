# 0001: 実装言語として Rust を採用する

## Status

Accepted

## Date

2026-06-25

## Context

iLLM Link は、Codex App / Codex CLI からのリクエストを受け付け、ローカルモデルまたは外部 LLM provider へ中継する localhost 専用の常駐ルーターです。

主な実行環境は Windows と Ubuntu Linux を想定します。

この用途では、次の性質を重視します。

- 小さいメモリ使用量
- 高い実行性能
- localhost 常駐プロセスとしての安定性
- HTTP request / response と streaming の効率的な中継
- Windows と Linux での単一バイナリ配布
- ルーティング、provider 分離、error handling などの境界を明確に保てること

候補としては、Go と Rust を中心に比較します。

## Decision

iLLM Link の実装言語として Rust を採用します。

Rust は GC を持たず、常駐プロセスのメモリ使用量を抑えやすいです。また、型システムにより、provider adapter、routing、streaming、error handling などの境界を明示しやすく、設計上の不変条件をコード上で表現しやすいです。

本プロジェクトでは実装作業の大部分を LLM 支援で進める前提があるため、Rust の相対的な弱点である初期実装難度や記述量の多さは、決定的な欠点にはなりにくいと判断します。

## Consequences

期待される利点は次の通りです。

- 小メモリで動作する常駐プロセスを目指しやすいです。
- 高性能な HTTP 中継処理を実装しやすいです。
- Windows と Ubuntu Linux 向けに単一バイナリとして配布しやすいです。
- 型によって provider ごとの差分、unsupported feature、error kind などを明示的に扱いやすいです。
- LLM 支援による実装でも、コンパイル時検査により構造的な誤りを検出しやすいです。

想定されるトレードオフは次の通りです。

- Go と比べると、実装の記述量は増えやすいです。
- async、lifetime、trait 設計などで設計判断が重くなる場面があります。
- 開発者が手作業で保守する場合、Go より参入コストが高くなる可能性があります。

ただし、本プロジェクトでは小メモリ・高性能・型安全性を優先し、実装難度は LLM 支援で補えるものとして扱います。

## Alternatives Considered

- Go: 標準ライブラリが成熟しており、HTTP server / client、CLI、単一バイナリ配布に強いです。実装速度と保守容易性にも優れます。一方で、GC を持つため、最小メモリ使用量やレイテンシ特性を最優先する場合は Rust のほうが適しています。本プロジェクトでは実装速度よりも小メモリ・高性能・型安全性を重視するため、採用しません。
- Rust: 小メモリ、高性能、型安全性、単一バイナリ配布の条件に最も合います。実装難度は Go より高いですが、LLM 支援を前提とするため許容可能と判断し、採用します。
