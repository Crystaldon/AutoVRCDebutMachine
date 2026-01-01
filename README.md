# VRChat Joke Tool

Web Speech APIを使用した、特定のワード（「ぶいあーるちゃっとで」など）に反応して音声を再生するジョークツールです。

## セットアップ

1. 依存関係のインストール
   ```bash
   npm install
   ```

2. 音声ファイルの準備
   - `public` フォルダ内に `alert.mp3` という名前で再生したい音声ファイルを配置してください。
   - ※ファイルがない場合、コンソールにエラーが出ますがアプリは動作します。

3. 開発サーバーの起動
   ```bash
   npm run dev
   ```

## デプロイ (GitHub Pages)

1. ビルド
   ```bash
   npm run build
   ```

2. `dist` フォルダの中身を GitHub Pages として公開してください。
   - `vite.config.js` で `base: './'` を設定済みなので、サブディレクトリでも動作します。

## 対応ブラウザ
- Google Chrome (PC/Android)
- Microsoft Edge
- その他 Web Speech API 対応ブラウザ
- ※iOS Safari は Web Speech API の挙動が異なるため、動作しない場合があります。

