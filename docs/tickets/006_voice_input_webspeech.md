# チケット006: Web Speech APIを使用した音声入力実装

## 概要
Web Speech APIを使用してリアルタイム音声認識機能を実装し、日本語と英語の音声入力をサポートする

## 優先度
**High** - 翻訳機能の入力インターフェース

## 推定時間
3時間

## 前提条件
- チケット004完了（WebXR初期化）
- HTTPS環境（Web Speech APIはHTTPS必須）
- Chrome/Edgeブラウザ（WebSpeech APIサポート）

## 作業内容

### 1. 音声認識サービスクラス
```typescript
// src/services/SpeechRecognitionService.ts
export interface SpeechRecognitionConfig {
  language: string;
  continuous: boolean;
  interimResults: boolean;
  maxAlternatives: number;
}

export interface TranscriptResult {
  transcript: string;
  confidence: number;
  isFinal: boolean;
  timestamp: Date;
  language: string;
}

export class SpeechRecognitionService {
  private recognition: SpeechRecognition | null = null;
  private isListening: boolean = false;
  private currentLanguage: string = 'ja-JP';
  private onResult?: (result: TranscriptResult) => void;
  private onError?: (error: Error) => void;
  private onStart?: () => void;
  private onEnd?: () => void;
  
  constructor() {
    this.initializeSpeechRecognition();
  }
  
  private initializeSpeechRecognition(): void {
    // Web Speech APIのベンダープレフィックス対応
    const SpeechRecognition = 
      (window as any).SpeechRecognition || 
      (window as any).webkitSpeechRecognition;
    
    if (!SpeechRecognition) {
      console.error('Web Speech API is not supported');
      return;
    }
    
    this.recognition = new SpeechRecognition();
    
    // 基本設定
    this.recognition.continuous = true; // 継続的に音声を認識
    this.recognition.interimResults = true; // 中間結果も取得
    this.recognition.maxAlternatives = 1; // 候補数
    this.recognition.lang = this.currentLanguage;
    
    this.setupEventListeners();
  }
  
  private setupEventListeners(): void {
    if (!this.recognition) return;
    
    // 音声認識開始
    this.recognition.onstart = () => {
      console.log('Speech recognition started');
      this.isListening = true;
      this.onStart?.();
    };
    
    // 音声認識終了
    this.recognition.onend = () => {
      console.log('Speech recognition ended');
      this.isListening = false;
      this.onEnd?.();
      
      // 継続リスニングが有効な場合、自動再開
      if (this.isListening) {
        this.start();
      }
    };
    
    // 認識結果
    this.recognition.onresult = (event: SpeechRecognitionEvent) => {
      const results = event.results;
      const lastResult = results[results.length - 1];
      
      if (lastResult) {
        const transcript = lastResult[0].transcript;
        const confidence = lastResult[0].confidence || 0.9;
        const isFinal = lastResult.isFinal;
        
        const result: TranscriptResult = {
          transcript,
          confidence,
          isFinal,
          timestamp: new Date(),
          language: this.currentLanguage
        };
        
        this.onResult?.(result);
      }
    };
    
    // エラーハンドリング
    this.recognition.onerror = (event: SpeechRecognitionError) => {
      console.error('Speech recognition error:', event.error);
      
      let errorMessage = 'Speech recognition error';
      
      switch (event.error) {
        case 'not-allowed':
          errorMessage = 'Microphone permission denied';
          break;
        case 'no-speech':
          errorMessage = 'No speech detected';
          break;
        case 'network':
          errorMessage = 'Network error';
          break;
        case 'aborted':
          errorMessage = 'Recognition aborted';
          break;
      }
      
      this.onError?.(new Error(errorMessage));
      
      // 一部のエラーは自動再試行
      if (event.error === 'no-speech' && this.isListening) {
        setTimeout(() => this.start(), 1000);
      }
    };
    
    // 音声検出
    this.recognition.onsoundstart = () => {
      console.log('Sound detected');
    };
    
    this.recognition.onsoundend = () => {
      console.log('Sound ended');
    };
  }
  
  // 音声認識開始
  async start(): Promise<void> {
    if (!this.recognition) {
      throw new Error('Speech recognition not initialized');
    }
    
    try {
      // マイク権限をリクエスト
      await navigator.mediaDevices.getUserMedia({ audio: true });
      
      this.recognition.start();
      this.isListening = true;
    } catch (error) {
      console.error('Failed to start speech recognition:', error);
      throw error;
    }
  }
  
  // 音声認識停止
  stop(): void {
    if (this.recognition && this.isListening) {
      this.recognition.stop();
      this.isListening = false;
    }
  }
  
  // 言語切り替え
  setLanguage(language: string): void {
    this.currentLanguage = language;
    
    if (this.recognition) {
      this.recognition.lang = language;
      
      // リスニング中の場合は再起動
      if (this.isListening) {
        this.stop();
        setTimeout(() => this.start(), 100);
      }
    }
  }
  
  // コールバック設定
  onTranscript(callback: (result: TranscriptResult) => void): void {
    this.onResult = callback;
  }
  
  onErrorHandler(callback: (error: Error) => void): void {
    this.onError = callback;
  }
  
  onStartHandler(callback: () => void): void {
    this.onStart = callback;
  }
  
  onEndHandler(callback: () => void): void {
    this.onEnd = callback;
  }
  
  // 現在の状態
  getStatus(): {
    isListening: boolean;
    language: string;
    isSupported: boolean;
  } {
    return {
      isListening: this.isListening,
      language: this.currentLanguage,
      isSupported: !!this.recognition
    };
  }
  
  dispose(): void {
    this.stop();
    this.recognition = null;
  }
}
```

### 2. 言語検出サービス
```typescript
// src/services/LanguageDetectionService.ts
export class LanguageDetectionService {
  private readonly languagePatterns = {
    'ja-JP': [
      /[぀-ゟ]/, // Hiragana
      /[゠-ヿ]/, // Katakana
      /[一-龯]/, // Kanji
    ],
    'en-US': [
      /^[a-zA-Z\s]+$/ // English alphabet
    ],
    'zh-CN': [
      /[一-鿿]/ // Simplified Chinese
    ],
    'ko-KR': [
      /[가-힯]/, // Hangul
      /[ᄀ-ᇿ]/  // Hangul Jamo
    ]
  };
  
  detectLanguage(text: string): string {
    // テキストが空の場合はデフォルト言語を返す
    if (!text || text.trim().length === 0) {
      return 'ja-JP';
    }
    
    // 各言語のパターンをチェック
    for (const [lang, patterns] of Object.entries(this.languagePatterns)) {
      for (const pattern of patterns) {
        if (pattern.test(text)) {
          return lang;
        }
      }
    }
    
    // マッチしない場合は英語として扱う
    return 'en-US';
  }
  
  // 言語名を取得
  getLanguageName(code: string): string {
    const languageNames: Record<string, string> = {
      'ja-JP': '日本語',
      'en-US': 'English',
      'zh-CN': '中文',
      'ko-KR': '한국어'
    };
    
    return languageNames[code] || code;
  }
}
```

### 3. Reactコンポーネント
```typescript
// src/components/VoiceInput.tsx
import React, { useState, useEffect, useRef } from 'react';
import { SpeechRecognitionService, TranscriptResult } from '@services/SpeechRecognitionService';
import { LanguageDetectionService } from '@services/LanguageDetectionService';

interface VoiceInputProps {
  onTranscript: (text: string, language: string, isFinal: boolean) => void;
  autoDetectLanguage?: boolean;
}

const VoiceInput: React.FC<VoiceInputProps> = ({ 
  onTranscript, 
  autoDetectLanguage = true 
}) => {
  const [isListening, setIsListening] = useState(false);
  const [currentLanguage, setCurrentLanguage] = useState('ja-JP');
  const [transcript, setTranscript] = useState('');
  const [error, setError] = useState<string | null>(null);
  const [volume, setVolume] = useState(0);
  
  const speechService = useRef<SpeechRecognitionService | null>(null);
  const languageService = useRef<LanguageDetectionService | null>(null);
  const audioContext = useRef<AudioContext | null>(null);
  const analyser = useRef<AnalyserNode | null>(null);
  
  useEffect(() => {
    // サービス初期化
    speechService.current = new SpeechRecognitionService();
    languageService.current = new LanguageDetectionService();
    
    // コールバック設定
    speechService.current.onTranscript((result: TranscriptResult) => {
      setTranscript(result.transcript);
      
      // 自動言語検出
      if (autoDetectLanguage && result.isFinal) {
        const detectedLang = languageService.current!.detectLanguage(
          result.transcript
        );
        
        if (detectedLang !== currentLanguage) {
          setCurrentLanguage(detectedLang);
          speechService.current!.setLanguage(detectedLang);
        }
      }
      
      onTranscript(result.transcript, result.language, result.isFinal);
    });
    
    speechService.current.onErrorHandler((error: Error) => {
      setError(error.message);
      setIsListening(false);
    });
    
    speechService.current.onStartHandler(() => {
      setIsListening(true);
      setError(null);
    });
    
    speechService.current.onEndHandler(() => {
      setIsListening(false);
    });
    
    // クリーンアップ
    return () => {
      speechService.current?.dispose();
      audioContext.current?.close();
    };
  }, [autoDetectLanguage, onTranscript]);
  
  // 音量レベル取得
  const setupAudioAnalyser = async () => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      audioContext.current = new AudioContext();
      analyser.current = audioContext.current.createAnalyser();
      
      const source = audioContext.current.createMediaStreamSource(stream);
      source.connect(analyser.current);
      
      analyser.current.fftSize = 256;
      const bufferLength = analyser.current.frequencyBinCount;
      const dataArray = new Uint8Array(bufferLength);
      
      const updateVolume = () => {
        if (!analyser.current) return;
        
        analyser.current.getByteFrequencyData(dataArray);
        const average = dataArray.reduce((a, b) => a + b) / bufferLength;
        setVolume(average / 255);
        
        if (isListening) {
          requestAnimationFrame(updateVolume);
        }
      };
      
      updateVolume();
    } catch (error) {
      console.error('Failed to setup audio analyser:', error);
    }
  };
  
  const handleStart = async () => {
    try {
      await speechService.current?.start();
      await setupAudioAnalyser();
    } catch (error) {
      setError('Failed to start voice input');
    }
  };
  
  const handleStop = () => {
    speechService.current?.stop();
  };
  
  const handleLanguageChange = (lang: string) => {
    setCurrentLanguage(lang);
    speechService.current?.setLanguage(lang);
  };
  
  return (
    <div style={{
      padding: 20,
      backgroundColor: 'rgba(0, 0, 0, 0.8)',
      borderRadius: 10,
      color: 'white'
    }}>
      <h3>Voice Input</h3>
      
      {/* 言語選択 */}
      <div style={{ marginBottom: 15 }}>
        <select
          value={currentLanguage}
          onChange={(e) => handleLanguageChange(e.target.value)}
          style={{
            padding: 5,
            borderRadius: 5,
            backgroundColor: '#333',
            color: 'white',
            border: '1px solid #555'
          }}
        >
          <option value="ja-JP">日本語</option>
          <option value="en-US">English</option>
          <option value="zh-CN">中文</option>
          <option value="ko-KR">한국어</option>
        </select>
      </div>
      
      {/* 録音ボタン */}
      <button
        onClick={isListening ? handleStop : handleStart}
        style={{
          padding: '10px 20px',
          backgroundColor: isListening ? '#f44336' : '#4CAF50',
          color: 'white',
          border: 'none',
          borderRadius: 5,
          cursor: 'pointer',
          marginBottom: 15
        }}
      >
        {isListening ? '■ Stop' : '● Record'}
      </button>
      
      {/* 音量インジケータ */}
      {isListening && (
        <div style={{ marginBottom: 15 }}>
          <div style={{
            height: 10,
            backgroundColor: '#333',
            borderRadius: 5,
            overflow: 'hidden'
          }}>
            <div style={{
              height: '100%',
              width: `${volume * 100}%`,
              backgroundColor: '#4CAF50',
              transition: 'width 0.1s'
            }} />
          </div>
        </div>
      )}
      
      {/* トランスクリプト表示 */}
      {transcript && (
        <div style={{
          padding: 10,
          backgroundColor: '#333',
          borderRadius: 5,
          marginBottom: 10,
          minHeight: 50
        }}>
          {transcript}
        </div>
      )}
      
      {/* エラー表示 */}
      {error && (
        <div style={{
          padding: 10,
          backgroundColor: '#d32f2f',
          borderRadius: 5,
          marginTop: 10
        }}>
          {error}
        </div>
      )}
    </div>
  );
};

export default VoiceInput;
```

## 技術詳細

### Web Speech APIの特徴（2025年現在）
- **ブラウザサポート**: Chrome、Edgeで完全サポート
- **リアルタイム処理**: 中間結果と最終結果の両方を取得可能
- **多言語対応**: 50以上の言語をサポート
- **クラウドベース**: Googleの音声認識エンジンを使用

### 日本語認識の最適化
- 句読点の自動挿入
- 漢字変換の精度向上
- 方言対応（一部）

### パフォーマンス考慮
- ネットワーク遅延: 100-300ms
- CPU使用率: 低（クラウド処理）
- バッテリー消費: 中程度

## 完了条件
- [ ] Web Speech APIが初期化される
- [ ] マイク権限が取得できる
- [ ] 音声入力が開始/停止できる
- [ ] リアルタイムで文字起こしされる
- [ ] 日本語と英語が認識される
- [ ] 言語自動検出が動作する
- [ ] エラーハンドリングが実装されている

## トラブルシューティング

### 問題: マイク権限が拒否される
- 解決: HTTPS環境でアクセス
- ブラウザ設定でマイク権限を確認

### 問題: 音声が認識されない
- 解決: マイクの入力レベルを確認
- ノイズキャンセリングを有効化

### 問題: 日本語の認識精度が低い
- 解決: lang属性を"ja-JP"に明示的に設定
- 発話速度をゆっくりに

## 参考資料
- [Web Speech API仕様](https://wicg.github.io/speech-api/)
- [MDN Web Speech API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Speech_API)
- [Chrome Web Speech APIドキュメント](https://developer.chrome.com/blog/voice-driven-web-apps-introduction-to-the-web-speech-api/)

## 次のステップ
→ 007_translation_api_integration.md