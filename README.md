# AITuberTest

講演「AI時代の教育」冒頭デモ用 AIVTuber システム。
マイクで話しかけると、AIアバターが教育×AIテーマで音声応答する。

## 構成

- **フロントエンド**: AITuberKit（git submodule）
- **LLM**: OpenAI API（Realtime API モード）
- **TTS**: VOICEVOX（ローカル）
- **STT**: OpenAI Realtime API
- **検索**: Google 検索グラウンディング（リアルタイム情報参照）
- **記憶**: セッション内の会話履歴保持（MAX 30件）
- **聴衆Q&A**: スマホから質問フォーム経由で送信

## セットアップ

### 1. リポジトリを clone

```bash
git clone --recurse-submodules <your-repo-url>
cd AITuberTest
```

> すでに clone 済みで submodule が空の場合：
> ```bash
> git submodule update --init
> ```

### 2. AITuberKit の依存パッケージをインストール

```bash
cd aituberkit
pnpm install
```

### 3. 聴衆向け質問フォームを配置

```bash
cp configs/question.html aituberkit/public/question.html
```

> このファイルはサブモジュール内で未追跡のまま置くだけでよい（git push に影響しない）

### 4. 環境変数を設定

```bash
cp configs/aituberkit.env aituberkit/.env
```

`.env` を開き、APIキーを記入する：

```env
OPENAI_API_KEY="sk-..."
NEXT_PUBLIC_OPENAI_API_KEY="sk-..."   # Realtime API 用
```

### 5. VOICEVOX を起動

VOICEVOX アプリを起動する（localhost:50021 で待受）。

### 6. AITuberKit を起動

```bash
cd aituberkit
pnpm dev
```

ブラウザで http://localhost:3000 を開く。

---

## 当日の起動手順

1. VOICEVOX アプリを起動
2. `cd aituberkit && pnpm dev`
3. ブラウザで http://localhost:3000 を開く
4. マイクアクセスを許可

### 聴衆Q&A を使う場合

5. 現在の IP を確認する：
   ```bash
   ipconfig getifaddr en0
   ```
6. ngrok でインターネット経由の URL を発行する：
   ```bash
   ngrok http 3000
   ```
7. 表示された `https://xxxx.ngrok-free.app/question.html` を QR コード化してスライドに表示

---

## 実装済み機能

| 機能 | 概要 |
|---|---|
| 音声対話 | OpenAI Realtime API による低レイテンシ会話 |
| 検索グラウンディング | Google 検索で最新情報を引用した回答 |
| 会話記憶 | セッション内の前の発言を参照した応答 |
| 教育文書参照 | システムプロンプトに教育資料を直接埋め込み |
| 聴衆Q&A | QR コード → スマホフォーム → AI が回答 |

---

## 注意事項

- LLM（OpenAI）は会場のネット接続が必要
- `aituberkit/.env` には APIキーを記入するため **git にコミットしない**
- `configs/aituberkit.env` は設定のテンプレート（APIキーは空欄）
- 会場の WiFi が変わるたびに IP が変わるため ngrok を推奨
