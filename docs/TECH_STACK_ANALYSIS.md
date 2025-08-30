# 技術スタック分析（2025年1月時点）

## WebXRフレームワーク比較

### 最新状況サマリー

| フレームワーク | バージョン | Quest 3対応 | 更新頻度 | 特徴 |
|--------------|-----------|------------|---------|------|
| **Babylon.js** | 8.0 (2025年4月) | ◎完全対応 | 活発 | Depth Sensing、パススルー対応 |
| **Three.js + R3F** | @react-three/xr 6.6.25 | ○対応 | 活発（9日前更新） | React統合、メッシュ検出 |
| **A-Frame** | 1.0.0+ | △一部問題 | 中程度 | Quest 3コントローラー認識問題報告 |

### パフォーマンス実績
- **A-Frame**: Quest 3で120fps達成可能（最大フレームレート）
- **Babylon.js**: 90-120fps（最適化により）
- **Three.js**: 実装依存（開発者責任）

## 推奨技術スタック

### フロントエンド（WebXR）

**第1選択: Babylon.js**
```yaml
理由:
  - Quest 3完全対応（2025年4月のv8.0）
  - Depth Sensing対応
  - WebXR組み込み済み（1行で有効化）
  - パススルー機能対応
  - 後方互換性重視
```

**第2選択: React Three Fiber + @react-three/xr**
```yaml
理由:
  - React開発者に親和性高い
  - 最新版が活発に更新（2025年1月）
  - Meta Quest エミュレータ対応
  - UIKit統合済み
注意:
  - Three.js知識必要
  - パフォーマンス最適化は開発者責任
```

### バックエンド

```yaml
ランタイム: Node.js 20 LTS
フレームワーク: Fastify (高パフォーマンス)
リアルタイム: Socket.io
データベース: 
  - PostgreSQL (メインDB)
  - Redis (キャッシュ・セッション)
ORM: Prisma (型安全)
```

### AI/ML統合

```yaml
音声認識:
  基本: Web Speech API
  高精度: Whisper API
  
画像認識・処理:
  軽量タスク: GPT-5 nano ($0.05/1M)
  複雑タスク: Gemini 2.5 Flash-Lite ($0.10/1M)
  推論必要時: Gemini 2.5 Flash with thinking
```

## 決定事項

### 採用技術スタック

```typescript
// フロントエンド
{
  "3Dフレームワーク": "Babylon.js 8.0",
  "UIフレームワーク": "React 18 + TypeScript",
  "状態管理": "Zustand",
  "ビルドツール": "Vite"
}

// バックエンド  
{
  "ランタイム": "Node.js 20 LTS",
  "APIフレームワーク": "Fastify",
  "WebSocket": "Socket.io",
  "DB": "PostgreSQL + Prisma",
  "キャッシュ": "Redis"
}
```

### 選定理由

1. **Babylon.js選定**
   - Quest 3の最新機能フル対応
   - 公式ドキュメント充実
   - WebXR実装が簡単（組み込み済み）
   - 活発なコミュニティ

2. **A-Frame不採用理由**
   - Quest 3コントローラー問題報告あり
   - 複雑なインタラクション実装に制限

3. **Three.js/R3F検討結果**
   - 学習コストが高い
   - パフォーマンス最適化が必要
   - ただし将来的な移行オプションとして残す