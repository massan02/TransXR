# チケット005: Quest 3パススルー実装

## 概要
Quest 3のMixed Reality機能を使用して、現実世界の映像を背景に表示し、ARコンテンツを重畳できるようにする

## 優先度
**Critical** - AR体験の中核機能

## 推定時間
2時間

## 前提条件
- チケット004完了（WebXR初期化）
- Quest 3実機またはImmersive Web Emulator
- Babylon.js 8.0以上

## 作業内容

### 1. パススルー機能マネージャー
```typescript
// src/scenes/PassthroughManager.ts
import {
  Scene,
  WebXRFeatureName,
  WebXRFeaturesManager,
  WebXRBackgroundRemover,
  Color3,
  StandardMaterial,
  MeshBuilder,
  Mesh
} from '@babylonjs/core';

export class PassthroughManager {
  private scene: Scene;
  private featuresManager: WebXRFeaturesManager | null = null;
  private backgroundRemover: WebXRBackgroundRemover | null = null;
  private passthroughEnabled: boolean = false;
  
  constructor(scene: Scene) {
    this.scene = scene;
  }
  
  async enablePassthrough(featuresManager: WebXRFeaturesManager): Promise<void> {
    this.featuresManager = featuresManager;
    
    try {
      // バックグラウンドリムーバー機能を有効化
      this.backgroundRemover = this.featuresManager.enableFeature(
        WebXRFeatureName.BACKGROUND_REMOVER,
        'latest',
        {
          backgroundMeshes: [], // 背景として扱うメッシュ
          ignoreDepthValues: false,
          environmentHelperRemovalFlags: {
            skyBox: true,
            ground: true
          }
        }
      ) as WebXRBackgroundRemover;
      
      // シーンの背景を透明に
      this.scene.clearColor = new Color3(0, 0, 0);
      this.scene.clearColor.a = 0;
      
      // 環境テクスチャを無効化
      if (this.scene.environmentTexture) {
        this.scene.environmentTexture.dispose();
        this.scene.environmentTexture = null;
      }
      
      this.passthroughEnabled = true;
      console.log('Passthrough enabled successfully');
      
    } catch (error) {
      console.error('Failed to enable passthrough:', error);
      // フォールバック: 半透明な背景を設定
      this.setupFallbackBackground();
    }
  }
  
  private setupFallbackBackground(): void {
    // パススルーが使えない場合の代替背景
    const fallbackPlane = MeshBuilder.CreatePlane(
      'fallbackBackground',
      { size: 100 },
      this.scene
    );
    
    fallbackPlane.position.z = 50;
    
    const material = new StandardMaterial('fallbackMat', this.scene);
    material.emissiveColor = new Color3(0.1, 0.1, 0.1);
    material.alpha = 0.3;
    material.backFaceCulling = false;
    
    fallbackPlane.material = material;
  }
  
  // パススルーの透明度を調整
  setPassthroughOpacity(opacity: number): void {
    if (!this.passthroughEnabled) return;
    
    // 0.0（完全に透明）から1.0（不透明）
    opacity = Math.max(0, Math.min(1, opacity));
    
    // ARコンテンツの不透明度を調整
    this.scene.meshes.forEach(mesh => {
      if (mesh.material && mesh.name !== 'fallbackBackground') {
        (mesh.material as StandardMaterial).alpha = opacity;
      }
    });
  }
  
  // パススルーのオン/オフ切り替え
  togglePassthrough(): void {
    if (this.passthroughEnabled) {
      this.disablePassthrough();
    } else if (this.featuresManager) {
      this.enablePassthrough(this.featuresManager);
    }
  }
  
  disablePassthrough(): void {
    if (this.backgroundRemover) {
      this.backgroundRemover.dispose();
      this.backgroundRemover = null;
    }
    
    // 背景色を元に戻す
    this.scene.clearColor = new Color3(0.2, 0.2, 0.3);
    this.scene.clearColor.a = 1;
    
    this.passthroughEnabled = false;
    console.log('Passthrough disabled');
  }
  
  isEnabled(): boolean {
    return this.passthroughEnabled;
  }
  
  dispose(): void {
    this.disablePassthrough();
  }
}
```

### 2. WebXRシーンへの統合
```typescript
// src/scenes/ARScene.ts
import {
  Engine,
  Scene,
  FreeCamera,
  Vector3,
  WebXRDefaultExperience,
  WebXRState
} from '@babylonjs/core';
import { PassthroughManager } from './PassthroughManager';

export class ARScene {
  private engine: Engine;
  private scene: Scene;
  private xrHelper: WebXRDefaultExperience | null = null;
  private passthroughManager: PassthroughManager;
  
  constructor(canvas: HTMLCanvasElement) {
    this.engine = new Engine(canvas, true);
    this.scene = new Scene(this.engine);
    this.passthroughManager = new PassthroughManager(this.scene);
    
    this.initScene();
    this.initXR();
    
    this.engine.runRenderLoop(() => {
      this.scene.render();
    });
  }
  
  private initScene(): void {
    // 基本カメラ（XRセッション外で使用）
    const camera = new FreeCamera(
      'camera',
      new Vector3(0, 1.6, -5),
      this.scene
    );
    camera.setTarget(Vector3.Zero());
    camera.attachControl(this.engine.getRenderingCanvas(), true);
    
    // ライトは最小限に（ARでは現実の照明を使用）
    this.scene.createDefaultLight();
  }
  
  private async initXR(): Promise<void> {
    try {
      // WebXRデフォルト体験を作成
      this.xrHelper = await this.scene.createDefaultXRExperienceAsync({
        uiOptions: {
          sessionMode: 'immersive-ar',
          referenceSpaceType: 'local-floor'
        },
        // Quest 3向け最適化
        optionalFeatures: true
      });
      
      // XRセッション開始時にパススルーを有効化
      this.xrHelper.baseExperience.onStateChangedObservable.add((state) => {
        if (state === WebXRState.IN_XR) {
          // XRセッション内でパススルーを有効化
          this.passthroughManager.enablePassthrough(
            this.xrHelper!.baseExperience.featuresManager
          );
        } else if (state === WebXRState.NOT_IN_XR) {
          // XRセッション終了時にパススルーを無効化
          this.passthroughManager.disablePassthrough();
        }
      });
      
    } catch (error) {
      console.error('WebXR initialization failed:', error);
    }
  }
  
  // パススルーの透明度を外部から調整
  setPassthroughOpacity(opacity: number): void {
    this.passthroughManager.setPassthroughOpacity(opacity);
  }
  
  // パススルーの切り替え
  togglePassthrough(): void {
    this.passthroughManager.togglePassthrough();
  }
  
  dispose(): void {
    this.passthroughManager.dispose();
    this.scene.dispose();
    this.engine.dispose();
  }
}
```

### 3. React UIコンポーネント
```typescript
// src/components/PassthroughControls.tsx
import React, { useState } from 'react';

interface PassthroughControlsProps {
  onOpacityChange: (opacity: number) => void;
  onToggle: () => void;
}

const PassthroughControls: React.FC<PassthroughControlsProps> = ({
  onOpacityChange,
  onToggle
}) => {
  const [opacity, setOpacity] = useState(1.0);
  const [isEnabled, setIsEnabled] = useState(false);
  
  const handleOpacityChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const newOpacity = parseFloat(e.target.value);
    setOpacity(newOpacity);
    onOpacityChange(newOpacity);
  };
  
  const handleToggle = () => {
    setIsEnabled(!isEnabled);
    onToggle();
  };
  
  return (
    <div style={{
      position: 'absolute',
      top: 20,
      left: 20,
      backgroundColor: 'rgba(0, 0, 0, 0.7)',
      padding: 15,
      borderRadius: 10,
      color: 'white'
    }}>
      <h3 style={{ margin: '0 0 10px 0' }}>Passthrough Controls</h3>
      
      <div style={{ marginBottom: 10 }}>
        <button
          onClick={handleToggle}
          style={{
            padding: '8px 16px',
            backgroundColor: isEnabled ? '#4CAF50' : '#f44336',
            color: 'white',
            border: 'none',
            borderRadius: 5,
            cursor: 'pointer'
          }}
        >
          {isEnabled ? 'Enabled' : 'Disabled'}
        </button>
      </div>
      
      <div>
        <label htmlFor="opacity-slider">
          AR Content Opacity: {opacity.toFixed(2)}
        </label>
        <input
          id="opacity-slider"
          type="range"
          min="0"
          max="1"
          step="0.1"
          value={opacity}
          onChange={handleOpacityChange}
          style={{ width: '100%', marginTop: 5 }}
        />
      </div>
      
      <div style={{ marginTop: 10, fontSize: 12, opacity: 0.8 }}>
        <p>Tips:</p>
        <ul style={{ margin: 5, paddingLeft: 20 }}>
          <li>Lower opacity = more visible real world</li>
          <li>Higher opacity = more visible AR content</li>
        </ul>
      </div>
    </div>
  );
};

export default PassthroughControls;
```

## 技術詳細

### Quest 3パススルーの特徴
- **高解像度カメラ**: 4MP×2のカラーカメラ
- **低遅延**: 12ms以下のパススルー遅延
- **深度センシング**: Babylon.js 8.0で対応

### WebXR Background Removerの仕組み
- DOM Overlayを使用して背景を透明化
- WebGLコンテキストのアルファブレンディング
- デプスバッファを使用したオクルージョン処理

### パフォーマンス最適化
- 不要なメッシュの非表示
- ライティング計算の最小化
- テクスチャメモリの節約

## 完了条件
- [ ] Quest 3でパススルーが有効になる
- [ ] 現実世界が背景として表示される
- [ ] ARコンテンツが重畳表示される
- [ ] 透明度調整が動作する
- [ ] パススルーのオン/オフが動作する
- [ ] フォールバック処理が動作する

## トラブルシューティング

### 問題: パススルーが有効にならない
- 解決: immersive-arモードを確認
- Quest 3のブラウザバージョンを確認（v35.3以上推奨）

### 問題: 背景が黒くなる
- 解決: clearColorのalpha値を0に設定
- WebGLコンテキストのalpha: trueを確認

## 参考資料
- [Babylon.js WebXR Background Remover](https://doc.babylonjs.com/features/featuresDeepDive/webXR/WebXRSelectedFeatures/WebXRBackgroundRemover)
- [Quest 3 Mixed Realityドキュメント](https://developer.oculus.com/documentation/web/webxr-mixed-reality/)
- [WebXR Augmented Reality Module](https://www.w3.org/TR/webxr-ar-module-1/)

## 次のステップ
→ 006_voice_input_webspeech.md