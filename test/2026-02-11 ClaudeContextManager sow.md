# Claude Context Manager Phase 1 実装計画

  

## Context — 背景と目的

  

Claude CLI/Code利用時の対話履歴は自動compactにより失われ、Lost-in-the-middle問題の可視化もできない。本プロジェクトは、対話履歴を完全保存し、コンテキスト管理・可視化を行うツールを開発する。

  

**解決する課題:**

  

1. 自動compactで履歴が失われる（ユーザー発言も消失）

2. 1セッションのtoken量がわからない

3. 過去の対話を検索・再利用できない

4. Zenn記事化など、対話内容の活用が困難

  

**要件定義:** `/Users/harieshokunin/claude-context-manager/spec.md`

  

---

  

## 事前準備：ディレクトリ名変更

  

**実施内容:** `/Users/harieshokunin/winmac_raku/` → `/Users/harieshokunin/claude-context-manager/`

  

```bash

cd /Users/harieshokunin

mv winmac_raku claude-context-manager

cd claude-context-manager

  

# Git設定を更新（リモートURLは変更不要）

git remote -v # 確認

  

# 既存のspec.mdのパス参照を更新

sed -i '' 's|/Users/harieshokunin/winmac_raku|/Users/harieshokunin/claude-context-manager|g' spec.md

```

  

**影響範囲:**

  

- `spec.md`内のパス参照

- `mcp-chatgpt-server`内の設定ファイル（`.env`など）

- `.claude/commands/research.md`内のパス参照

  

---

  

## アーキテクチャ概要

  

**新規プロジェクト:** `/Users/harieshokunin/claude-context-manager/`

  

- 既存: `mcp-chatgpt-server/`（MCPサーバー）

- 新規: `claude-context-manager/`（Hookベースの対話履歴管理）

- `spec.md`（要件定義書）

- `PRINCIPAL.md`（機密情報、`.gitignore`済み）

  

### 技術スタック

  

| レイヤー | 技術 |

|---------|------|

| Hook実装 | Python 3.x（Claude Code Hookの制約） |

| コアロジック | TypeScript + Node.js 20.x |

| Token計測 | tiktoken（p50k_base、推定値） |

| Markdown保存 | `file-manager.ts` のロジック再利用 |

| CLI | Commander.js |

  

### データフロー

  

```

1. ユーザープロンプト送信

↓ UserPromptSubmit hook

→ user-prompt-submit.py（一時ログ保存）

  

2. Claude返答完了

↓ PostToolUse hook

→ post-tool-use.py（一時ログ追記）

  

3. セッション終了

↓ Stop hook

→ stop.py

→ TypeScript finalize-session.ts（Markdown化）

→ ~/.claude/context-history/sessions/YYYY-MM-DD/session-{id}.md 保存

```

  

---

  

## ディレクトリ構造

  

### プロジェクト構造

  

```

/Users/harieshokunin/claude-context-manager/

├── spec.md # 要件定義書

├── PRINCIPAL.md # 機密情報（.gitignore）

├── .gitignore

├── mcp-chatgpt-server/ # 既存プロジェクト

├── claude-context-manager/ # 新規プロジェクト（Phase 1）

│ ├── src/

│ │ ├── hooks/ # Claude Code Hook実装

│ │ │ ├── user-prompt-submit.py

│ │ │ ├── post-tool-use.py

│ │ │ ├── stop.py

│ │ │ └── shared/

│ │ │ ├── logger.py # 一時ログ保存

│ │ │ └── config.py # 設定読み込み

│ │ ├── core/ # TypeScript コアロジック

│ │ │ ├── session-tracker.ts # セッション管理

│ │ │ ├── markdown-writer.ts # Markdown保存

│ │ │ ├── tokenizer.ts # Token計測

│ │ │ └── rotation-manager.ts # ログローテーション

│ │ ├── cli/ # CLI実装

│ │ │ ├── index.ts # エントリーポイント

│ │ │ ├── commands/

│ │ │ │ ├── status.ts # token量表示

│ │ │ │ ├── search.ts # 検索

│ │ │ │ ├── export.ts # Export

│ │ │ │ ├── rotate.ts # ログローテーション

│ │ │ │ └── config.ts # 設定管理

│ │ │ └── finalize-session.ts # セッション終了処理

│ │ ├── types/ # 型定義

│ │ │ ├── session.ts

│ │ │ └── hook-input.ts

│ │ └── utils/ # ユーティリティ

│ │ └── file-utils.ts

│ ├── .claude/

│ │ └── hooks/

│ │ └── hooks.json # Hook設定

│ ├── tests/

│ ├── package.json

│ ├── tsconfig.json

│ └── README.md

```

  

### データ保存先

  

```

~/.claude/context-history/

├── sessions/

│ └── 2026-02-11/

│ ├── session-abc123.md # セッション単位のMarkdown

│ └── session-def456.md

├── archives/

│ └── claude-context-2026-01.tar.gz

├── .tmp/

│ └── session-abc123.json # 一時ログ（Stop hookで削除）

└── .metadata/

├── sessions.json # セッション一覧

└── config.json # 設定ファイル

```

  

---

  

## 実装順序（Week 1–8）

  

### Week 1–2: 基盤実装

  

#### 1. プロジェクトセットアップ

  

```bash

cd /Users/harieshokunin/claude-context-manager

mkdir claude-context-manager

cd claude-context-manager

  

# TypeScriptプロジェクト初期化

npm init -y

npm install --save-dev typescript @types/node tsx jest @types/jest ts-jest

npm install commander chalk ora cli-table3 tiktoken zod

```

  

**tsconfig.json:**

  

```json

{

"compilerOptions": {

"target": "ES2022",

"module": "Node16",

"moduleResolution": "Node16",

"outDir": "./build",

"rootDir": "./src",

"strict": true,

"esModuleInterop": true,

"skipLibCheck": true,

"forceConsistentCasingInFileNames": true

},

"include": ["src/**/*"],

"exclude": ["node_modules", "build", "tests"]

}

```

  

#### 2. Hook基盤実装（Python）

  

**hooks.json:**

  

```json

{

"description": "Claude Context Manager - 対話履歴完全保存",

"hooks": {

"UserPromptSubmit": [

{

"hooks": [

{

"type": "command",

"command": "python3 /Users/harieshokunin/claude-context-manager/claude-context-manager/src/hooks/user-prompt-submit.py",

"timeout": 5

}

]

}

],

"PostToolUse": [

{

"hooks": [

{

"type": "command",

"command": "python3 /Users/harieshokunin/claude-context-manager/claude-context-manager/src/hooks/post-tool-use.py",

"timeout": 5

}

]

}

],

"Stop": [

{

"hooks": [

{

"type": "command",

"command": "python3 /Users/harieshokunin/claude-context-manager/claude-context-manager/src/hooks/stop.py",

"timeout": 10

}

]

}

]

}

}

```

  

**user-prompt-submit.py（骨格）:**

  

```python

#!/usr/bin/env python3

import json

import sys

from pathlib import Path

from datetime import datetime

  

def main():

# stdin から Hook入力を読み込み

input_data = json.load(sys.stdin)

  

# セッションIDを取得（または生成）

session_id = input_data.get('sessionId', 'unknown')

user_prompt = input_data.get('userPrompt', '')

  

# 一時ログファイルに保存

log_dir = Path.home() / '.claude' / 'context-history' / '.tmp'

log_dir.mkdir(parents=True, exist_ok=True)

  

log_file = log_dir / f'session-{session_id}.json'

  

# ログエントリ作成

log_entry = {

'timestamp': datetime.now().isoformat(),

'type': 'user',

'content': user_prompt,

'tokens_estimate': len(user_prompt) // 4 # 簡易推定

}

  

# 既存ログに追記

logs = []

if log_file.exists():

with open(log_file, 'r') as f:

logs = json.load(f)

logs.append(log_entry)

  

with open(log_file, 'w') as f:

json.dump(logs, f, indent=2)

  

# Hook出力

output = {

"hookSpecificOutput": {

"status": "logged",

"session_id": session_id

}

}

print(json.dumps(output))

  

if __name__ == '__main__':

main()

```

  

**post-tool-use.py（骨格）:**

  

```python

#!/usr/bin/env python3

import json

import sys

from pathlib import Path

from datetime import datetime

  

def main():

input_data = json.load(sys.stdin)

  

session_id = input_data.get('sessionId', 'unknown')

tool_name = input_data.get('toolName', '')

tool_result = input_data.get('toolResult', '')

  

log_dir = Path.home() / '.claude' / 'context-history' / '.tmp'

log_file = log_dir / f'session-{session_id}.json'

  

log_entry = {

'timestamp': datetime.now().isoformat(),

'type': 'assistant',

'tool_name': tool_name,

'content': tool_result,

'tokens_estimate': len(tool_result) // 4

}

  

logs = []

if log_file.exists():

with open(log_file, 'r') as f:

logs = json.load(f)

logs.append(log_entry)

  

with open(log_file, 'w') as f:

json.dump(logs, f, indent=2)

  

output = {"hookSpecificOutput": {"status": "logged"}}

print(json.dumps(output))

  

if __name__ == '__main__':

main()

```

  

**stop.py（骨格）:**

  

```python

#!/usr/bin/env python3

import json

import sys

import subprocess

from pathlib import Path

  

def main():

input_data = json.load(sys.stdin)

session_id = input_data.get('sessionId', 'unknown')

  

# TypeScriptスクリプトを呼び出してMarkdown化

ts_script = Path(__file__).parent.parent / 'cli' / 'finalize-session.ts'

subprocess.run([

'npx', 'tsx', str(ts_script), session_id

])

  

output = {"hookSpecificOutput": {"status": "finalized"}}

print(json.dumps(output))

  

if __name__ == '__main__':

main()

```

  

#### 3. セッション管理（TypeScript）

  

**finalize-session.ts:**

  

```typescript

#!/usr/bin/env node

import * as fs from 'fs/promises';

import * as path from 'path';

import { MarkdownWriter } from '../core/markdown-writer';

  

async function main() {

const sessionId = process.argv[2];

if (!sessionId) {

console.error('Session ID required');

process.exit(1);

}

  

const tmpDir = path.join(process.env.HOME!, '.claude', 'context-history', '.tmp');

const logFile = path.join(tmpDir, `session-${sessionId}.json`);

  

// 一時ログを読み込み

const logsJson = await fs.readFile(logFile, 'utf-8');

const logs = JSON.parse(logsJson);

  

// Markdown化

const writer = new MarkdownWriter();

const markdown = await writer.generate(sessionId, logs);

  

// 保存

const today = new Date().toISOString().split('T')[0]; // YYYY-MM-DD

const outputDir = path.join(process.env.HOME!, '.claude', 'context-history', 'sessions', today);

await fs.mkdir(outputDir, { recursive: true });

  

const outputFile = path.join(outputDir, `session-${sessionId}.md`);

await fs.writeFile(outputFile, markdown, 'utf-8');

  

// 一時ログ削除

await fs.unlink(logFile);

  

console.log(`Session finalized: ${outputFile}`);

}

  

main().catch(console.error);

```

  

**markdown-writer.ts（`file-manager.ts`のロジック再利用）:**

  

```typescript

export class MarkdownWriter {

async generate(sessionId: string, logs: any[]): Promise<string> {

const firstLog = logs[0];

const lastLog = logs[logs.length - 1];

  

const startTime = new Date(firstLog.timestamp);

const endTime = new Date(lastLog.timestamp);

  

// Token集計

const totalTokens = logs.reduce((sum, log) => sum + (log.tokens_estimate || 0), 0);

const userTokens = logs

.filter(l => l.type === 'user')

.reduce((sum, log) => sum + (log.tokens_estimate || 0), 0);

const assistantTokens = logs

.filter(l => l.type === 'assistant')

.reduce((sum, log) => sum + (log.tokens_estimate || 0), 0);

  

// Frontmatter

let markdown = `---

date: ${startTime.toISOString()}

session_id: ${sessionId}

total_tokens: ${totalTokens}

user_tokens: ${userTokens}

assistant_tokens: ${assistantTokens}

compact_detected: false

tags: [claude, conversation]

---

  

# Claude対話履歴 - ${startTime.toISOString().split('T')[0]}

  

`;

  

// ログエントリ

for (const log of logs) {

const time = new Date(log.timestamp).toLocaleTimeString('ja-JP');

const role = log.type === 'user'

? 'ユーザー'

: `Claude${log.tool_name ? ` (${log.tool_name})` : ''}`;

  

markdown += `\n## ${time} - ${role}\n\n`;

markdown += `${log.content}\n\n`;

markdown += `**Tokens**: ${log.tokens_estimate} (推定)\n`;

}

  

// 統計

const durationMinutes = Math.round(

(endTime.getTime() - startTime.getTime()) / 60000

);

markdown += `\n---\n\n**セッション統計**:\n`;

markdown += `- 総Token数: ${totalTokens} (推定)\n`;

markdown += `- ユーザーToken: ${userTokens}\n`;

markdown += `- ClaudeToken: ${assistantTokens}\n`;

markdown += `- Compact: なし\n`;

markdown += `- セッション時間: ${durationMinutes}分\n`;

  

return markdown;

}

}

```

  

---

  

### Week 3–4: Token可視化

  

**tokenizer.ts:**

  

```typescript

import { get_encoding } from 'tiktoken';

  

export function estimateTokens(text: string): number {

const encoding = get_encoding('p50k_base'); // Claude 3の近似

const tokens = encoding.encode(text);

encoding.free();

return tokens.length;

}

```

  

**CLI status コマンド（`src/cli/commands/status.ts`）:**

  

```typescript

import * as fs from 'fs/promises';

import * as path from 'path';

import Table from 'cli-table3';

  

export async function statusCommand() {

const tmpDir = path.join(process.env.HOME!, '.claude', 'context-history', '.tmp');

const files = await fs.readdir(tmpDir);

  

if (files.length === 0) {

console.log('No active sessions');

return;

}

  

for (const file of files) {

const logFile = path.join(tmpDir, file);

const logsJson = await fs.readFile(logFile, 'utf-8');

const logs = JSON.parse(logsJson);

  

const totalTokens = logs.reduce(

(sum: number, log: any) => sum + (log.tokens_estimate || 0), 0

);

const userTokens = logs

.filter((l: any) => l.type === 'user')

.reduce((sum: number, log: any) => sum + (log.tokens_estimate || 0), 0);

const assistantTokens = logs

.filter((l: any) => l.type === 'assistant')

.reduce((sum: number, log: any) => sum + (log.tokens_estimate || 0), 0);

  

const table = new Table({

head: ['Session', 'Total Tokens', 'User', 'Assistant'],

colWidths: [20, 15, 10, 10]

});

  

table.push([

file.replace('session-', '').replace('.json', ''),

totalTokens,

userTokens,

assistantTokens

]);

  

console.log(table.toString());

  

// Lost-in-the-middle警告

if (totalTokens > 50000) {

console.warn(

'⚠️ Warning: Context size exceeded 50,000 tokens. Lost-in-the-middle may occur.'

);

}

}

}

```

  

---

  

### Week 5–6: Export機能

  

**search.ts（grepベース）:**

  

```typescript

import * as path from 'path';

import { execSync } from 'child_process';

  

export async function searchCommand(keyword: string) {

const sessionsDir = path.join(

process.env.HOME!, '.claude', 'context-history', 'sessions'

);

  

try {

const result = execSync(`grep -r "${keyword}" ${sessionsDir}`, {

encoding: 'utf-8'

});

console.log(result);

} catch (err: any) {

if (err.status === 1) {

console.log('No matches found');

} else {

throw err;

}

}

}

```

  

**export.ts（Obsidian/Zenn形式）:**

  

```typescript

export async function exportCommand(format: string, sessionId?: string) {

// Obsidian形式: そのまま（既にObsidian互換）

if (format === 'obsidian') {

console.log('Sessions are already in Obsidian-compatible Markdown format');

console.log(`Location: ~/.claude/context-history/sessions/`);

return;

}

  

// Zenn形式: frontmatterを調整

if (format === 'zenn') {

// TODO: Zenn形式に変換

}

}

```

  

---

  

### Week 7–8: ログローテーション、テスト、ドキュメント

  

**rotation-manager.ts:**

  

```typescript

import * as fs from 'fs/promises';

import * as path from 'path';

import { execSync } from 'child_process';

  

export class RotationManager {

async rotate(days: number = 30) {

const sessionsDir = path.join(

process.env.HOME!, '.claude', 'context-history', 'sessions'

);

const archivesDir = path.join(

process.env.HOME!, '.claude', 'context-history', 'archives'

);

  

const cutoffDate = new Date();

cutoffDate.setDate(cutoffDate.getDate() - days);

  

const dateDirs = await fs.readdir(sessionsDir);

  

for (const dateDir of dateDirs) {

const date = new Date(dateDir);

if (date < cutoffDate) {

const archiveName = `claude-context-${dateDir}.tar.gz`;

const archivePath = path.join(archivesDir, archiveName);

  

await fs.mkdir(archivesDir, { recursive: true });

execSync(`tar -czf ${archivePath} -C ${sessionsDir} ${dateDir}`);

  

// 削除

await fs.rm(path.join(sessionsDir, dateDir), { recursive: true });

console.log(`Archived and removed: ${dateDir}`);

}

}

}

}

```

  

**CLI rotate コマンド:**

  

```typescript

import { RotationManager } from '../core/rotation-manager';

  

export async function rotateCommand(days?: number) {

const manager = new RotationManager();

await manager.rotate(days || 30);

console.log('Log rotation completed');

}

```

  

---

  

## 重要ファイル一覧

  

### 既存ファイル（参考実装）

  

| ファイル | 用途 |

|---------|------|

| `mcp-chatgpt-server/src/utils/file-manager.ts` | Markdown保存ロジック、ディレクトリ作成、ファイル名生成 |

| `~/.claude/plugins/.../hookify/hooks/userpromptsubmit.py` | Hook実装パターン、stdin/stdout処理 |

| `~/.claude/plugins/.../security-guidance/hooks/security_reminder_hook.py` | Hook実装ベストプラクティス |

  

### 新規作成ファイル（Phase 1）

  

| ファイル | 用途 |

|---------|------|

| `src/hooks/user-prompt-submit.py` | ユーザープロンプトのログ記録 |

| `src/hooks/post-tool-use.py` | ツール使用結果のログ記録 |

| `src/hooks/stop.py` | セッション終了・Markdown化トリガー |

| `src/core/markdown-writer.ts` | Markdown生成ロジック |

| `src/cli/finalize-session.ts` | セッション終了処理 |

| `src/cli/commands/status.ts` | Token量表示CLI |

  

---

  

## 検証方法

  

### 1. セットアップ確認

  

```bash

cd /Users/harieshokunin/claude-context-manager/claude-context-manager

npm install

npm run build # tsconfig.jsonに合わせて tsc を実行

  

# Hook設定

cp .claude/hooks/hooks.json ~/.claude/hooks/context-manager-hooks.json

# または既存の hooks.json に統合

```

  

### 2. Hook動作テスト（mockデータ）

  

```bash

# UserPromptSubmit hook テスト

echo '{"sessionId": "test-001", "userPrompt": "Hello Claude"}' \

| python3 src/hooks/user-prompt-submit.py

  

# 一時ログ確認

cat ~/.claude/context-history/.tmp/session-test-001.json

```

  

### 3. End-to-Endテスト

  

```bash

# Claude Code起動

claude

  

# 簡単な対話

> こんにちは

> Rustの非同期処理について教えてください

  

# セッション終了

/exit

  

# Markdownファイル確認

ls ~/.claude/context-history/sessions/$(date +%Y-%m-%d)/

cat ~/.claude/context-history/sessions/$(date +%Y-%m-%d)/session-*.md

```

  

### 4. CLI動作確認

  

```bash

npx tsx src/cli/index.ts status # Token量表示

npx tsx src/cli/index.ts search "非同期" # 検索

npx tsx src/cli/index.ts export --format obsidian # Export

npx tsx src/cli/index.ts rotate --days 30 # ログローテーション

```

  

### 5. Obsidian連携確認

  

```bash

# シンボリックリンク作成（任意）

ln -s ~/.claude/context-history/sessions \

~/Library/Mobile\ Documents/iCloud~md~obsidian/Documents/my-vault/claude-sessions

  

# Obsidianで開いて確認

```

  

---

  

## Phase 1 MVP のスコープ

  

### 実装する機能（Week 1–6）

  

- ✅ 対話履歴の完全保存（ユーザー + Claude返答）

- ✅ Markdown形式で保存（Obsidian互換）

- ✅ Token可視化（推定値、tiktoken p50k_base）

- ✅ 基本的な検索機能（grepベース）

- ✅ Obsidian連携

  

### Phase 2に先送り

  

- ❌ Web Dashboard

- ❌ SQLiteによる高速検索

- ❌ Anthropic APIでの正確なtoken計測（`client.messages.countTokens()`）

- ❌ Compact検出・差分表示（APIが非公開）

- ❌ 画像・スクリーンショットの保存（Hookでの画像取得方法を調査中）

- ❌ 自動ログローテーション（cron設定は手動）

  

---

  

## リスクと対策

  

| リスク | 影響 | 対策 |

|-------|------|------|

| Hook APIの変更 | 実装が動作しなくなる | Claude Code公式ドキュメントを定期的に確認 |

| Token計測の精度（70–80%） | Lost-in-the-middle警告が不正確 | Phase 1では「推定値」として明示、Phase 2でAPI呼び出しによる補正 |

| 一時ログファイルの肥大化 | ディスク容量圧迫 | Stop hookで確実に削除、定期的なクリーンアップスクリプト |

| Compact検出ができない | compact前後の差分が取れない | Phase 2で調査、ヒューリスティック検出を検討 |

  

---

  

## 次のアクション

  

1. **ディレクトリ名変更:** `winmac_raku` → `claude-context-manager`

2. **プロジェクトセットアップ:** TypeScript環境構築

3. **Hook実装:** `user-prompt-submit.py`, `post-tool-use.py`, `stop.py`

4. **動作確認:** mockデータでHook動作テスト

5. **End-to-End確認:** Claude Codeで実際の対話をテスト