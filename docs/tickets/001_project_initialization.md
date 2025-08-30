# チケット001: プロジェクト初期化

## 概要
TransXRプロジェクトのNode.js環境を初期化し、基本的なプロジェクト構造を構築する

## 優先度
**Critical** - すべての開発作業の前提条件

## 推定時間
1時間

## 前提条件
- Node.js 20 LTS インストール済み
- npmまたはyarnが利用可能
- Quest 3デバイスまたはエミュレータ準備済み

## 作業内容

### 1. プロジェクトディレクトリの作成
```bash
mkdir TransXR
cd TransXR
```

### 2. npm初期化
```bash
npm init -y
```

### 3. package.json基本設定
```json
{
  "name": "transxr",
  "version": "0.1.0",
  "description": "AR Real-time Translation Assistant for Quest 3",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint src --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "type-check": "tsc --noEmit"
  },
  "keywords": ["webxr", "babylon.js", "ar", "translation", "quest3"],
  "author": "",
  "license": "MIT"
}
```

### 4. .gitignore作成
```
node_modules/
dist/
.env
.env.local
*.log
.DS_Store
.vscode/
.idea/
coverage/
*.local
```

### 5. README.md作成
基本的なプロジェクト説明とセットアップ手順を記載

## 技術詳細

### package.json設定のポイント
- **"type": "module"** - ES6モジュールを使用（Babylon.jsのツリーシェイキング最適化のため）
- **scripts** - 開発、ビルド、型チェックのコマンドを定義

### ディレクトリ構造（推奨）
```
TransXR/
├── src/
│   ├── components/     # Reactコンポーネント
│   ├── scenes/         # Babylon.jsシーン
│   ├── plugins/        # プラグインシステム
│   ├── core/           # コンテキストエンジン
│   ├── services/       # API連携
│   ├── utils/          # ユーティリティ
│   ├── types/          # TypeScript型定義
│   └── main.tsx        # エントリーポイント
├── public/             # 静的ファイル
├── docs/               # ドキュメント
└── tests/              # テストファイル
```

## 完了条件
- [ ] package.jsonが作成され、基本設定が完了している
- [ ] .gitignoreが適切に設定されている
- [ ] 基本的なディレクトリ構造が作成されている
- [ ] READMEファイルが存在する
- [ ] git initが実行され、初期コミットが作成されている

## 参考資料
- [Node.js公式ドキュメント](https://nodejs.org/docs/latest-v20.x/api/)
- [npm package.json仕様](https://docs.npmjs.com/cli/v10/configuring-npm/package-json)

## 次のステップ
→ 002_vite_typescript_setup.md