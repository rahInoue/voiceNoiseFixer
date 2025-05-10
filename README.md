## まるっとボイス エディター サービス仕様書

### 1. サービス概要

**まるっとボイス エディター** は、ブラウザ上で音声ファイルをアップロードし、AI／Web Audio API によるノイズ除去・整音・EQ・コンプレッサー・リミッター処理をリアルタイムにプレビューし、加工後の音声をダウンロードできるSPA（シングルページアプリ）です。

### 2. 主な機能

1. **音声アップロード**

    * ユーザーのローカル音声ファイル（WAV・MP3など）を `<input type="file">` で受け取る
2. **ノイズ除去ブレンド**

    * RNNoise（WebAssembly）によるノイズ除去とオリジナル音声のミックス比率を 0.00〜1.00 で調整
3. **音量調整（Gain）**

    * 値域 0.1〜2.0 で最終出力音量を変更
4. **コンプレッサー**

    * Web Audio API `DynamicsCompressorNode` 使用
    * 設定可能パラメータ：Threshold
5. **EQ（3バンド）**

    * Web Audio API `BiquadFilterNode` 使用
    * Low（lowshelf）, Mid（peaking）, High（highshelf）各バンドのゲイン調整
6. **リミッター**

    * DynamicsCompressorNode を利用したブリックウォールリミッティング
    * 設定可能パラメータ：閾値（Threshold）
7. **波形ビジュアライザー**

    * `AnalyserNode.getByteTimeDomainData()` を Canvas にリアルタイム描画
8. **処理＆ダウンロード**

    * MediaRecorder によるストリーム録音 → WebM 形式でダウンロード

### 3. ユーザー操作フロー

1. ファイル選択
2. パラメータ調整（スライダー操作）→ Canvas で波形変化を確認
3. 「処理してダウンロード」ボタン押下
4. 自動的に音声処理実行 → 処理後ファイルをダウンロード

### 4. 技術スタック

* フレームワーク：Nuxt 3（SPA モード／SSR オフ）
* 言語：Vue 3 Composition API + TypeScript 設定（nuxt.config.ts）
* UI：Tailwind CSS
* 音声処理：

    * ノイズ除去：`@shiguredo/rnnoise-wasm`（WebAssembly）
    * Web Audio API（GainNode, DynamicsCompressorNode, BiquadFilterNode, AnalyserNode）
* ビルド：Vite
* デプロイ：Cloudflare Pages

### 5. アセット配置

* `.wasm` ファイル：`nuxt.config.ts` で `assetsInclude: ['**/*.wasm']` を指定し、`/assets/` 以下へ出力
* wasm ロード時に `Rnnoise.load({ locateFile: file => '/_nuxt/assets/' + file })` で正確なパス指定

### 6. パフォーマンス要件・制限

* 最大音声長：現状メモリ上処理に依存（推奨 3 分以内）
* フレームサイズ：480 サンプル単位で分割処理
* ブラウザ要件：WASM と Web Audio API 対応（Chrome/Firefox/Edge/Safari 最新版推奨）

### 7. 今後の拡張候補

* リアルタイムプレビュー再生中の調整反映（オートメーション）
* スペクトラムアナライザー表示（周波数ドメイン）
* プリセット保存・読み込み機能
* 複数ファイル一括処理（バッチモード）
* モバイル端末最適化（WebAudioWorker 化）

---

以上が現状の機能一覧と技術仕様です。要件追加や修正があればお知らせください。
