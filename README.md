# AITuberTest

講演「AI時代の教育」冒頭デモ用ローカル AIVTuber システム。
マイクで話しかけると、AIアバターが教育×AIテーマで音声応答する。

## 構成

- **フロントエンド**: AITuberKit（git submodule）
- **LLM**: OpenAI API（クラウド）
- **TTS**: VOICEVOX（ローカル）
- **STT**: Web Speech API

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

### 3. 環境変数を設定

```bash
cp configs/aituberkit.env aituberkit/.env
```

`.env` を開き、APIキーを記入する：

```env
OPENAI_API_KEY="sk-..."
```

### 4. VOICEVOX を起動

VOICEVOX アプリを起動する（localhost:50021 で待受）。

### 5. AITuberKit を起動

```bash
cd aituberkit
pnpm dev
```

ブラウザで http://localhost:3000 を開く。

## 当日の起動手順

1. VOICEVOX アプリを起動
2. `cd aituberkit && pnpm dev`
3. ブラウザで http://localhost:3000 を開く
4. マイクアクセスを許可
5. 音声入力ボタンを押して話しかける

## 注意事項

- LLM（OpenAI）は会場のネット接続が必要
- `aituberkit/.env` には APIキーを記入するため **git にコミットしない**
- `configs/aituberkit.env` は設定のテンプレート（APIキーは空欄）
