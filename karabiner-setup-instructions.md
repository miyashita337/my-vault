# Karabiner-Elements ダブルEnter設定手順

## 概要
Slack、Chrome（ChatGPT/YouTube）、VSCode（ClaudeCode）でEnterキーの誤送信を防ぐため、ダブルEnterでのみメッセージ送信を有効にする設定です。

## 前提条件
- Karabiner-Elementsがインストールされている
- macOSでの実行

## インストール手順

### 1. Karabiner-Elementsのインストール（未インストールの場合）
```bash
brew install --cask karabiner-elements
```

### 2. 設定ファイルの配置
作成した `karabiner-double-enter-config.json` を以下のディレクトリに配置：
```
~/.config/karabiner/assets/complex_modifications/
```

### 3. Karabiner-Elementsでの設定
1. Karabiner-Elementsを起動
2. 「Complex Modifications」タブを開く
3. 「Add predefined rule」をクリック
4. 「Double Enter for Specific Apps (Slack, Chrome, VSCode)」を選択
5. 「Enable」をクリック

## 動作対象アプリケーション
- **Slack**: `com.tinyspeck.slackmacgap`
- **Chrome**: `com.google.Chrome` (ChatGPT、YouTube含む)
- **VSCode**: `com.microsoft.VSCode` (ClaudeCode含む)

## 動作仕様
- **単発Enter**: 無効化（何もしない）
- **ダブルEnter**: 通常のEnter動作を実行
- **タイミング**: 500ms以内に2回目のEnterキーを押す必要がある

## 使用方法
1. 指定アプリケーションでテキストを入力
2. 送信したい場合は**Enter → Enter**（素早く2回）
3. 改行したい場合は**Shift + Enter**（従来通り）

## トラブルシューティング

### 設定が反映されない場合
1. Karabiner-EventViewerで「Frontmost Application」タブを確認
2. 対象アプリのBundle Identifierが正しいか確認
3. Karabiner-Elementsを再起動

### タイミングの調整
JSONファイル内の `to_delayed_action_delay_milliseconds` を調整（デフォルト500ms）

## 設定の無効化
必要に応じて、Karabiner-ElementsのComplex Modificationsタブで該当ルールを無効化できます。