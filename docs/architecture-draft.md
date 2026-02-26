# アーキテクチャ設計草案

## 基本方針

**AITuberKit を最大限活用し、不足する部分のみ独自実装する。**

AITuberKit は VRM アバター・VOICEVOX・Ollama・音声認識をすでに統合しており、
本プロジェクトの大部分を追加実装なしにカバーできる。

---

## 全体構成

```
[講演者のマイク]
      ↓ 音声（Web Speech API または Whisper ※要検証）
      ↓
┌─────────────────────────────────────────┐
│           AITuberKit (Next.js)          │
│  - 音声入力 / テキスト入力              │
│  - LLM 呼び出し（Ollama）               │
│  - VOICEVOX 音声合成                    │
│  - VRM アバター表示 / リップシンク       │
│  - 状態表示 UI                          │
└─────────────┬───────────────────────────┘
              │ HTTP
      ┌───────┴───────┐
      ↓               ↓
[Ollama :11434]  [VOICEVOX :50021]
 ローカル LLM     ローカル TTS
      ↑
 RAG ミドルウェア（独自実装・要否は検証次第）
      ↑
 教育文書インデックス（ローカル）
```

---

## 主要コンポーネント

### AITuberKit が担う部分（既存機能をそのまま使う）

| 機能 | AITuberKit での対応 |
|---|---|
| VRM アバター表示 | Three.js + @pixiv/three-vrm で実装済み |
| リップシンク | 音声に合わせた口パク制御が実装済み |
| 表情・モーション制御 | Neutral / Happy 等の感情タグで制御可能 |
| VOICEVOX 連携 | `VOICEVOX_SERVER_URL` で設定するだけで動作 |
| Ollama 連携 | LLM プロバイダーとして Ollama を選択可能 |
| 音声入力 | ブラウザの Web Speech API（**オフライン可否は要検証**） |
| 状態表示 UI | 「考え中」等の表示が標準搭載 |

### 独自追加が必要な部分

| 機能 | 理由 |
|---|---|
| RAG（文書検索）| AITuberKit 単体では教育文書の参照機能がない |
| システムプロンプト設計 | AI時代の教育テーマに絞った人格・制約の定義 |
| （音声入力が offline 不可の場合）ローカル STT | Web Speech API がオフラインで動かない場合のみ |

---

## RAG の組み込み方針（3 つの選択肢）

RAG をどこに組み込むかは、複雑さとメンテナビリティのトレードオフになる。
**Phase 0 の技術検証後に選択する。**

### 選択肢 A: システムプロンプトへの直接埋め込み（最もシンプル）

教育文書のポイントをシステムプロンプトに直接記述する。
文書量が少ない（数千〜1万トークン以内）場合は RAG 不要。

```
pros: 追加実装ゼロ、AITuberKit の設定だけで完結
cons: 文書が多い場合はコンテキスト長の限界に達する
```

### 選択肢 B: Ollama の前段に RAG プロキシを挟む（中程度の実装）

AITuberKit → (RAG プロキシ) → Ollama という構成。
AITuberKit の Ollama エンドポイントをプロキシの URL に変更するだけで動作する。

```
pros: AITuberKit 本体を改変しない、文書量が多くても対応可能
cons: プロキシの実装が必要（Node.js または Python、数百行程度）
```

### 選択肢 C: AITuberKit に直接統合（最も高機能、最も複雑）

AITuberKit のソースを fork して RAG ロジックを組み込む。

```
pros: 完全な制御が可能
cons: AITuberKit の更新を追いかけるコストが高い。MVP には過剰
```

**推奨: まず選択肢 A で試す。文書量が多ければ選択肢 B へ移行。**

---

## コンポーネント起動構成

本番当日に起動が必要なプロセス：

| プロセス | 起動方法 | ポート |
|---|---|---|
| VOICEVOX | アプリ起動（GUIあり） | 50021 |
| Ollama | `ollama serve` | 11434 |
| AITuberKit | `npm run dev` or Electron | 3000 |
| RAG プロキシ（選択肢 B の場合のみ） | `node proxy.js` 等 | 任意 |

起動手順を1つのスクリプトにまとめることで、当日の操作を最小化できる。

---

## AITuberKit の設定方針

`.env` ファイルで以下を設定する（UI 設定より環境変数を優先するオプションを有効にする）：

```env
# LLM
NEXT_PUBLIC_SELECT_AI_SERVICE=ollama
NEXT_PUBLIC_OLLAMA_URL=http://localhost:11434
NEXT_PUBLIC_OLLAMA_MODEL=（選定したモデル）

# TTS
NEXT_PUBLIC_VOICE_LANGUAGE=ja-JP
NEXT_PUBLIC_SELECT_VOICE=voicevox
VOICEVOX_SERVER_URL=http://localhost:50021

# キャラクター
NEXT_PUBLIC_SYSTEM_PROMPT=（AI時代の教育専門家としてのプロンプト）

# UI 簡素化（配信者向け機能を非表示）
NEXT_PUBLIC_SHOW_PRESET_QUESTIONS=false
```

---

## Mac 環境での注意点

| 項目 | 内容 |
|---|---|
| Ollama GPU 利用 | Apple Silicon (M1/M2/M3) では Metal 経由で GPU 推論が動く。CPU より大幅に速い |
| Whisper（選択肢 B の場合） | macOS 版 Whisper.cpp は Apple Silicon に最適化済み |
| マイクアクセス | システム設定 → プライバシー → マイク でブラウザに許可が必要 |
| VOICEVOX | macOS 版は GUI アプリとして提供されている |
| Node.js | `.tool-versions` に `24.9.0` 指定済み。asdf で管理 |
| ポート競合 | VOICEVOX: 50021 / AITuberKit: 3000 / Ollama: 11434 が競合しないよう注意 |

---

## どこまで既存機能、どこから独自実装か（まとめ）

```
AITuberKit そのまま使う
  ├── VRM アバター表示・リップシンク・モーション
  ├── VOICEVOX TTS 連携
  ├── Ollama LLM 連携
  ├── 音声入力（Web Speech API）
  └── 基本 UI

設定・カスタマイズで対応（実装不要）
  ├── システムプロンプト（教育テーマへの絞り込み）
  └── .env による UI 簡素化

独自実装（MVP では最小限）
  ├── RAG（選択肢 A なら設定のみ、選択肢 B ならプロキシ実装）
  └── ローカル STT（Web Speech API がオフライン不可の場合のみ）
```
