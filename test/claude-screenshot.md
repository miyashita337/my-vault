現場のclaude-screenshotで問題があるみたいです

# win版
* 実際にWindows版を入れようとしたら、現場WindowsのClaudeはWSL版のUbuntuのみの環境でしか入らない、パスは以下が正解らしい
	* C:\Users\{ユーザー名}\Pictures\Screenshots
	* あとスクリーン・ショットファイル名には半角スペースがあるため、ダブルクォーテーションで囲まないと読み取れない

# mac版
前まで実装されてたのに「command+shift+control+4」での範囲キャプチャがいつの間にか消えてた、復活してほしい


不明点あればヒアリン


すいません、やはりinstall.psパスのところを修正してほしいい
>
     このガイドは、Windows環境（特にWSL環境）でclaude-screenshotを動作させるための修正方法を説明します。
     ## 現在の問題点
     1. **パスの問題**
        - 現在の実装はWSLのパスを想定していない
        - 正しいWindowsパス: `C:\Users\{ユーザー名}\Pictures\Screenshots`
        - WSLからのアクセス: `/mnt/c/Users/{ユーザー名}/Pictures/Screenshots`