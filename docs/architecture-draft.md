# アーキテクチャ設計草案

## 基本方針

**AITuberKit を最大限活用し、不足する部分のみ独自実装する。**

- LLM は Gemini または OpenAI（クラウド API）を使用する。会場はネット接続可能
- ローカル LLM（Ollama / LM Studio）は不要
- TTS は VOICEVOX（ローカル）で完結
- アバターは VRM 形式

---

## 全体構成

```
[講演者のマイク]
      ↓ 音声（Web Speech API ※オフライン可否は要検証）
      ↓
┌─────────────────────────────────────────┐
│           AITuberKit (Next.js)          │
│  - 音声入力（Web Speech API）           │
│  - LLM 呼び出し（Gemini / OpenAI）      │
│  - VOICEVOX 音声合成                    │
│  - VRM アバター表示 / リップシンク       │
│  - 状態表示 UI                          │
└─────────┬───────────────┬───────────────┘
          │ HTTPS         │ HTTP
          ↓               ↓
[Gemini / OpenAI API]  [VOICEVOX :50021]
  クラウド LLM           ローカル TTS
```

**RAG について**: 教育文書の量が少なければシステムプロンプトへの直接埋め込みで対応。
文書量が多い場合は RAG ミドルウェアを追加する（要判断）。

---

## 主要コンポーネントと責務

### AITuberKit が担う部分（既存機能をそのまま使う）

| 機能 | AITuberKit での対応 |
|---|---|
| VRM アバター表示 | Three.js + @pixiv/three-vrm で実装済み |
| リップシンク | 音声に合わせた口パク制御が実装済み |
| 表情・モーション制御 | Neutral / Happy 等の感情タグで制御可能 |
| VOICEVOX 連携 | `VOICEVOX_SERVER_URL` で設定するだけで動作 |
| Gemini / OpenAI 連携 | LLM プロバイダーとして選択・設定可能 |
| 音声入力 | ブラウザの Web Speech API（**オフライン可否は要検証**） |
| 状態表示 UI | 「考え中」等の表示が標準搭載 |

### 独自追加が必要な部分

| 機能 | 理由 |
|---|---|
| システムプロンプト設計 | AI時代の教育テーマに絞った人格・制約の定義 |
| RAG（文書量が多い場合のみ） | 教育文書が多量の場合、プロンプトへの直接埋め込みでは対応できない |
| 音声入力代替（必要な場合のみ） | Web Speech API がオフラインで動かない場合のみ Whisper 等を検討 |

---

## RAG の組み込み方針

**まずシステムプロンプトへの直接埋め込みで試す。文書量次第で判断する。**

### 選択肢 A: システムプロンプトへの直接埋め込み（最もシンプル）

教育文書のポイントをシステムプロンプトに直接記述する。
Gemini 1.5 / 2.0 は 100 万トークン以上のコンテキストをサポートするため、
相当量の文書でも埋め込み可能。

```
pros: 追加実装ゼロ。AITuberKit の設定だけで完結
cons: 文書が非常に多い場合はコスト・レイテンシに影響
```

### 選択肢 B: RAG プロキシを挟む（文書量が膨大な場合）

AITuberKit → (RAG プロキシ) → Gemini/OpenAI という構成。
AITuberKit の LLM エンドポイントをプロキシの URL に変更するだけで動作する。

```
pros: AITuberKit 本体を改変しない。大量文書に対応可能
cons: プロキシの実装が必要（数百行程度）
```

**推奨: まず選択肢 A で試す。Gemini のコンテキスト長を活かす。**

---

## コンポーネント起動構成

本番当日に起動が必要なプロセス：

| プロセス | 起動方法 | ポート |
|---|---|---|
| VOICEVOX | アプリ起動（GUI あり） | 50021 |
| AITuberKit | `pnpm dev` | 3000 |
| RAG プロキシ（選択肢 B の場合のみ） | 別途スクリプト | 任意 |

起動手順を1つのスクリプトにまとめることで、当日の操作を最小化できる。

---

## AITuberKit の設定方針

`.env` ファイルで以下を設定する：

```env
# LLM（Gemini を使う場合）
NEXT_PUBLIC_SELECT_AI_SERVICE=gemini
NEXT_PUBLIC_GEMINI_KEY=（API キー）
NEXT_PUBLIC_GEMINI_MODEL=gemini-2.0-flash

# TTS
NEXT_PUBLIC_SELECT_VOICE=voicevox
VOICEVOX_SERVER_URL=http://localhost:50021

# キャラクター・プロンプト
NEXT_PUBLIC_SYSTEM_PROMPT=（AI時代の教育専門家としてのプロンプト）

# UI 簡素化
NEXT_PUBLIC_SHOW_PRESET_QUESTIONS=false
```

---

## Mac 環境での注意点

| 項目 | 内容 |
|---|---|
| マイクアクセス | システム設定 → プライバシー → マイク でブラウザに許可が必要 |
| VOICEVOX | macOS 版は GUI アプリとして提供されている |
| Node.js | `.tool-versions` に `24.9.0` 指定済み。asdf で管理 |
| パッケージ管理 | pnpm を使用（npm は使わない） |
| ポート競合 | VOICEVOX: 50021 / AITuberKit: 3000 が競合しないよう注意 |
| ネット接続 | LLM（Gemini / OpenAI）はクラウド API を使用するため会場のネット接続が必要 |

---

## どこまで既存機能、どこから独自実装か（まとめ）

```
AITuberKit そのまま使う
  ├── VRM アバター表示・リップシンク・モーション
  ├── VOICEVOX TTS 連携
  ├── Gemini / OpenAI LLM 連携
  ├── 音声入力（Web Speech API）
  └── 基本 UI

設定・カスタマイズで対応（実装不要）
  ├── システムプロンプト（教育テーマへの絞り込み）
  └── .env による UI 簡素化・LLM 設定

独自実装（MVP では最小限）
  ├── RAG（文書量が多い場合のみ、選択肢 B）
  └── ローカル STT（Web Speech API がオフライン不可の場合のみ）
```
