# チケット008: AR字幕表示実装

## 概要
Babylon.js GUIシステムを使用して、3D空間内に翻訳字幕を表示し、話者の位置に追従するAR字幕機能を実装する

## 優先度
**Critical** - 翻訳結果の表示インターフェース

## 推定時間
4時間

## 前提条件
- チケット004完了（WebXR初期化）
- チケット005完了（パススルー実装）
- Babylon.js GUIライブラリ

## 作業内容

### 1. 字幕コンポーネントクラス
```typescript
// src/components/ar/SubtitleComponent.ts
import {
  Scene,
  Mesh,
  MeshBuilder,
  StandardMaterial,
  Color3,
  Vector3,
  TransformNode,
  Animation,
  AnimationKeys,
  IAnimationKey
} from '@babylonjs/core';
import {
  AdvancedDynamicTexture,
  Rectangle,
  TextBlock,
  Control
} from '@babylonjs/gui';

export interface SubtitleConfig {
  text: string;
  position?: Vector3;
  fontSize?: number;
  fontFamily?: string;
  color?: string;
  backgroundColor?: string;
  backgroundAlpha?: number;
  duration?: number;
  fadeIn?: boolean;
  fadeOut?: boolean;
}

export class SubtitleComponent {
  private scene: Scene;
  private plane: Mesh;
  private advancedTexture: AdvancedDynamicTexture;
  private textBlock: TextBlock;
  private backgroundRect: Rectangle;
  private container: TransformNode;
  private isVisible: boolean = false;
  
  constructor(scene: Scene, id: string) {
    this.scene = scene;
    
    // コンテナノード作成
    this.container = new TransformNode(`subtitle_container_${id}`, scene);
    
    // 字幕用プレーン作成
    this.plane = MeshBuilder.CreatePlane(
      `subtitle_plane_${id}`,
      {
        width: 3,
        height: 0.8,
        sideOrientation: Mesh.DOUBLESIDE
      },
      scene
    );
    
    this.plane.parent = this.container;
    this.plane.billboardMode = Mesh.BILLBOARDMODE_ALL; // 常にカメラに向く
    
    // GUIテクスチャ作成
    this.advancedTexture = AdvancedDynamicTexture.CreateForMesh(
      this.plane,
      1024,
      256
    );
    
    // 背景矩形
    this.backgroundRect = new Rectangle(`bg_${id}`);
    this.backgroundRect.width = 1;
    this.backgroundRect.height = 1;
    this.backgroundRect.cornerRadius = 20;
    this.backgroundRect.color = "white";
    this.backgroundRect.thickness = 0;
    this.backgroundRect.background = "black";
    this.backgroundRect.alpha = 0.7;
    this.advancedTexture.addControl(this.backgroundRect);
    
    // テキストブロック
    this.textBlock = new TextBlock(`text_${id}`);
    this.textBlock.text = "";
    this.textBlock.color = "white";
    this.textBlock.fontSize = 48;
    this.textBlock.fontFamily = "Noto Sans JP, Arial";
    this.textBlock.textWrapping = true;
    this.textBlock.resizeToFit = true;
    this.textBlock.paddingTop = "10px";
    this.textBlock.paddingBottom = "10px";
    this.textBlock.paddingLeft = "20px";
    this.textBlock.paddingRight = "20px";
    
    this.backgroundRect.addControl(this.textBlock);
    
    // 初期状態は非表示
    this.hide();
  }
  
  // 字幕表示
  show(config: SubtitleConfig): void {
    // テキスト設定
    this.textBlock.text = config.text;
    
    // スタイル設定
    if (config.fontSize) {
      this.textBlock.fontSize = config.fontSize;
    }
    
    if (config.fontFamily) {
      this.textBlock.fontFamily = config.fontFamily;
    }
    
    if (config.color) {
      this.textBlock.color = config.color;
    }
    
    if (config.backgroundColor) {
      this.backgroundRect.background = config.backgroundColor;
    }
    
    if (config.backgroundAlpha !== undefined) {
      this.backgroundRect.alpha = config.backgroundAlpha;
    }
    
    // 位置設定
    if (config.position) {
      this.container.position = config.position;
    }
    
    // 表示
    this.plane.isVisible = true;
    this.isVisible = true;
    
    // フェードインアニメーション
    if (config.fadeIn) {
      this.fadeIn(300);
    }
    
    // 自動非表示
    if (config.duration && config.duration > 0) {
      setTimeout(() => {
        if (config.fadeOut) {
          this.fadeOut(300, () => this.hide());
        } else {
          this.hide();
        }
      }, config.duration);
    }
  }
  
  // 字幕非表示
  hide(): void {
    this.plane.isVisible = false;
    this.isVisible = false;
  }
  
  // フェードイン
  private fadeIn(duration: number): void {
    const material = this.plane.material as StandardMaterial;
    if (!material) return;
    
    Animation.CreateAndStartAnimation(
      'fadeIn',
      material,
      'alpha',
      60,
      60 * (duration / 1000),
      0,
      1,
      Animation.ANIMATIONLOOPMODE_CONSTANT
    );
  }
  
  // フェードアウト
  private fadeOut(duration: number, callback?: () => void): void {
    const material = this.plane.material as StandardMaterial;
    if (!material) return;
    
    const animation = Animation.CreateAndStartAnimation(
      'fadeOut',
      material,
      'alpha',
      60,
      60 * (duration / 1000),
      1,
      0,
      Animation.ANIMATIONLOOPMODE_CONSTANT
    );
    
    if (animation && callback) {
      animation.onAnimationEnd = callback;
    }
  }
  
  // 位置更新
  updatePosition(position: Vector3): void {
    this.container.position = position;
  }
  
  // テキスト更新
  updateText(text: string): void {
    this.textBlock.text = text;
  }
  
  // サイズ調整
  resize(width: number, height: number): void {
    this.plane.scaling = new Vector3(width, height, 1);
  }
  
  // 取得メソッド
  getPosition(): Vector3 {
    return this.container.position;
  }
  
  getText(): string {
    return this.textBlock.text;
  }
  
  getVisibility(): boolean {
    return this.isVisible;
  }
  
  // 破棄
  dispose(): void {
    this.advancedTexture.dispose();
    this.plane.dispose();
    this.container.dispose();
  }
}
```

### 2. 字幕マネージャー
```typescript
// src/managers/SubtitleManager.ts
import { Scene, Vector3, Camera } from '@babylonjs/core';
import { SubtitleComponent, SubtitleConfig } from '@components/ar/SubtitleComponent';

export interface SubtitleEntry {
  id: string;
  text: string;
  language: string;
  timestamp: Date;
  speaker?: string;
  position?: Vector3;
}

export class SubtitleManager {
  private scene: Scene;
  private subtitles: Map<string, SubtitleComponent>;
  private activeSubtitles: SubtitleEntry[];
  private maxSubtitles: number = 5;
  private defaultDuration: number = 5000; // 5秒
  private displayMode: 'floating' | 'fixed' | 'speaker' = 'floating';
  
  constructor(scene: Scene) {
    this.scene = scene;
    this.subtitles = new Map();
    this.activeSubtitles = [];
  }
  
  // 新しい字幕を表示
  displaySubtitle(entry: SubtitleEntry): void {
    // 既存の字幕を再利用または新規作成
    let subtitle = this.subtitles.get(entry.id);
    
    if (!subtitle) {
      subtitle = new SubtitleComponent(this.scene, entry.id);
      this.subtitles.set(entry.id, subtitle);
    }
    
    // 位置計算
    const position = this.calculatePosition(entry);
    
    // 表示設定
    const config: SubtitleConfig = {
      text: entry.text,
      position,
      fontSize: this.calculateFontSize(position),
      color: this.getTextColor(entry.language),
      backgroundColor: this.getBackgroundColor(entry.speaker),
      backgroundAlpha: 0.8,
      duration: this.defaultDuration,
      fadeIn: true,
      fadeOut: true
    };
    
    subtitle.show(config);
    
    // アクティブリストに追加
    this.activeSubtitles.push(entry);
    
    // 古い字幕を削除
    this.cleanupOldSubtitles();
  }
  
  // 位置計算
  private calculatePosition(entry: SubtitleEntry): Vector3 {
    const camera = this.scene.activeCamera;
    if (!camera) return Vector3.Zero();
    
    switch (this.displayMode) {
      case 'fixed':
        // 画面下部固定
        return camera.position.add(
          camera.getForwardRay().direction.scale(3)
        ).add(new Vector3(0, -1, 0));
        
      case 'speaker':
        // 話者の位置に表示
        if (entry.position) {
          return entry.position.add(new Vector3(0, 0.5, 0));
        }
        // フォールバックでfloating
        
      case 'floating':
      default:
        // カメラ前方に浮遊
        const offset = this.getStackOffset();
        return camera.position.add(
          camera.getForwardRay().direction.scale(2.5)
        ).add(offset);
    }
  }
  
  // スタック表示用オフセット
  private getStackOffset(): Vector3 {
    const activeCount = this.activeSubtitles.filter(
      s => Date.now() - s.timestamp.getTime() < this.defaultDuration
    ).length;
    
    return new Vector3(0, -0.3 * activeCount, 0);
  }
  
  // フォントサイズ計算（距離に応じて）
  private calculateFontSize(position: Vector3): number {
    const camera = this.scene.activeCamera;
    if (!camera) return 48;
    
    const distance = Vector3.Distance(camera.position, position);
    
    // 距離に応じてフォントサイズ調整
    if (distance < 2) return 36;
    if (distance < 3) return 48;
    if (distance < 5) return 60;
    return 72;
  }
  
  // 言語別テキスト色
  private getTextColor(language: string): string {
    const colors: Record<string, string> = {
      'ja-JP': '#FFFFFF', // 白
      'en-US': '#E3F2FD', // 淡い青
      'zh-CN': '#FFF3E0', // 淡いオレンジ
      'ko-KR': '#F3E5F5'  // 淡い紫
    };
    
    return colors[language] || '#FFFFFF';
  }
  
  // 話者別背景色
  private getBackgroundColor(speaker?: string): string {
    if (!speaker) return '#000000';
    
    // 話者にIDを割り当てて色分け
    const hash = this.hashCode(speaker);
    const colors = [
      '#1A237E', // 濃紺
      '#B71C1C', // 濃い赤
      '#1B5E20', // 濃い緑
      '#E65100', // 濃いオレンジ
      '#4A148C'  // 濃い紫
    ];
    
    return colors[Math.abs(hash) % colors.length];
  }
  
  // 文字列ハッシュ
  private hashCode(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash;
    }
    return hash;
  }
  
  // 古い字幕を削除
  private cleanupOldSubtitles(): void {
    const now = Date.now();
    
    // 期限切れの字幕を削除
    this.activeSubtitles = this.activeSubtitles.filter(entry => {
      const age = now - entry.timestamp.getTime();
      
      if (age > this.defaultDuration + 1000) {
        // 表示終了
        const subtitle = this.subtitles.get(entry.id);
        if (subtitle) {
          subtitle.hide();
        }
        return false;
      }
      
      return true;
    });
    
    // 最大数を超えたら古いものから削除
    while (this.activeSubtitles.length > this.maxSubtitles) {
      const oldest = this.activeSubtitles.shift();
      if (oldest) {
        const subtitle = this.subtitles.get(oldest.id);
        if (subtitle) {
          subtitle.hide();
        }
      }
    }
  }
  
  // 表示モード切り替え
  setDisplayMode(mode: 'floating' | 'fixed' | 'speaker'): void {
    this.displayMode = mode;
    
    // 現在表示中の字幕を再配置
    this.activeSubtitles.forEach(entry => {
      const subtitle = this.subtitles.get(entry.id);
      if (subtitle) {
        const newPosition = this.calculatePosition(entry);
        subtitle.updatePosition(newPosition);
      }
    });
  }
  
  // 表示時間設定
  setDuration(duration: number): void {
    this.defaultDuration = duration;
  }
  
  // 最大表示数設定
  setMaxSubtitles(max: number): void {
    this.maxSubtitles = max;
    this.cleanupOldSubtitles();
  }
  
  // すべての字幕をクリア
  clearAll(): void {
    this.subtitles.forEach(subtitle => {
      subtitle.hide();
    });
    this.activeSubtitles = [];
  }
  
  // リソース解放
  dispose(): void {
    this.subtitles.forEach(subtitle => {
      subtitle.dispose();
    });
    this.subtitles.clear();
    this.activeSubtitles = [];
  }
}
```

### 3. React統合コンポーネント
```typescript
// src/components/ARSubtitleOverlay.tsx
import React, { useEffect, useRef, useState } from 'react';
import { SubtitleManager, SubtitleEntry } from '@managers/SubtitleManager';
import { Scene, Vector3 } from '@babylonjs/core';

interface ARSubtitleOverlayProps {
  scene: Scene;
  onTranscript: (text: string, language: string) => void;
}

const ARSubtitleOverlay: React.FC<ARSubtitleOverlayProps> = ({
  scene,
  onTranscript
}) => {
  const [displayMode, setDisplayMode] = useState<'floating' | 'fixed' | 'speaker'>('floating');
  const [duration, setDuration] = useState(5000);
  const [maxSubtitles, setMaxSubtitles] = useState(5);
  const subtitleManager = useRef<SubtitleManager | null>(null);
  
  useEffect(() => {
    if (!scene) return;
    
    // 字幕マネージャー初期化
    subtitleManager.current = new SubtitleManager(scene);
    
    return () => {
      subtitleManager.current?.dispose();
    };
  }, [scene]);
  
  // 字幕表示関数
  const displaySubtitle = (
    text: string,
    language: string,
    speaker?: string,
    position?: Vector3
  ) => {
    if (!subtitleManager.current) return;
    
    const entry: SubtitleEntry = {
      id: `subtitle_${Date.now()}`,
      text,
      language,
      timestamp: new Date(),
      speaker,
      position
    };
    
    subtitleManager.current.displaySubtitle(entry);
  };
  
  // 表示モード変更
  const handleModeChange = (mode: 'floating' | 'fixed' | 'speaker') => {
    setDisplayMode(mode);
    subtitleManager.current?.setDisplayMode(mode);
  };
  
  // 表示時間変更
  const handleDurationChange = (newDuration: number) => {
    setDuration(newDuration);
    subtitleManager.current?.setDuration(newDuration);
  };
  
  // 最大表示数変更
  const handleMaxSubtitlesChange = (max: number) => {
    setMaxSubtitles(max);
    subtitleManager.current?.setMaxSubtitles(max);
  };
  
  // テスト用サンプル表示
  const showTestSubtitle = () => {
    displaySubtitle(
      'こんにちは、テスト字幕です',
      'ja-JP',
      'Speaker1'
    );
    
    setTimeout(() => {
      displaySubtitle(
        'Hello, this is a test subtitle',
        'en-US',
        'Speaker2'
      );
    }, 1000);
  };
  
  // コントロールUI
  return (
    <div style={{
      position: 'absolute',
      top: 20,
      right: 20,
      backgroundColor: 'rgba(0, 0, 0, 0.7)',
      padding: 15,
      borderRadius: 10,
      color: 'white',
      minWidth: 200
    }}>
      <h3 style={{ margin: '0 0 15px 0' }}>Subtitle Settings</h3>
      
      {/* 表示モード */}
      <div style={{ marginBottom: 10 }}>
        <label>Display Mode:</label>
        <select
          value={displayMode}
          onChange={(e) => handleModeChange(e.target.value as any)}
          style={{
            width: '100%',
            padding: 5,
            marginTop: 5,
            backgroundColor: '#333',
            color: 'white',
            border: '1px solid #555',
            borderRadius: 3
          }}
        >
          <option value="floating">Floating</option>
          <option value="fixed">Fixed Bottom</option>
          <option value="speaker">Near Speaker</option>
        </select>
      </div>
      
      {/* 表示時間 */}
      <div style={{ marginBottom: 10 }}>
        <label>Duration: {duration / 1000}s</label>
        <input
          type="range"
          min="2000"
          max="10000"
          step="1000"
          value={duration}
          onChange={(e) => handleDurationChange(Number(e.target.value))}
          style={{ width: '100%', marginTop: 5 }}
        />
      </div>
      
      {/* 最大表示数 */}
      <div style={{ marginBottom: 10 }}>
        <label>Max Subtitles: {maxSubtitles}</label>
        <input
          type="range"
          min="1"
          max="10"
          value={maxSubtitles}
          onChange={(e) => handleMaxSubtitlesChange(Number(e.target.value))}
          style={{ width: '100%', marginTop: 5 }}
        />
      </div>
      
      {/* テストボタン */}
      <button
        onClick={showTestSubtitle}
        style={{
          width: '100%',
          padding: '8px',
          backgroundColor: '#4CAF50',
          color: 'white',
          border: 'none',
          borderRadius: 5,
          cursor: 'pointer',
          marginTop: 10
        }}
      >
        Test Subtitle
      </button>
      
      {/* クリアボタン */}
      <button
        onClick={() => subtitleManager.current?.clearAll()}
        style={{
          width: '100%',
          padding: '8px',
          backgroundColor: '#f44336',
          color: 'white',
          border: 'none',
          borderRadius: 5,
          cursor: 'pointer',
          marginTop: 5
        }}
      >
        Clear All
      </button>
    </div>
  );
};

export default ARSubtitleOverlay;
```

## 技術詳細

### Babylon.js GUIシステム
- **AdvancedDynamicTexture**: 3Dメッシュ上に2D GUIを表示
- **Billboard Mode**: 常にカメラに向くように自動回転
- **テキストラップ**: 長いテキストを自動折り返し

### 日本語フォント対応
- **Noto Sans JP**: Google Fontsから読み込み
- **Web Font Loader**: 動的フォント読み込み
- **フォールバック**: Arialやシステムフォント

### パフォーマンス最適化
- **オブジェクトプール**: 字幕コンポーネントの再利用
- **テクスチャサイズ**: 1024x256でバランス
- **自動削除**: 古い字幕の自動クリーンアップ

## 完了条件
- [ ] 字幕が3D空間に表示される
- [ ] 日本語テキストが正しく表示される
- [ ] フェードイン/アウトが動作する
- [ ] 複数字幕のスタック表示が動作する
- [ ] 表示モード切り替えが動作する
- [ ] カメラ追従（Billboard）が動作する
- [ ] 距離に応じたサイズ調整が動作する

## トラブルシューティング

### 問題: 日本語フォントが表示されない
- 解決: Web Fontをロード
- Noto Sans JPをCDNから読み込み
- フォールバックフォントを設定

### 問題: 字幕が読みにくい
- 解決: 背景の透明度調整
- フォントサイズを距離に応じて調整
- コントラストを高める

### 問題: パフォーマンスが低い
- 解決: テクスチャサイズを最適化
- 字幕オブジェクトをプール化
- 最大表示数を制限

## 参考資料
- [Babylon.js GUIドキュメント](https://doc.babylonjs.com/features/featuresDeepDive/gui/gui)
- [Babylon.js Billboard Mode](https://doc.babylonjs.com/features/featuresDeepDive/mesh/billboards)
- [Babylon.js Dynamic Texture](https://doc.babylonjs.com/features/featuresDeepDive/materials/using/dynamicTexture)

## 次のステップ
→ 009_context_engine_implementation.md