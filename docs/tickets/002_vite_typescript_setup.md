# チケット002: Vite + TypeScript環境構築

## 概要
Viteをビルドツールとして採用し、TypeScriptをセットアップして型安全な開発環境を構築する

## 優先度
**Critical** - 開発環境の基盤

## 推定時間
2時間

## 前提条件
- チケット001完了（プロジェクト初期化）

## 作業内容

### 1. ViteとReact・ TypeScript依存関係インストール
```bash
# ViteとReact関連パッケージ
npm install vite @vitejs/plugin-react

# Reactと型定義
npm install react react-dom
npm install -D @types/react @types/react-dom

# TypeScriptと関連ツール
npm install -D typescript
```

### 2. tsconfig.json作成
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@scenes/*": ["src/scenes/*"],
      "@plugins/*": ["src/plugins/*"],
      "@core/*": ["src/core/*"],
      "@services/*": ["src/services/*"],
      "@utils/*": ["src/utils/*"],
      "@types/*": ["src/types/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

### 3. tsconfig.node.json作成
```json
{
  "compilerOptions": {
    "composite": true,
    "skipLibCheck": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true,
    "strict": true
  },
  "include": ["vite.config.ts"]
}
```

### 4. vite.config.ts作成
```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
      '@scenes': path.resolve(__dirname, './src/scenes'),
      '@plugins': path.resolve(__dirname, './src/plugins'),
      '@core': path.resolve(__dirname, './src/core'),
      '@services': path.resolve(__dirname, './src/services'),
      '@utils': path.resolve(__dirname, './src/utils'),
      '@types': path.resolve(__dirname, './src/types')
    }
  },
  server: {
    port: 3000,
    https: true, // WebXRにHTTPS必須
    host: true // ネットワーク経由でアクセス可能に
  },
  build: {
    target: 'esnext',
    minify: 'esbuild',
    sourcemap: true
  },
  optimizeDeps: {
    include: ['@babylonjs/core', '@babylonjs/gui', 'react', 'react-dom']
  }
})
```

### 5. index.html作成
```html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="description" content="TransXR - AR Real-time Translation for Quest 3" />
    <title>TransXR</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

### 6. src/main.tsx作成
```typescript
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App'
import './index.css'

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

## 技術詳細

### Viteの利点（2025年現在）
- **超高速ホットリロード**: ミリ秒単位での更新
- **ESモジュールネイティブサポート**: Babylon.jsのツリーシェイキングに最適
- **プロダクションビルド最適化**: Rollupベースの高度な最適化

### TypeScript設定のポイント
- **strict: true**: 型安全性を最大限に活用
- **pathsエイリアス**: インポートパスを簡潔に
- **jsx: react-jsx**: React 17+の新しいJSXトランスフォーム

### HTTPS設定の重要性
- WebXR APIはセキュリティ上の理由からHTTPS必須
- 自己署名証明書でもQuest 3でのテストは可能
- ngrokやLocalTunnelを使用して外部からアクセス可能に

## 完了条件
- [ ] ViteとTypeScriptの依存関係がインストールされている
- [ ] tsconfig.jsonが適切に設定されている
- [ ] vite.config.tsが作成され、WebXR向け設定が完了
- [ ] npm run devで開発サーバーが起動する
- [ ] HTTPSでアクセスできる
- [ ] ビルドが成功する（npm run build）

## 参考資料
- [Vite公式ドキュメント](https://vite.dev/guide/)
- [TypeScript公式ドキュメント](https://www.typescriptlang.org/docs/)
- [Using Vite with Babylon.js](https://doc.babylonjs.com/guidedLearning/usingVite)

## 次のステップ
→ 003_babylonjs_dependencies.md