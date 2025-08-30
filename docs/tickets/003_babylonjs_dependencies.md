# チケット003: Babylon.js依存関係インストール

## 概要
Babylon.js 8.0とReact統合ライブラリをインストールし、WebXR開発に必要な依存関係をセットアップする

## 優先度
**Critical** - 3D/AR機能の基盤

## 推定時間
1時間

## 前提条件
- チケット002完了（Vite + TypeScript環境構築）

## 作業内容

### 1. Babylon.jsコアパッケージインストール
```bash
# Babylon.jsコア機能
npm install @babylonjs/core@^8.0.0

# Babylon.js GUIシステム
npm install @babylonjs/gui@^8.0.0

# 3Dモデルローダー（glTF、OBJ等）
npm install @babylonjs/loaders@^8.0.0

# マテリアルライブラリ
npm install @babylonjs/materials@^8.0.0
```

### 2. React統合ライブラリインストール
```bash
# react-babylonjsラッパー
npm install react-babylonjs
```

### 3. 状態管理ライブラリ
```bash
# Zustand（軽量状態管理）
npm install zustand
```

### 4. 型定義ファイル作成
```typescript
// src/types/babylon.d.ts
/// <reference types="@babylonjs/core" />
/// <reference types="@babylonjs/gui" />
/// <reference types="@babylonjs/loaders" />

declare module '*.babylon' {
  const content: string;
  export default content;
}

declare module '*.glb' {
  const content: string;
  export default content;
}

declare module '*.gltf' {
  const content: string;
  export default content;
}
```

### 5. Babylon.js初期化テストコード
```typescript
// src/scenes/TestScene.ts
import { Engine, Scene, FreeCamera, HemisphericLight, MeshBuilder, Vector3 } from '@babylonjs/core';

export class TestScene {
  private engine: Engine;
  private scene: Scene;
  
  constructor(canvas: HTMLCanvasElement) {
    this.engine = new Engine(canvas, true);
    this.scene = this.createScene();
    
    this.engine.runRenderLoop(() => {
      this.scene.render();
    });
    
    window.addEventListener('resize', () => {
      this.engine.resize();
    });
  }
  
  private createScene(): Scene {
    const scene = new Scene(this.engine);
    
    // カメラ
    const camera = new FreeCamera('camera', new Vector3(0, 5, -10), scene);
    camera.setTarget(Vector3.Zero());
    camera.attachControl(this.engine.getRenderingCanvas(), true);
    
    // ライト
    const light = new HemisphericLight('light', new Vector3(0, 1, 0), scene);
    
    // テスト用キューブ
    const box = MeshBuilder.CreateBox('box', { size: 2 }, scene);
    box.position.y = 1;
    
    return scene;
  }
  
  dispose(): void {
    this.scene.dispose();
    this.engine.dispose();
  }
}
```

### 6. Reactコンポーネントラッパー
```typescript
// src/components/BabylonCanvas.tsx
import React, { useEffect, useRef } from 'react';
import { TestScene } from '@scenes/TestScene';

const BabylonCanvas: React.FC = () => {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const sceneRef = useRef<TestScene | null>(null);
  
  useEffect(() => {
    if (canvasRef.current && !sceneRef.current) {
      sceneRef.current = new TestScene(canvasRef.current);
    }
    
    return () => {
      sceneRef.current?.dispose();
      sceneRef.current = null;
    };
  }, []);
  
  return (
    <canvas
      ref={canvasRef}
      style={{ width: '100%', height: '100vh' }}
      id="babylon-canvas"
    />
  );
};

export default BabylonCanvas;
```

## 技術詳細

### Babylon.js 8.0の新機能（2025年現在）
- **Depth Sensing**: Quest 3の深度情報を活用したAR体験
- **パススルー完全対応**: Mixed Reality機能のネイティブサポート
- **WebXR Layers**: パフォーマンス最適化
- **glTF 2.0完全サポート**: 最新のglTF拡張機能に対応

### react-babylonjsの利点
- **宣言的なAPI**: ReactコンポーネントとしてBabylon.jsを扱える
- **ホットリロード対応**: Reactのコンポーネントライフサイクルと統合
- **エスケープハッチ**: 必要に応じて命令的なコードも使用可能

### ツリーシェイキング最適化
- ES6モジュールを使用して必要な機能のみインポート
- @babylonjs/coreから個別にインポートすることでバンドルサイズ削減
- 本番ビルド時に不要なコードが自動除去される

## 完了条件
- [ ] Babylon.js 8.0以上がインストールされている
- [ ] react-babylonjsがインストールされている
- [ ] テストシーンが表示される
- [ ] キューブが画面に表示される
- [ ] リサイズ時に適切に描画が更新される
- [ ] コンソールにエラーが出ていない

## トラブルシューティング

### 問題: モジュールが見つからない
- 解決: package.jsonの"type": "module"を確認
- tsconfig.jsonの"module": "ESNext"を確認

### 問題: キャンバスが表示されない
- 解決: canvas要素にwidthとheightを明示的に設定
- useEffectの依存配列を確認

## 参考資料
- [Babylon.js公式ドキュメント](https://doc.babylonjs.com/)
- [react-babylonjs GitHub](https://github.com/brianzinn/react-babylonjs)
- [Babylon.js 8.0リリースノート](https://blogs.windows.com/windowsdeveloper/2025/04/03/part-3-babylon-js-8-0-gltf-usdz-and-webxr-advancements/)

## 次のステップ
→ 004_webxr_initialization.md