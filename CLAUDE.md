# TransXR開発ガイド - Claude/AI駆動開発向け

## 🎯 プロジェクト概要

**TransXR**は、Meta Quest 3向けのARリアルタイム翻訳アシスタントシステムです。
ARグラスを通じて常時AIアシスタントが利用可能な社会を実現するプロトタイプを開発します。

### コアコンセプト
- **拡張メモリ機能**: 長期的な文脈情報の蓄積・活用
- **AIオーケストレーター**: 複数AIサービスを状況に応じて最適選択  
- **シームレスな入出力**: 視覚・音声による自然なインタラクション

---

## 🚀 クイックスタート

### 必要なツール
```bash
# 必須
- Node.js 20 LTS
- npm or yarn
- Chrome/Edge（WebXR対応）
- Quest 3実機またはエミュレータ

# AI開発支援
- Claude Code（推奨）
- Cursor（オプション）
```

### プロジェクト初期化
```bash
# チケット001を参照して実装
cd TransXR
npm init -y
# 以降、docs/tickets/001_project_initialization.mdに従う
```

---

## 📚 アーキテクチャ

### システム構成（ハイブリッド型）
```
コアプラットフォーム
├── コンテキストエンジン（拡張メモリ）
├── AI オーケストレーター
└── プラグインシステム
    ├── 翻訳プラグイン ← MVP優先
    ├── 質問応答プラグイン
    └── 日記プラグイン
```

### 技術スタック（2025年1月確定）

#### フロントエンド
- **3Dフレームワーク**: Babylon.js 8.0（Quest 3完全対応）
- **UIフレームワーク**: React 18 + TypeScript
- **状態管理**: Zustand
- **ビルドツール**: Vite

#### バックエンド
- **ランタイム**: Node.js 20 LTS
- **APIフレームワーク**: Fastify
- **リアルタイム通信**: Socket.io
- **データベース**: PostgreSQL + Prisma
- **キャッシュ**: Redis

#### AI/ML
- **音声認識**: Web Speech API → Whisper API（高精度時）
- **翻訳**: GPT-5 nano ($0.05/1M) / Gemini 2.5 Flash-Lite ($0.10/1M)
- **画像認識**: GPT-5 nano（将来実装）

---

## 🎪 MVP機能（2週間で実装）

### 1. リアルタイム翻訳機能
- **対応言語**: 日本語、英語、中国語、韓国語
- **表示方式**: AR字幕オーバーレイ（話者の近く）
- **遅延**: 最大2秒
- **実装チケット**: 001-008

### 2. コンテキストエンジン（基盤）
- **データ収集**: 視覚30秒ごと、音声継続的
- **保存期間**: 詳細24時間、要約30日
- **プライバシー**: 顔自動ぼかし

### 3. WebXR基本機能
- **パススルー**: Quest 3 Mixed Reality対応
- **ハンドトラッキング**: 基本ジェスチャー
- **音声コマンド**: 基本操作

---

## 📝 実装チケット一覧

### 基盤構築（必須）
1. `001_project_initialization.md` - プロジェクト初期化
2. `002_vite_typescript_setup.md` - Vite + TypeScript環境
3. `003_babylonjs_dependencies.md` - Babylon.js依存関係
4. `004_webxr_initialization.md` - WebXR初期化

### AR機能（MVP）
5. `005_passthrough_implementation.md` - Quest 3パススルー
6. `006_voice_input_webspeech.md` - 音声入力実装
7. `007_translation_api_integration.md` - 翻訳API連携
8. `008_ar_subtitle_display.md` - AR字幕表示

各チケットには完全な実装コード、トラブルシューティング、参考資料が含まれています。

---

## 🤖 AI駆動開発の進め方

### Claude Codeでの開発フロー

```bash
# 1. 環境構築
"docs/tickets/001を実装してください"
→ 自動でpackage.json作成、ディレクトリ構築

# 2. 機能実装
"docs/tickets/004のWebXR初期化を実装"
→ 完全なコードを生成、動作確認まで

# 3. エラー対処
"npm run devで'Module not found: @babylonjs/core'エラー"
→ 修正コマンドと解決策を提示
```

### 効率的な指示の出し方

#### ✅ 良い例
```
"チケット007を参考に、GPT-5 nanoを使った日英翻訳機能を実装してください。
APIキーは環境変数から読み込み、キャッシュ機能も含めてください。"
```

#### ❌ 悪い例
```
"翻訳機能作って"
```

### デバッグのコツ
1. **エラーメッセージは完全にコピー**
2. **実行したコマンドを明記**
3. **期待する動作を説明**
4. **試した解決策を共有**

---

## 🧪 テスト環境

### Quest 3実機テスト
```bash
# 1. ローカルサーバー起動
npm run dev

# 2. ngrokでHTTPS公開
npx localtunnel --port 3000

# 3. Quest 3ブラウザでアクセス
# URLをQuest 3で開く
```

### デバッグツール
- Chrome DevTools Remote Debugging
- adbコマンドでログ取得
- Immersive Web Emulator（Chrome拡張）

---

## 📊 パフォーマンス目標

| 項目 | 目標値 | 測定方法 |
|-----|--------|----------|
| 起動時間 | < 3秒 | アプリ起動から利用可能まで |
| レスポンス | < 100ms | ユーザー操作から反応まで |
| フレームレート | 60fps以上 | 通常使用時 |
| 翻訳遅延 | < 2秒 | 音声入力から字幕表示まで |
| バッテリー | 3時間以上 | 連続使用 |

---

## 🔒 セキュリティ・プライバシー

### 実装済み対策
- **顔検出時の自動ぼかし処理**
- **ローカル優先処理**（可能な限り）
- **データ暗号化**: TLS 1.3（通信）、AES-256（保存）
- **録音・撮影の同意取得UI**

### APIキー管理
```javascript
// .env.localファイルで管理
OPENAI_API_KEY=your_key_here
GOOGLE_AI_API_KEY=your_key_here

// 絶対にコミットしない
// .gitignoreに追加済み
```

---

## 💡 開発のヒント

### Babylon.js WebXR
```javascript
// WebXR有効化は1行で完了
const xrHelper = await scene.createDefaultXRExperienceAsync();
```

### 日本語フォント対応
```javascript
// Noto Sans JPをCDNから読み込み
// Babylon.js GUIで日本語表示可能
```

### コスト最適化
- **GPT-5 nano**: 高速・低コスト（$0.05/1M）
- **Gemini 2.5 Flash-Lite**: バランス型（$0.10/1M）
- **キャッシュ活用**: 同じ翻訳を再利用

---

## 📈 開発ロードマップ

### Phase 1: WebXR MVP（Week 1-2）
- [x] 要件定義・技術選定
- [ ] 開発環境構築（チケット001-003）
- [ ] WebXR基本実装（チケット004-005）
- [ ] 翻訳機能実装（チケット006-008）

### Phase 2: 機能拡張（Week 3-4）
- [ ] コンテキストエンジン強化
- [ ] 複数話者対応
- [ ] 設定UI実装

### Phase 3: 最適化（Week 5-6）
- [ ] パフォーマンスチューニング
- [ ] バッテリー最適化
- [ ] UIブラッシュアップ

---

## ❓ トラブルシューティング

### よくある問題と解決策

#### WebXRが動作しない
- HTTPS環境を確認（localhost or ngrok）
- Chrome/Edgeの最新版を使用
- Quest 3ブラウザのWebXRフラグ確認

#### 日本語が文字化けする
- フォントファイルの読み込み確認
- UTF-8エンコーディング設定
- Babylon.js GUIのフォント設定

#### API料金が高い
- キャッシュ実装を確認
- 不要なAPI呼び出しを削減
- GPT-5 nanoへの切り替え検討

---

## 📖 参考資料

### 公式ドキュメント
- [Babylon.js WebXR](https://doc.babylonjs.com/features/featuresDeepDive/webXR)
- [Meta Quest WebXR](https://developer.oculus.com/documentation/web/)
- [OpenAI GPT-5 API](https://platform.openai.com/docs/models/gpt-5)
- [Google AI Studio](https://ai.google.dev/)

### 学習リソース
- [Microsoft Learn Babylon.js WebXR](https://learn.microsoft.com/en-us/windows/mixed-reality/develop/javascript/tutorials/babylonjs-webxr-helloworld/introduction-01)
- [Immersive Web Developer](https://immersiveweb.dev/)
- [JavaScript Primer（日本語）](https://jsprimer.net/)

### コミュニティ
- [Babylon.js Forum](https://forum.babylonjs.com/)
- [WebXR Discord](https://discord.gg/webxr)

---

## 🎓 必要スキルと学習パス

### 最小限必要なスキル（1-2週間）
1. **HTML/CSS基礎**（3-5日）
2. **JavaScript基礎**（1週間）
3. **npm基本コマンド**（1日）

### AI駆動開発でカバー可能
- Babylon.js詳細実装
- WebXR API
- TypeScript型定義
- React最適化
- Socket.io実装

### 推奨学習順序
1. JavaScript基礎 → 2. npm操作 → 3. Claude Codeで実装開始

---

## 🚦 開発開始チェックリスト

- [ ] Node.js 20 LTSインストール済み
- [ ] VS Codeインストール済み
- [ ] Claude Code/Cursorセットアップ済み
- [ ] Quest 3またはエミュレータ準備
- [ ] OpenAI/Google AIのAPIキー取得
- [ ] このCLAUDE.mdを読了
- [ ] docs/tickets/を確認

準備ができたら、`docs/tickets/001_project_initialization.md`から開始してください！

---

*最終更新: 2025年1月30日*
*プロジェクトバージョン: 0.1.0*