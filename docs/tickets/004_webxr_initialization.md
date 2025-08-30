# チケット004: WebXR初期化と基本セッション管理

## 概要
Babylon.jsを使用してWebXRセッションを初期化し、Quest 3でのAR体験を可能にする

## 優先度
**Critical** - AR機能のコア

## 推定時間
3時間

## 前提条件
- チケット003完了（Babylon.js依存関係）
- HTTPS環境が構築されている
- Quest 3ブラウザでアクセス可能

## 作業内容

### 1. WebXRヘルパークラス作成
```typescript
// src/scenes/WebXRManager.ts
import {
  Scene,
  WebXRDefaultExperience,
  WebXRSessionManager,
  WebXRState,
  WebXRFeatureName
} from '@babylonjs/core';

export class WebXRManager {
  private xrHelper: WebXRDefaultExperience | null = null;
  private scene: Scene;
  private onSessionStart?: () => void;
  private onSessionEnd?: () => void;
  
  constructor(scene: Scene) {
    this.scene = scene;
  }
  
  async initializeXR(): Promise<void> {
    try {
      // WebXRサポート確認
      if (!navigator.xr) {
        console.warn('WebXR is not supported in this browser');
        return;
      }
      
      // WebXRデフォルト体験を作成
      this.xrHelper = await this.scene.createDefaultXRExperienceAsync({
        // ARモードを優先
        uiOptions: {
          sessionMode: 'immersive-ar',
          referenceSpaceType: 'local-floor',
          onError: (error) => {
            console.error('WebXR Error:', error);
          }
        },
        // Quest 3向け最適化
        optionalFeatures: true
      });
      
      // セッション管理
      this.setupSessionListeners();
      
      console.log('WebXR initialized successfully');
    } catch (error) {
      console.error('Failed to initialize WebXR:', error);
      throw error;
    }
  }
  
  private setupSessionListeners(): void {
    if (!this.xrHelper) return;
    
    // XRセッション開始時
    this.xrHelper.baseExperience.sessionManager.onXRSessionInit.add(() => {
      console.log('XR Session started');
      this.onSessionStart?.();
    });
    
    // XRセッション終了時
    this.xrHelper.baseExperience.sessionManager.onXRSessionEnded.add(() => {
      console.log('XR Session ended');
      this.onSessionEnd?.();
    });
    
    // 状態変化の監視
    this.xrHelper.baseExperience.onStateChangedObservable.add((state) => {
      switch(state) {
        case WebXRState.IN_XR:
          console.log('Entered XR');
          break;
        case WebXRState.ENTERING_XR:
          console.log('Entering XR...');
          break;
        case WebXRState.EXITING_XR:
          console.log('Exiting XR...');
          break;
        case WebXRState.NOT_IN_XR:
          console.log('Not in XR');
          break;
      }
    });
  }
  
  // XRセッションがアクティブか確認
  isInXR(): boolean {
    return this.xrHelper?.baseExperience.state === WebXRState.IN_XR;
  }
  
  // 手動でXRセッションを開始
  async enterXR(): Promise<void> {
    if (this.xrHelper) {
      await this.xrHelper.baseExperience.enterXRAsync(
        'immersive-ar',
        'local-floor'
      );
    }
  }
  
  // XRセッションを終了
  async exitXR(): Promise<void> {
    if (this.xrHelper && this.isInXR()) {
      await this.xrHelper.baseExperience.exitXRAsync();
    }
  }
  
  dispose(): void {
    this.xrHelper?.dispose();
  }
}
```

### 2. ReactコンポーネントでのWebXR統合
```typescript
// src/components/WebXRScene.tsx
import React, { useEffect, useRef, useState } from 'react';
import { Engine, Scene, FreeCamera, HemisphericLight, Vector3 } from '@babylonjs/core';
import { WebXRManager } from '@scenes/WebXRManager';

const WebXRScene: React.FC = () => {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const engineRef = useRef<Engine | null>(null);
  const sceneRef = useRef<Scene | null>(null);
  const xrManagerRef = useRef<WebXRManager | null>(null);
  const [isXRSupported, setIsXRSupported] = useState(false);
  const [isInXR, setIsInXR] = useState(false);
  
  useEffect(() => {
    if (!canvasRef.current) return;
    
    // EngineとSceneの初期化
    engineRef.current = new Engine(canvasRef.current, true);
    sceneRef.current = new Scene(engineRef.current);
    
    // 基本的なシーン設定
    const camera = new FreeCamera('camera', new Vector3(0, 1.6, -3), sceneRef.current);
    camera.setTarget(Vector3.Zero());
    camera.attachControl(canvasRef.current, true);
    
    const light = new HemisphericLight('light', new Vector3(0, 1, 0), sceneRef.current);
    light.intensity = 0.7;
    
    // WebXR初期化
    xrManagerRef.current = new WebXRManager(sceneRef.current);
    
    const initXR = async () => {
      try {
        await xrManagerRef.current!.initializeXR();
        setIsXRSupported(true);
      } catch (error) {
        console.error('XR initialization failed:', error);
        setIsXRSupported(false);
      }
    };
    
    initXR();
    
    // レンダーループ
    engineRef.current.runRenderLoop(() => {
      sceneRef.current?.render();
    });
    
    // リサイズハンドラ
    const handleResize = () => {
      engineRef.current?.resize();
    };
    window.addEventListener('resize', handleResize);
    
    // クリーンアップ
    return () => {
      window.removeEventListener('resize', handleResize);
      xrManagerRef.current?.dispose();
      sceneRef.current?.dispose();
      engineRef.current?.dispose();
    };
  }, []);
  
  const handleEnterXR = async () => {
    try {
      await xrManagerRef.current?.enterXR();
      setIsInXR(true);
    } catch (error) {
      console.error('Failed to enter XR:', error);
    }
  };
  
  const handleExitXR = async () => {
    try {
      await xrManagerRef.current?.exitXR();
      setIsInXR(false);
    } catch (error) {
      console.error('Failed to exit XR:', error);
    }
  };
  
  return (
    <div style={{ position: 'relative', width: '100%', height: '100vh' }}>
      <canvas ref={canvasRef} style={{ width: '100%', height: '100%' }} />
      
      {/* WebXRコントロール */}
      <div style={{
        position: 'absolute',
        bottom: 20,
        right: 20,
        display: 'flex',
        gap: 10
      }}>
        {isXRSupported && (
          <>
            {!isInXR ? (
              <button
                onClick={handleEnterXR}
                style={{
                  padding: '10px 20px',
                  backgroundColor: '#007AFF',
                  color: 'white',
                  border: 'none',
                  borderRadius: 5,
                  cursor: 'pointer'
                }}
              >
                Enter AR
              </button>
            ) : (
              <button
                onClick={handleExitXR}
                style={{
                  padding: '10px 20px',
                  backgroundColor: '#FF3B30',
                  color: 'white',
                  border: 'none',
                  borderRadius: 5,
                  cursor: 'pointer'
                }}
              >
                Exit AR
              </button>
            )}
          </>
        )}
        
        {!isXRSupported && (
          <div style={{
            padding: '10px 20px',
            backgroundColor: '#FFA500',
            color: 'white',
            borderRadius: 5
          }}>
            WebXR not supported
          </div>
        )}
      </div>
    </div>
  );
};

export default WebXRScene;
```

### 3. フォールバック処理
```typescript
// src/utils/webxr-fallback.ts
export class WebXRFallback {
  static checkSupport(): {
    supported: boolean;
    reason?: string;
    recommendations?: string[];
  } {
    // WebXRサポート確認
    if (!('xr' in navigator)) {
      return {
        supported: false,
        reason: 'WebXR API not available',
        recommendations: [
          'Use a WebXR-compatible browser (Chrome, Edge)',
          'Enable WebXR flags in browser settings',
          'Update your browser to the latest version'
        ]
      };
    }
    
    // HTTPS確認
    if (window.location.protocol !== 'https:' && 
        window.location.hostname !== 'localhost') {
      return {
        supported: false,
        reason: 'HTTPS required for WebXR',
        recommendations: [
          'Use HTTPS connection',
          'Use localhost for development',
          'Deploy to HTTPS-enabled server'
        ]
      };
    }
    
    return { supported: true };
  }
  
  static async checkARSupport(): Promise<boolean> {
    if (!navigator.xr) return false;
    
    try {
      const isSupported = await navigator.xr.isSessionSupported('immersive-ar');
      return isSupported;
    } catch (error) {
      console.error('AR support check failed:', error);
      return false;
    }
  }
}
```

## 技術詳細

### WebXRセッションモード
- **immersive-ar**: AR体験用（Quest 3のパススルーを活用）
- **immersive-vr**: VR体験用（完全な仮想空間）
- **inline**: ブラウザ内での3D表示

### Reference Space Types
- **local-floor**: 床レベルを基準とした座標系
- **bounded-floor**: 境界付き空間
- **unbounded**: 無制限の空間

### Quest 3固有の最適化
- 120Hzリフレッシュレート対応
- パススルー機能の活用
- ハンドトラッキング対応（次チケット）

## 完了条件
- [ ] WebXRセッションが初期化される
- [ ] "Enter AR"ボタンが表示される
- [ ] Quest 3でARモードに入れる
- [ ] セッション終了が正常に動作する
- [ ] エラーハンドリングが実装されている
- [ ] フォールバックメッセージが適切に表示される

## テスト方法

### ローカルテスト（Quest 3実機）
1. ngrokまたはLocalTunnelでローカルサーバーを公開
   ```bash
   npx localtunnel --port 3000
   ```
2. Quest 3ブラウザでURLにアクセス
3. "Enter AR"ボタンをタップ

### デバッグ方法
- Chrome DevToolsのRemote Debuggingを使用
- adbコマンドでQuest 3のログを取得

## 参考資料
- [WebXR Device API仕様](https://www.w3.org/TR/webxr/)
- [Babylon.js WebXRドキュメント](https://doc.babylonjs.com/features/featuresDeepDive/webXR)
- [Meta Quest WebXR開発ガイド](https://developer.oculus.com/documentation/web/)

## 次のステップ
→ 005_passthrough_implementation.md