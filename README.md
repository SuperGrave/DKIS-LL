# DKIS-LINE

**DKIS-LINE** は、DKIS の会話・ツール実行ロジックを LINE Bot 向けに軽量化した Flask アプリです。

LINE の Webhook で受け取ったテキストメッセージを OpenAI に渡し、DKIS 形式の `[CMD] / [ARGS] / [ARGS-2] / [TEXT] / [NOTE]` を解釈して、検索・ニュース・天気・Webページ読み取りなどを行います。

本家 DKIS と違い、Web フロントエンド、SSE、VOICEVOX 音声合成、Spotify / YouTube 再生、ローカルファイル操作は含みません。

---

## 特徴

- LINE Bot Webhook 対応
- Flask による軽量HTTPサーバー
- OpenAI APIによるキャラクター会話
- DKIS互換のコマンド形式
- ユーザー別会話履歴
- 前回処理結果を含めた文脈入力
- RETRY連鎖による多段処理
- Google Custom Search APIによる検索
- Google News RSSによるニュース取得
- Open-Meteoによる天気取得
- `trafilatura` によるWebページ本文抽出
- `dist/settings.json` によるモデル・プロンプト・履歴長などの設定管理

---

## 本家DKISとの違い

| 項目 | DKIS | DKIS-LINE |
| --- | --- | --- |
| 実行環境 | ローカルWeb UI | LINE Bot |
| UI | HTML / SSE | LINEトーク画面 |
| 音声合成 | VOICEVOX対応 | なし |
| Spotify | 対応 | なし |
| YouTube | 対応 | なし |
| 現在地連携 | ブラウザGPSあり | なし |
| ローカルファイル操作 | あり | なし |
| 会話履歴 | サーバー内管理 | ユーザーIDごとに管理 |
| 主な用途 | ドライブ・ローカルAIコンパニオン | スマホから使える軽量DKIS Bot |

---

## 使えるコマンド

AI は以下のコマンドを選択できます。

| コマンド | 内容 |
| --- | --- |
| `SPEAK` | 通常会話 |
| `SAVE-LOG` | クラウド版では無効。案内メッセージを返す |
| `SEARCH` | Google Custom Search APIで検索 |
| `NEWS` | Google News RSSでニュース検索 |
| `WEATHER` | Open-Meteoで地名ベースの天気取得 |
| `READ-PAGE` | URL先のWebページ本文を抽出 |

`SEARCH`、`NEWS`、`WEATHER`、`READ-PAGE` は、取得した生結果を `RI` としてそのまま次のAI呼び出しに再投入します。  
本家DKISのような中間要約専用モデル呼び出しはありません。

---

## ディレクトリ構成

```text
DKIS-LL/
├── main.py                    # Flaskアプリ起動
├── pyproject.toml             # プロジェクト定義
├── README.md
├── dist/
│   └── settings.json          # モデル・プロンプト・履歴長などの設定
└── line_bot_app/
    ├── app.py                 # Flask app / LINE webhook
    ├── ai.py                  # 後方互換エイリアス
    ├── engine.py              # 会話履歴・コマンド実行・RETRY連鎖
    ├── commands.py            # SEARCH / NEWS / WEATHER / READ-PAGE等
    ├── config.py              # 環境変数とsettings.jsonの読み込み
    ├── settings_loader.py     # settings.json読み込み
    ├── parsing.py             # DKIS形式の応答パース
    ├── input_build.py         # AI入力構築
    ├── google_cse.py          # Google Custom Search
    ├── news_rss.py            # Google News RSS
    ├── scrape_page.py         # Webページ本文抽出
    ├── weather_openmeteo.py   # Open-Meteo天気取得
    └── line_messages.py       # LINE送信用テキスト分割
```

---

## 必要なもの

- Python 3.10 以上
- uv
- LINE Developers の Messaging API チャンネル
- OpenAI APIキー
- Google検索を使う場合は Google Programmable Search Engine

---

## Environment Variables

### 必須

| 変数 | 内容 |
| --- | --- |
| `LINE_CHANNEL_SECRET` | LINE Messaging API の Channel secret |
| `LINE_CHANNEL_ACCESS_TOKEN` | LINE Messaging API の Channel access token |
| `OPENAI_API_KEY` | OpenAI APIキー |

### 任意

| 変数 | 内容 |
| --- | --- |
| `DKIS_SETTINGS_PATH` | 設定ファイルパス。未指定時はリポジトリ内の `dist/settings.json` |
| `GOOGLE_API_KEY` | Google Custom Search APIキー。`SEARCH` 用 |
| `GOOGLE_CX` | Programmable Search Engine の検索エンジンID。`SEARCH` 用 |
| `PORT` | Flaskの待受ポート。未指定時は `5000` |

---

## セットアップ

### 1. リポジトリを取得

```bash
git clone https://github.com/SuperGrave/DKIS-LL.git
cd DKIS-LL
```

### 2. 依存関係をインストール

```bash
uv sync
```

### 3. 環境変数を設定

ローカル開発では `.env` を使えます。

```bash
cp .env.example .env
```

`.env` に以下を設定してください。

```env
LINE_CHANNEL_SECRET=xxxxxxxxxxxxxxxx
LINE_CHANNEL_ACCESS_TOKEN=xxxxxxxxxxxxxxxx
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxx

# SEARCHコマンドを使う場合のみ
GOOGLE_API_KEY=xxxxxxxxxxxxxxxx
GOOGLE_CX=xxxxxxxxxxxxxxxx

# 任意
PORT=5000
DKIS_SETTINGS_PATH=dist/settings.json
```

`.env.example` がない環境では、上記の内容で `.env` を作成してください。

### 4. 起動

```bash
uv run python main.py
```

起動すると、標準では以下で待ち受けます。

```text
http://127.0.0.1:5000
```

ヘルスチェック:

```text
GET /
```

Webhook:

```text
POST /webhook
```

---

## LINE Developers 側の設定

LINE Developers Console で Messaging API チャンネルを作成し、Webhook URL にデプロイ先のURLを設定します。

```text
https://<your-domain>/webhook
```

設定の目安:

- Webhook: 有効
- 応答メッセージ: 無効推奨
- あいさつメッセージ: 任意

ローカルでテストする場合は ngrok などで外部公開URLを作り、その `/webhook` をLINE側に登録してください。

---

## Run Locally

```bash
uv sync
uv run python main.py
```

Windows PowerShell:

```powershell
uv sync
uv run python main.py
```

---

## デプロイ例

Render などのWebサービスにデプロイする場合:

1. GitHubリポジトリを接続
2. Build Command に `uv sync` を設定
3. Start Command に `uv run python main.py` を設定
4. Environment Variables に必須環境変数を設定
5. 発行されたURLの `/webhook` をLINE Developers Consoleに登録

---

## 設定ファイル

`dist/settings.json` で主に以下を管理します。

- メインシステムプロンプト
- 使用するOpenAIモデル
- RETRY上限
- 会話履歴ターン数
- Google検索件数
- ニュース取得件数
- 天気APIタイムアウト
- AI入力フォーマット

`DKIS_SETTINGS_PATH` を指定すると、別の設定ファイルを読み込めます。

---

## AI応答形式

AIは以下の形式で応答します。

```text
[CMD]SPEAK
[ARGS]none
[ARGS-2]{"retry": false}
[TEXT](無)こんにちは、マスター。今日は何を調べますか？
[NOTE]通常会話として応答。
```

多段処理が必要な場合は以下のように `retry` を有効にします。

```text
[CMD]SEARCH
[ARGS]{"query":"今日のAIニュース", "result_count":5}
[ARGS-2]{"retry": true}
[TEXT](無)調べてみますね。
[NOTE]検索結果を取得してから回答する。
```

`retry=true` の場合、ツール結果が `RI` として次のモデル入力に渡され、最終回答を生成します。

---

## コマンド詳細

### SPEAK

通常会話を返します。

```text
[CMD]SPEAK
[ARGS]none
[ARGS-2]{"retry": false}
[TEXT](笑)了解です、マスター。
[NOTE]通常会話。
```

### SEARCH

Google Custom Search APIを使って検索します。  
`GOOGLE_API_KEY` と `GOOGLE_CX` が未設定の場合は、設定不足の案内を返します。

```text
[CMD]SEARCH
[ARGS]{"query":"OpenAI 最新ニュース", "result_count":5}
[ARGS-2]{"retry": true}
[TEXT](無)検索して確認します。
[NOTE]Google検索を実行。
```

### NEWS

Google News RSSからニュースを取得します。

```text
[CMD]NEWS
[ARGS]{"query":"生成AI", "location":"日本", "time_filter":"today", "max_items":5}
[ARGS-2]{"retry": true}
[TEXT](無)ニュースを見てきます。
[NOTE]ニュース検索を実行。
```

### WEATHER

Open-Meteoで天気を取得します。  
LINE版には現在地フォールバックがないため、`現在地` や空欄ではなく、具体的な地名が必要です。

```text
[CMD]WEATHER
[ARGS]{"w_location":"広島"}
[ARGS-2]{"retry": true}
[TEXT](無)天気を確認します。
[NOTE]天気情報を取得。
```

### READ-PAGE

指定URLのWebページ本文を抽出します。

```text
[CMD]READ-PAGE
[ARGS]{"url":"https://example.com"}
[ARGS-2]{"retry": true}
[TEXT](無)ページを読んでみます。
[NOTE]Webページ本文を取得。
```

### SAVE-LOG

LINE版では会話ログのファイル保存は行いません。  
実行された場合は、その旨を案内します。

---

## 注意事項

- LINEの返信メッセージには文字数・通数制限があります。長文は分割して返信されます。
- WebhookはLINE署名検証を行います。
- Google検索機能には `GOOGLE_API_KEY` と `GOOGLE_CX` が必要です。
- 天気機能では現在地GPSを使いません。地名を明示してください。
- `READ-PAGE` はサイト構造によって本文抽出に失敗する場合があります。
- APIキーを含む `.env` は絶対に公開しないでください。

---

## 開発メモ

- `main.py` は起動専用です。
- `line_bot_app/app.py` がFlaskとLINE webhookを担当します。
- `line_bot_app/engine.py` がユーザー別履歴、AI呼び出し、RETRY連鎖を担当します。
- `line_bot_app/commands.py` が各ツール処理を担当します。
- `line_bot_app/settings_loader.py` が `dist/settings.json` を読み込みます。

---

## 今後の改善候補

- `.env.example` の整備
- Render向け設定例の追加
- GitHub Actionsでの起動チェック
- READMEにLINE Developers画面の設定スクリーンショット追加
- ツール結果の最大文字数制限
- 会話履歴の永続化
- テスト追加

---

## ライセンス

未定。

---

## 作者

SuperGrave
