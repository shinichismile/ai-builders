# サブエージェント定義ファイルの仕様（2026-07時点・公式docs確認済み）

出典: https://code.claude.com/docs/en/sub-agents — 仕様は更新されるので、珍しいフィールドを使うときや動かないときはWebFetchで最新を再確認すること。

## 基本構造

YAMLフロントマター＋Markdown本文。**本文がそのままシステムプロンプトになる**（Claude Code本体のシステムプロンプトは届かない。届くのは本文＋作業ディレクトリ等の基本環境情報のみ）。

```markdown
---
name: code-reviewer
description: コードの品質とベストプラクティスをレビューする。コード変更後は積極的に使うこと。
tools: Read, Glob, Grep
model: sonnet
---

あなたはコードレビュアーです。呼ばれたら対象コードを分析し、
品質・セキュリティ・ベストプラクティスの観点で具体的で実行可能な指摘を返してください。
報告は要点のみ、重要度順に。
```

## 保存場所と優先順位（同名なら上が勝つ）

| 場所 | スコープ |
|---|---|
| 管理者設定（managed settings） | 組織全体 |
| `--agents` CLIフラグ（JSON） | そのセッションのみ |
| `.claude/agents/` | 現在のプロジェクト |
| `~/.claude/agents/` | 全プロジェクト |
| プラグインの `agents/` | プラグイン有効な場所 |

- ディレクトリは再帰的にスキャンされる（`agents/review/` のようなサブフォルダ整理OK。識別子は`name`フィールドのみで決まる）
- ファイルを直接置いた/編集した場合も**数秒で自動検知される（原則、再起動不要）**。まれに認識されないとき（そのスコープで初めてagentsフォルダを作った直後など）だけ、新しいセッションで確認する
- プラグイン配布のサブエージェントでは `hooks` / `mcpServers` / `permissionMode` は無視される（セキュリティ上の理由）

## フロントマター全フィールド

必須は `name` と `description` のみ。

| フィールド | 説明 |
|---|---|
| `name` | 一意の識別子。小文字とハイフンのみ |
| `description` | 親Claudeが委任を判断する唯一の材料。場面まで書く |
| `tools` | 許可リスト。省略時は親の全ツールを継承。`Read, Glob, Grep` のようにカンマ区切り |
| `disallowedTools` | 拒否リスト。継承 or tools指定から差し引く。両方指定時はdisallowedTools適用→残りにtoolsを解決 |
| `model` | `sonnet` / `opus` / `haiku` / `fable` / フルモデルID / `inherit`（省略時はinherit＝親と同じ） |
| `permissionMode` | `default` / `manual`（defaultの別名・v2.1.200以降） / `acceptEdits` / `auto` / `dontAsk` / `bypassPermissions` / `plan`。**bypassPermissionsは原則使わない**（確認なしで何でも実行できてしまう） |
| `maxTurns` | 最大ターン数。暴走防止の上限として有効 |
| `skills` | 起動時にコンテキストへ**全文**プリロードするスキル。指定しなくてもSkillツール経由の呼び出しは可能 |
| `mcpServers` | このサブエージェント専用のMCPサーバー。設定済みサーバー名の参照 or インライン定義 |
| `hooks` | このサブエージェント専用のライフサイクルフック |
| `memory` | `user` / `project` / `local`。セッションをまたいで学習を蓄積する永続メモリを持たせる |
| `background` | `true`で常にバックグラウンド実行 |
| `effort` | `low` / `medium` / `high` / `xhigh` / `max`（モデルにより異なる）。セッションの努力レベルを上書き |
| `isolation` | `worktree`で一時的なgit worktree（リポジトリの隔離コピー）内で実行。変更がなければ自動掃除 |
| `color` | UI表示色: red / blue / green / yellow / purple / orange / pink / cyan |
| `initialPrompt` | `claude --agent`でメインセッションとして起動したときに自動送信される最初のプロンプト |

## ツール指定の詳細

- MCPサーバー単位のパターンが使える: `mcp__<server>` または `mcp__<server>__*` でそのサーバーの全ツール。`disallowedTools`では `mcp__*` で全MCPツールを除外
- サブエージェントでは使えないツール（listしても無効）: `AskUserQuestion`, `EnterPlanMode`, `ExitPlanMode`（permissionMode: plan時を除く）, `ScheduleWakeup` 等、親セッションのUIに依存するもの
- `tools`に`Agent`を含めると、そのサブエージェントは入れ子のサブエージェントを起動できる

## モデル解決の優先順位

1. 環境変数 `CLAUDE_CODE_SUBAGENT_MODEL`（全サブエージェントを強制。コスト上限管理に有効）
2. 呼び出し時の`model`パラメータ
3. フロントマターの`model`
4. 親会話のモデル

## モデル選択の目安（コスト最適化）

| 仕事の性質 | 推奨 | 理由 |
|---|---|---|
| 検索・分類・定型チェック | haiku | 速く安く、十分な精度 |
| 標準的な分析・レビュー・執筆 | sonnet | 能力とコストのバランス |
| 高度な設計判断・難しい推論 | opus / fable | 精度が事故コストを上回る場面のみ |
| 迷ったら | inherit（省略） | 親と同じ挙動で試してから下げる |
