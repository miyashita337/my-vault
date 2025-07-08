# Claude Screenshot Setup 2025-07-09

## 概要

macOSでスクリーンショットを自動的にファイル保存し、Obsidian vaultで参照できるシステムのセットアップ手順とスクリプト。

## 実装した機能

### 1. スクリーンショット保存場所の変更
- デフォルトの保存場所を `~/Pictures/Screenshots` に変更
- macOSの設定を永続化

### 2. 自動監視・参照システム
- 新しいスクリーンショットを自動検出
- `Screenshots.md` に自動的に参照を追加
- ファイル情報（パス、ファイル名、タイムスタンプ）を記録

### 3. 便利なコマンドエイリアス
- `screenshot-process`: 既存スクリーンショットを処理
- `screenshot-monitor`: リアルタイム監視開始

## セットアップ手順

### 1. 基本設定
```bash
# スクリーンショット保存ディレクトリ作成
mkdir -p ~/Pictures/Screenshots

# macOSのスクリーンショット保存場所を変更
defaults write com.apple.screencapture location ~/Pictures/Screenshots
killall SystemUIServer
```

### 2. 監視ツールのインストール
```bash
# fswatch（ファイル監視ツール）をインストール
brew install fswatch
```

### 3. スクリプトのセットアップ
```bash
# セットアップスクリプトを実行
~/Pictures/Screenshots/setup_screenshot_automation.sh
```

## 作成されたファイル

### screenshot_manager.sh
メインの監視・処理スクリプト
- 新しいスクリーンショットを検出
- `Screenshots.md` に参照を自動追加
- 手動処理モードと監視モードの両方をサポート

### setup_screenshot_automation.sh
初期セットアップ用スクリプト
- fswatch のインストール
- シェルエイリアスの設定
- LaunchAgent の作成（自動起動用）

## 使用方法

### スクリーンショット撮影
- **Command+Shift+4**: 部分キャプチャ（ファイルに保存）
- **Command+Shift+Control+4**: 部分キャプチャ（クリップボードのみ）

### 管理コマンド
```bash
# 既存のスクリーンショットを処理
screenshot-process

# リアルタイム監視開始
screenshot-monitor

# 直接実行
~/Pictures/Screenshots/screenshot_manager.sh [monitor|process|help]
```

### 自動起動設定
```bash
# 自動起動を有効にする
launchctl load ~/Library/LaunchAgents/com.user.screenshot-monitor.plist

# 自動起動を無効にする
launchctl unload ~/Library/LaunchAgents/com.user.screenshot-monitor.plist
```

## ファイル構造

```
~/Pictures/Screenshots/
├── screenshot_manager.sh          # メイン監視スクリプト
├── setup_screenshot_automation.sh # セットアップスクリプト
└── *.png                         # スクリーンショットファイル

~/my-vault/test/
└── Screenshots.md                # 自動生成される参照ファイル
```

## Screenshots.md の形式

```markdown
# Screenshots

Automatically generated screenshot references.

## Screenshot - 2025-07-09 00:54:04

![Screenshot](/Users/username/Pictures/Screenshots/スクリーンショット 2025-07-09 0.53.29.png)

**File:** `スクリーンショット 2025-07-09 0.53.29.png`
**Path:** `/Users/username/Pictures/Screenshots/スクリーンショット 2025-07-09 0.53.29.png`

---
```

## トラブルシューティング

### スクリーンショットが保存されない場合
1. Command+Shift+4（Controlキーなし）を使用
2. 設定をリセット:
   ```bash
   defaults write com.apple.screencapture location ~/Pictures/Screenshots
   killall SystemUIServer
   ```

### 監視が動作しない場合
1. fswatch がインストールされているか確認
2. 手動処理でテスト: `screenshot-process`
3. ログを確認: `~/Library/Logs/screenshot-monitor.log`

## 参考情報

- 元のGist: https://gist.github.com/miyashita337/9cf694f01ee19c3c16e6d34afab64e8e
- セットアップ日: 2025-07-09
- 対応OS: macOS Monterey (12.x), Ventura (13.x), Sonoma (14.x)