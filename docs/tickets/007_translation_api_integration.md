# チケット007: 翻訳API連携実装

## 概要
GPT-5 nanoとGemini 2.5 Flash-Liteを使用したリアルタイム翻訳機能を実装し、コストと品質のバランスを最適化する

## 優先度
**Critical** - 翻訳機能のコア

## 推定時間
4時間

## 前提条件
- チケット006完了（音声入力実装）
- OpenAI APIキー取得済み
- Google AI APIキー取得済み
- Node.jsバックエンド環境

## 作業内容

### 1. 翻訳サービス抽象化
```typescript
// src/services/translation/TranslationService.ts
export interface TranslationRequest {
  text: string;
  sourceLang: string;
  targetLang: string;
  context?: string;
  quality?: 'fast' | 'balanced' | 'high';
}

export interface TranslationResponse {
  translatedText: string;
  sourceLang: string;
  targetLang: string;
  confidence: number;
  provider: string;
  latency: number;
  cost: number;
}

export abstract class TranslationService {
  protected apiKey: string;
  protected baseUrl: string;
  
  constructor(apiKey: string, baseUrl: string) {
    this.apiKey = apiKey;
    this.baseUrl = baseUrl;
  }
  
  abstract translate(request: TranslationRequest): Promise<TranslationResponse>;
  abstract estimateCost(textLength: number): number;
  abstract getLatencyEstimate(): number;
  abstract getSupportedLanguages(): string[];
}
```

### 2. GPT-5 nano実装
```typescript
// src/services/translation/GPT5NanoTranslator.ts
import { TranslationService, TranslationRequest, TranslationResponse } from './TranslationService';

export class GPT5NanoTranslator extends TranslationService {
  private readonly MODEL = 'gpt-5-nano-2025-08-07';
  private readonly INPUT_COST = 0.05 / 1_000_000; // $0.05 per 1M tokens
  private readonly OUTPUT_COST = 0.40 / 1_000_000; // $0.40 per 1M tokens
  
  constructor(apiKey: string) {
    super(apiKey, 'https://api.openai.com/v1');
  }
  
  async translate(request: TranslationRequest): Promise<TranslationResponse> {
    const startTime = Date.now();
    
    const systemPrompt = this.buildSystemPrompt(request);
    const userPrompt = this.buildUserPrompt(request);
    
    try {
      const response = await fetch(`${this.baseUrl}/chat/completions`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${this.apiKey}`
        },
        body: JSON.stringify({
          model: this.MODEL,
          messages: [
            { role: 'system', content: systemPrompt },
            { role: 'user', content: userPrompt }
          ],
          max_completion_tokens: 500,
          verbosity: 'brief', // GPT-5 nano特有のパラメータ
          temperature: 0.3, // 翻訳の一貫性を重視
          stream: false
        })
      });
      
      if (!response.ok) {
        throw new Error(`API error: ${response.status}`);
      }
      
      const data = await response.json();
      const translatedText = data.choices[0].message.content.trim();
      const latency = Date.now() - startTime;
      
      // コスト計算
      const inputTokens = data.usage.prompt_tokens;
      const outputTokens = data.usage.completion_tokens;
      const cost = (inputTokens * this.INPUT_COST) + (outputTokens * this.OUTPUT_COST);
      
      return {
        translatedText,
        sourceLang: request.sourceLang,
        targetLang: request.targetLang,
        confidence: 0.95, // GPT-5 nanoは高精度
        provider: 'gpt-5-nano',
        latency,
        cost
      };
      
    } catch (error) {
      console.error('GPT-5 nano translation error:', error);
      throw new Error('Translation failed');
    }
  }
  
  private buildSystemPrompt(request: TranslationRequest): string {
    return `You are a professional translator. 
Translate from ${request.sourceLang} to ${request.targetLang}.
Maintain the tone and context of the original text.
Provide only the translation without explanations.`;
  }
  
  private buildUserPrompt(request: TranslationRequest): string {
    let prompt = `Translate: "${request.text}"`;
    
    if (request.context) {
      prompt += `\n\nContext: ${request.context}`;
    }
    
    return prompt;
  }
  
  estimateCost(textLength: number): number {
    // 簡単なトークン数推定 (1文字 ≈ 0.5トークン)
    const estimatedTokens = textLength * 0.5;
    return estimatedTokens * (this.INPUT_COST + this.OUTPUT_COST);
  }
  
  getLatencyEstimate(): number {
    return 500; // GPT-5 nanoは超低遅延 (500ms)
  }
  
  getSupportedLanguages(): string[] {
    return [
      'ja-JP', 'en-US', 'zh-CN', 'ko-KR', 'es-ES', 'fr-FR',
      'de-DE', 'it-IT', 'pt-BR', 'ru-RU', 'ar-SA', 'hi-IN'
    ];
  }
}
```

### 3. Gemini 2.5 Flash-Lite実装
```typescript
// src/services/translation/GeminiTranslator.ts
import { TranslationService, TranslationRequest, TranslationResponse } from './TranslationService';

export class GeminiTranslator extends TranslationService {
  private readonly MODEL = 'gemini-2.5-flash-lite';
  private readonly INPUT_COST = 0.10 / 1_000_000; // $0.10 per 1M tokens
  private readonly OUTPUT_COST = 0.30 / 1_000_000; // $0.30 per 1M tokens
  
  constructor(apiKey: string) {
    super(apiKey, 'https://generativelanguage.googleapis.com/v1beta');
  }
  
  async translate(request: TranslationRequest): Promise<TranslationResponse> {
    const startTime = Date.now();
    
    try {
      const response = await fetch(
        `${this.baseUrl}/models/${this.MODEL}:generateContent?key=${this.apiKey}`,
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            contents: [{
              parts: [{
                text: this.buildPrompt(request)
              }]
            }],
            generationConfig: {
              temperature: 0.3,
              maxOutputTokens: 500,
              topP: 0.8,
              topK: 10
            },
            safetySettings: [
              {
                category: 'HARM_CATEGORY_HATE_SPEECH',
                threshold: 'BLOCK_ONLY_HIGH'
              }
            ]
          })
        }
      );
      
      if (!response.ok) {
        throw new Error(`API error: ${response.status}`);
      }
      
      const data = await response.json();
      const translatedText = data.candidates[0].content.parts[0].text.trim();
      const latency = Date.now() - startTime;
      
      // 簡単なコスト推定
      const estimatedTokens = (request.text.length + translatedText.length) * 0.5;
      const cost = estimatedTokens * (this.INPUT_COST + this.OUTPUT_COST);
      
      return {
        translatedText,
        sourceLang: request.sourceLang,
        targetLang: request.targetLang,
        confidence: 0.9,
        provider: 'gemini-2.5-flash-lite',
        latency,
        cost
      };
      
    } catch (error) {
      console.error('Gemini translation error:', error);
      throw new Error('Translation failed');
    }
  }
  
  private buildPrompt(request: TranslationRequest): string {
    let prompt = `Translate the following text from ${request.sourceLang} to ${request.targetLang}.\n`;
    prompt += `Only provide the translation, no explanations.\n\n`;
    prompt += `Text: "${request.text}"`;
    
    if (request.context) {
      prompt += `\n\nContext: ${request.context}`;
    }
    
    return prompt;
  }
  
  estimateCost(textLength: number): number {
    const estimatedTokens = textLength * 0.5;
    return estimatedTokens * (this.INPUT_COST + this.OUTPUT_COST);
  }
  
  getLatencyEstimate(): number {
    return 800; // Gemini Flash-Lite: 800ms
  }
  
  getSupportedLanguages(): string[] {
    return [
      'ja-JP', 'en-US', 'zh-CN', 'ko-KR', 'es-ES', 'fr-FR',
      'de-DE', 'it-IT', 'pt-BR', 'ru-RU', 'ar-SA', 'hi-IN',
      'th-TH', 'vi-VN', 'id-ID', 'ms-MY'
    ];
  }
}
```

### 4. 翻訳オーケストレータ
```typescript
// src/services/translation/TranslationOrchestrator.ts
import { GPT5NanoTranslator } from './GPT5NanoTranslator';
import { GeminiTranslator } from './GeminiTranslator';
import { TranslationRequest, TranslationResponse } from './TranslationService';

export class TranslationOrchestrator {
  private gpt5Nano: GPT5NanoTranslator;
  private gemini: GeminiTranslator;
  private cache: Map<string, TranslationResponse>;
  private readonly CACHE_TTL = 3600000; // 1時間
  
  constructor(openaiKey: string, googleKey: string) {
    this.gpt5Nano = new GPT5NanoTranslator(openaiKey);
    this.gemini = new GeminiTranslator(googleKey);
    this.cache = new Map();
  }
  
  async translate(request: TranslationRequest): Promise<TranslationResponse> {
    // キャッシュチェック
    const cacheKey = this.getCacheKey(request);
    const cached = this.cache.get(cacheKey);
    
    if (cached && this.isCacheValid(cached)) {
      return { ...cached, latency: 0 };
    }
    
    // プロバイダー選択
    const provider = this.selectProvider(request);
    
    try {
      let response: TranslationResponse;
      
      if (provider === 'gpt5') {
        response = await this.gpt5Nano.translate(request);
      } else {
        response = await this.gemini.translate(request);
      }
      
      // キャッシュ保存
      this.cache.set(cacheKey, {
        ...response,
        timestamp: Date.now()
      });
      
      return response;
      
    } catch (error) {
      // フォールバック
      console.error(`Primary provider failed, trying fallback...`);
      
      if (provider === 'gpt5') {
        return await this.gemini.translate(request);
      } else {
        return await this.gpt5Nano.translate(request);
      }
    }
  }
  
  private selectProvider(request: TranslationRequest): 'gpt5' | 'gemini' {
    // 品質要求に基づいて選択
    if (request.quality === 'high') {
      return 'gpt5'; // GPT-5 nanoは高精度
    }
    
    if (request.quality === 'fast') {
      return 'gpt5'; // GPT-5 nanoは超低遅延
    }
    
    // コスト優先の場合
    const textLength = request.text.length;
    const gpt5Cost = this.gpt5Nano.estimateCost(textLength);
    const geminiCost = this.gemini.estimateCost(textLength);
    
    // コストが低い方を選択
    return gpt5Cost <= geminiCost ? 'gpt5' : 'gemini';
  }
  
  private getCacheKey(request: TranslationRequest): string {
    return `${request.sourceLang}-${request.targetLang}-${request.text}`;
  }
  
  private isCacheValid(cached: any): boolean {
    return cached.timestamp && (Date.now() - cached.timestamp) < this.CACHE_TTL;
  }
  
  // バッチ翻訳（複数テキストをまとめて処理）
  async translateBatch(
    requests: TranslationRequest[]
  ): Promise<TranslationResponse[]> {
    return Promise.all(requests.map(req => this.translate(req)));
  }
  
  // コスト統計取得
  getCostStatistics(): {
    totalCost: number;
    requestCount: number;
    averageCost: number;
  } {
    let totalCost = 0;
    let requestCount = 0;
    
    this.cache.forEach(response => {
      totalCost += response.cost;
      requestCount++;
    });
    
    return {
      totalCost,
      requestCount,
      averageCost: requestCount > 0 ? totalCost / requestCount : 0
    };
  }
}
```

### 5. Socket.ioリアルタイム通信
```typescript
// src/services/RealtimeTranslationService.ts
import { io, Socket } from 'socket.io-client';
import { TranslationRequest, TranslationResponse } from './translation/TranslationService';

export class RealtimeTranslationService {
  private socket: Socket | null = null;
  private serverUrl: string;
  private onTranslation?: (response: TranslationResponse) => void;
  private onError?: (error: Error) => void;
  private onConnect?: () => void;
  private onDisconnect?: () => void;
  
  constructor(serverUrl: string = 'http://localhost:3001') {
    this.serverUrl = serverUrl;
  }
  
  connect(): void {
    this.socket = io(this.serverUrl, {
      transports: ['websocket'],
      reconnection: true,
      reconnectionAttempts: 5,
      reconnectionDelay: 1000
    });
    
    this.setupEventListeners();
  }
  
  private setupEventListeners(): void {
    if (!this.socket) return;
    
    // 接続成功
    this.socket.on('connect', () => {
      console.log('Connected to translation server');
      this.onConnect?.();
    });
    
    // 接続切断
    this.socket.on('disconnect', () => {
      console.log('Disconnected from translation server');
      this.onDisconnect?.();
    });
    
    // 翻訳結果受信
    this.socket.on('translation:result', (response: TranslationResponse) => {
      this.onTranslation?.(response);
    });
    
    // エラー受信
    this.socket.on('translation:error', (error: any) => {
      this.onError?.(new Error(error.message || 'Translation error'));
    });
    
    // プログレス更新（ストリーミング対応）
    this.socket.on('translation:progress', (data: {
      partial: string;
      progress: number;
    }) => {
      console.log(`Translation progress: ${data.progress}%`);
    });
  }
  
  // 翻訳リクエスト送信
  sendTranslationRequest(request: TranslationRequest): void {
    if (!this.socket || !this.socket.connected) {
      throw new Error('Not connected to translation server');
    }
    
    this.socket.emit('translation:request', request);
  }
  
  // ストリーミング翻訳開始
  startStreamingTranslation(
    sourceLang: string,
    targetLang: string
  ): void {
    if (!this.socket) return;
    
    this.socket.emit('translation:stream:start', {
      sourceLang,
      targetLang
    });
  }
  
  // ストリーミングテキスト送信
  sendStreamingText(text: string): void {
    if (!this.socket) return;
    
    this.socket.emit('translation:stream:text', { text });
  }
  
  // ストリーミング翻訳終了
  endStreamingTranslation(): void {
    if (!this.socket) return;
    
    this.socket.emit('translation:stream:end');
  }
  
  // コールバック設定
  onTranslationResult(callback: (response: TranslationResponse) => void): void {
    this.onTranslation = callback;
  }
  
  onErrorHandler(callback: (error: Error) => void): void {
    this.onError = callback;
  }
  
  onConnectionChange(
    onConnect: () => void,
    onDisconnect: () => void
  ): void {
    this.onConnect = onConnect;
    this.onDisconnect = onDisconnect;
  }
  
  // 接続状態確認
  isConnected(): boolean {
    return this.socket?.connected || false;
  }
  
  // 切断
  disconnect(): void {
    this.socket?.disconnect();
    this.socket = null;
  }
}
```

## 技術詳細

### GPT-5 nanoの特徴（2025年現在）
- **価格**: $0.05/1M入力トークン、$0.40/1M出力トークン
- **遅延**: 超低遅延（500ms以下）
- **精度**: 高精度な多言語翻訳
- **特有機能**: reasoning_effort、verbosityパラメータ

### Gemini 2.5 Flash-Liteの特徴
- **価格**: $0.10/1M入力トークン、$0.30/1M出力トークン
- **遅延**: 低遅延（800ms程度）
- **特徴**: アジア言語に強い

### Socket.ioの利点
- **自動再接続**: ネットワーク障害時の回復
- **フォールバック**: WebSocket不可時はHTTPポーリング
- **イベントベース**: 直感的なAPI

## 完了条件
- [ ] GPT-5 nano APIが動作する
- [ ] Gemini 2.5 Flash-Lite APIが動作する
- [ ] 自動プロバイダー選択が動作する
- [ ] キャッシュ機能が動作する
- [ ] Socket.ioリアルタイム通信が動作する
- [ ] 日本語→英語翻訳が正確
- [ ] 英語→日本語翻訳が正確
- [ ] コスト計算が正確

## トラブルシューティング

### 問題: APIキーが無効
- 解決: 環境変数を確認
- OpenAI Platformでキーを再生成

### 問題: レートリミットエラー
- 解決: リトライロジック実装
- キャッシュを活用

### 問題: 翻訳精度が低い
- 解決: temperatureを下げる（0.3以下）
- コンテキスト情報を追加

## 参考資料
- [OpenAI GPT-5 APIドキュメント](https://platform.openai.com/docs/models/gpt-5)
- [Google AI Studio](https://ai.google.dev/)
- [Socket.ioドキュメント](https://socket.io/docs/v4/)

## 次のステップ
→ 008_ar_subtitle_display.md