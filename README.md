# TransXR

AR Real-time Translation Assistant for Meta Quest 3

## 概要

TransXRは、Meta Quest 3向けのARリアルタイム翻訳アシスタントシステムです。ARグラスを通じて常時AIアシスタントが利用可能な社会を実現するプロトタイプを開発します。

### コアコンセプト
- **拡張メモリ機能**: 長期的な文脈情報の蓄積・活用
- **AIオーケストレーター**: 複数AIサービスを状況に応じて最適選択  
- **シームレスな入出力**: 視覚・音声による自然なインタラクション

## 技術スタック

- **3Dフレームワーク**: Babylon.js 8.0
- **UIフレームワーク**: React 18 + TypeScript
- **状態管理**: Zustand
- **ビルドツール**: Vite
- **AI/ML**: GPT-5 nano / Gemini 2.5 Flash-Lite

## セットアップ

### 必要な環境
- Node.js 20 LTS
- Chrome/Edge（WebXR対応）
- Quest 3実機またはエミュレータ

### インストール

```bash
npm install
```

### 開発サーバー起動

```bash
npm run dev
```

### ビルド

```bash
npm run build
```

## プロジェクト構造

```
src/
├── components/     # Reactコンポーネント
├── scenes/         # Babylon.jsシーン
├── plugins/        # プラグインシステム
├── core/           # コンテキストエンジン
├── services/       # API連携
├── utils/          # ユーティリティ
├── types/          # TypeScript型定義
└── main.tsx        # エントリーポイント
```

## 開発

プロジェクトの詳細な開発ガイドは `docs/tickets/` フォルダ内のチケットを参照してください。

## ライセンス

MIT