# Codie 1.6.0 HTTP API — ベータ版接続ガイド

1.6.0 を実行している Codie に、LAN 内のスクリプト、デスクトップアプリ、モバイルアプリから接続するテスター向けのガイドです。master ブランチのソースコードに基づいており、全端末への配信完了や全 API の実機検証完了を意味するものではありません。

## パッケージの使い方

`Codie-HTTP-API-1.6.0.zip` を展開し、`index.html` を開きます。全公開 API のパラメーター、要求・応答構造、機能権限をオフラインで検索できます。操作数とパス数は `manifest.json` を参照してください。サーバーの起動や端末の実験室ページへのアクセスは不要です。

| ファイル | 用途 |
| --- | --- |
| `index.html` | 全 API のオフラインリファレンス。検索・ブラウザー印刷に対応 |
| `guide.zh-CN.html` / `GUIDE.zh-CN.md` | 中国語ガイド |
| `guide.ja.html` / `GUIDE.ja.md` | 日本語ガイド |
| `guide.en.html` / `GUIDE.en.md` | 英語ガイド |
| `guide.de.html` / `GUIDE.de.md` | ドイツ語ガイド |
| `guide.fr.html` / `GUIDE.fr.md` | フランス語ガイド |
| `openapi.json` | 既定の英語版 OpenAPI 3.1 定義。API ツール側で HMAC 署名の実装が必要 |
| `openapi.zh-CN.json` | 中国語版 OpenAPI。パス・フィールド・列挙値・署名要件は英語版と同じ |
| `examples/local_api.py` | ペアリングとチャット。テキストは Python 3 標準ライブラリを使用し、マイクの依存パッケージはスクリプトに記載 |
| `examples/external_tools.py` | 外部ツールの登録、接続、呼び出し結果の返送 |
| `examples/media_session.py` | 一時的な画像・音声・動画セッション |
| `manifest.json` | 端末バージョン、ソースコミット、生成日時、ファイルの SHA-256 |

5 言語のガイドを HTML と Markdown で同梱しています。`index.html` 上部から言語を選択できます。全 API リファレンスのナビゲーションは中国語、説明文は既定で英語です。OpenAPI は英語版 `openapi.json` と中国語版 `openapi.zh-CN.json` を同梱しています。パス、フィールド名、列挙値、サンプルコードは共通です。

OpenAPI の `info.version=1` は既存の API 定義のバージョンです。端末ソフトウェアのバージョン 1.6.0 は、`x-device-version` とマニフェストに別途記載しています。

## 1. 端末の準備

1. 端末が 1.6.0 で、ユーザーアカウントにログイン済みであることを確認します。
2. PC と端末を相互通信できる LAN に接続し、端末の IP を確認します。既定のアドレスは `http://DEVICE_IP:8081` です。変更済みの場合は実際のポートを使います。
3. 会話の前に端末をスリープ解除し、ロックを解除します。API は端末の現在のユーザーとキャラクタースロットを使用します。
4. 疎通確認後、サンプルでペアリングします。

```sh
curl http://DEVICE_IP:8081/v1/api/auth/ping
```

HTTP のベースパスは `/v1/api` です。完全な URL の例は `http://DEVICE_IP:8081/v1/api/data/user/system` です。サンプルに渡す端末アドレスにはスキーム、ホスト、ポートだけを指定し、`/v1/api` は付けません。

## 2. テキスト会話を試す

展開したパッケージのルートディレクトリで、Python のサンプルを実行します。

```sh
python examples/local_api.py http://DEVICE_IP:8081 "こんにちは" --credentials credentials.json
```

初回は端末に表示されたペアリングコードをターミナルに入力します。サンプルは認証情報を保存し、現在のスロットを取得してテキスト会話を送信し、状態をポーリングして返答を表示します。次回以降は同じ認証情報ファイルを再利用します。

端末で返答を再生する場合は `--speech device` を追加します。

```sh
python examples/local_api.py http://DEVICE_IP:8081 "こんにちは" --credentials credentials.json --speech device --timeout 180
```

既定は `--speech none` で音声再生なし、待機時間は 120 秒です。`device` の `completed` は生成と端末での再生の両方が完了した状態です。クライアントのタイムアウトではサーバー側のターンはキャンセルされません。表示されたターン ID で引き続き照会できます。

サンプルは明示的に `session` 認証情報を要求します。有効期間は 12 時間で、端末サービスの再起動後は無効です。既存の長期認証情報が必要な場合は初回に `--auth-scope permanent` を追加します。session は公開ユーザーAPIを利用でき、管理APIには permanent が必要です。端末の機能制限も適用されます。既存ファイルは上書きも自動変更もされないため、再ペアリング時は新しいファイル名を指定してください。認証情報は適切に保管し、フィードバックや文書配布に含めないでください。

## 3. ペアリングの実装

HMAC が不要な公開 HTTP 操作は次の 5 個のみです。

| メソッド | パス（ベースパスを含む） | 用途 |
| --- | --- | --- |
| GET | `/v1/api/auth/ping` | 疎通確認 |
| GET | `/v1/api/auth/bootstrap` | 現在の `user_id`、ログイン状態、言語 |
| POST | `/v1/api/auth/pair/request` | コードを端末に表示し `challenge_id` を返す。有効期間 5 分 |
| POST | `/v1/api/auth/pair/confirm` | コードを送信し認証情報を取得 |
| POST | `/v1/api/auth/pair/cancel` | ペアリング要求をキャンセル |

ペアリング確定の JSON：

```json
{
  "challenge_id": "pair/request が返した値",
  "pairing_code": "端末に表示されたコード",
  "auth_scope": "session"
}
```

`auth_scope` は `session` または `permanent` です。省略時の API 既定値は `permanent` ですが、サンプルは `session` を明示します。`data.local_secret`、`data.local_session_id`、`data.local_session_expires_at_ms`、`data.auth_scope` と、bootstrap の `data.user_id` を保存します。スクリプトとネイティブアプリは `Origin/Referer` を省略できます。ブラウザーには既存の送信元検証が適用されます。

## 4. 各業務リクエストの署名

次の順序で `|` で連結し、末尾に改行を付けません。

```text
METHOD|PATH|TIMESTAMP|NONCE|PAYLOAD_MODE|BODY_HASH|USER_ID
```

```text
signature = hex_lower(HMAC_SHA256(UTF8(local_secret), UTF8(canonical)))
```

| ヘッダー | 値 |
| --- | --- |
| `X-Local-Sig` | 64 桁の小文字 16 進数による署名 |
| `X-Local-Timestamp` | Unix 秒の文字列。端末時刻との許容差は 300 秒 |
| `X-Local-Nonce` | 毎回新しい乱数。UUID 推奨。1～128 文字、縦棒と改行は不可 |
| `X-Local-Payload-Mode` | 下表のモード |
| `X-Local-Body-Hash` | 下表のハッシュ |
| `X-Local-User-Id` | bootstrap が返す現在の `user_id` |
| `X-Local-Session-Id` | ペアリング結果の `local_session_id` が空でない場合は必須 |

`METHOD` は大文字です。`PATH` は `/v1/api` を含む URL デコード後のパスで、ホストとクエリ文字列を含めません。鍵には元の `local_secret` 文字列の UTF-8 バイトを使い、16 進デコードはしません。実際に送信する本文バイトを署名し、署名後に JSON を再整形しないでください。

| 本文 | PAYLOAD_MODE | BODY_HASH |
| --- | --- | --- |
| 本文なし | `empty` | 空バイトの SHA-256、小文字 16 進数 |
| JSON | `json` | 実際の JSON バイトの SHA-256、小文字 16 進数 |
| ファイルフォーム | `multipart` | 固定文字列 `UNSIGNED-PAYLOAD` |
| zstd 圧縮チャンク | `zstd-stream` | 実際の圧縮バイトの SHA-256、小文字 16 進数 |
| gzip 圧縮チャンク | `gzip-stream` | 実際の圧縮バイトの SHA-256、小文字 16 進数 |

空本文のハッシュは `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` です。圧縮チャンクは `application/octet-stream` を使い、明示的に対応する API のみで使用します。multipart の `Content-Type` と boundary は HTTP ライブラリに生成させてください。

## 5. 統一会話 API

| メソッド | `/v1/api` からの相対パス | 用途 |
| --- | --- | --- |
| POST | `/data/user/chat/turns` | 会話ターンを作成 |
| GET | `/data/user/chat/turns/{turn_id}` | 状態、文字起こし、返答を取得 |
| POST | `/data/user/chat/turns/{turn_id}/input/complete` | 端末マイク入力を終了して生成を続行。空本文 |
| POST | `/data/user/chat/turns/{turn_id}/cancel` | そのターンの入力、生成、再生をキャンセル。空本文 |

最初に `GET /data/user/slot` の `data.slot_number` を取得します。新規ターンごとに新しい UUID を `request_id` に使い、`slot` を現在のスロットに置き換えます。

```json
{
  "request_id": "3f743580-bd45-4c24-96c0-61a67ac5f741",
  "slot": 1,
  "input": {"type": "text", "text": "こんにちは"},
  "output": {"speech": "none"}
}
```

| 入力 | Content-Type | 内容 |
| --- | --- | --- |
| `text` | `application/json` | `input.text` 必須。前後の空白除去後に空でないこと。最大 16000 文字 |
| `device_microphone` | `application/json` | 端末マイクで録音。`text` は送らず、`input/complete` で入力終了 |
| `image_text` | `multipart/form-data` | `request` に設定の JSON 文字列。`input.type=image_text`、`text` は任意。ファイル欄は `image` |
| `audio_file` | `multipart/form-data` | `request` に設定の JSON 文字列。`input.type=audio_file`、`text` は送らない。ファイル欄は `audio` |

画像または WAV は最大 1 MiB。WAV は PCM16、16 kHz、モノラル、最大30秒です。必須の `output.speech` は `none`、`device`、`client` に対応し、未知のフィールドは拒否されます。

任意の `interrupt` は既定で `true` です。ターン開始時にデバイスが会話・発話・タスク実行中なら、その会話を先に終了します。`false` の場合は中断せず、ターンは `failed`、`error.code=DEVICE_BUSY`（`retryable=true`）で終わり、そのユーザーメッセージは履歴から取り消されます。後で再試行してください。

任意の `retention` はターンがデバイスに残すものを決め、既定は `full` です。チャット履歴に保存し、会話状態・要約・長期記憶・日記を更新します（いずれもそのターンのチャット履歴から抽出）。`chat` はチャット履歴にだけ保存し、これらの更新は行いません。`none` は何も残しません。チャット履歴・画像の保存・更新をすべて行わず、返答はそのターンの `assistant_text` にのみ入り、`user_chat_ids` と `assistant_chat_ids` は空になります。いずれの値でも、返答は既存のチャット履歴と記憶を読んで生成されます。

任意の `tools` はこのターンでモデルがツールを呼び出せるかを決め、既定は `all` です。モデルはセッションで利用できるデバイスツールと外部ツールを呼び出せます。`none` にするとこのターンではモデルにツールを一切提供しません。返答だけが必要で、デバイス操作や外部タスクを起こしてはならないターンに向いています。

`input.type=audio_stream` も JSON で利用できます。`audio.format=pcm_s16le`、`sample_rate=16000`、`channels=1`、`container=raw` を指定し、最大30秒です。音声ストリーム入力または `speech=client` は受付後15秒以内に、署名付き WebSocket GET で `/v1/ws/chat/turns/{turn_id}/audio` に接続し、`ready` を待ちます。出力はメタデータに応じて MP3、WAV、Ogg Opus です。`audio.end` 後に最終シーケンスの `audio.ack` を送信します。これは受信確認であり再生完了ではありません。切断でタスクは終了し、再接続・再送には対応しません。メッセージ定義は OpenAPI を参照してください。

```sh
python -m pip install websocket-client sounddevice av
python examples/local_api.py http://DEVICE_IP:8081 --mic --seconds 10 --credentials credentials.json --timeout 180
```

受理応答の `data.turn_id` で、例えば 1 秒ごとに照会します。終端状態は `completed`、`failed`、`cancelled` です。完了時は `data.assistant_text`、失敗時は `data.error` を読みます。`transcript` は文字起こし、`user_chat_ids` と `assistant_chat_ids` は履歴レコードとの関連付けです。中間状態を最終成功と判断しないでください。

中間状態は `queued`（待機）、`starting`（接続開始）、`receiving_input`（録音・ストリーム入力）、`thinking`（生成待ち）、`responding`（応答生成・配信）、`cancelling`（取消処理中）です。`input_complete` は入力終了のみを意味し、`transcript` は終端状態より前に取得できる場合があります。`device_microphone` は最大30秒で入力を自動終了し、`input/complete` で早く終了できます。`speech=none` でもモデルは実行しますが、音声再生はしません。履歴を保存するかどうかは `retention` で決まります。各リクエスト前に `/data/user/device/state` で現在の利用条件を確認してください。

`202` は受理のみを意味します。保持期間中に同一認証情報、同一 `request_id`、同一内容で再送すると元のターンを返し、内容が異なると `409 REQUEST_ID_CONFLICT` になります。HTTP 再試行でも署名 nonce は毎回更新します。`interrupt=true` の場合、新規ターンは旧セッションを停止しますが、同一要求の再送では再度割り込みません。

終端ターンの保持期間は 30 分、キャッシュ上限は 256 件です。端末サービス再起動でターンと冪等性の記録は失われます。キャラクター・ユーザー変更、認証失効、履歴削除により、旧ターンが停止または参照不能になることがあります。キャンセルできるのは、その認証情報で作成した指定ターンのみです。

### 独立したテキスト読み上げ（TTS）

Home Assistant の実行結果など、外部アプリで確定したテキストを現在のキャラクターが直接読み上げます。LLM の呼び出し、キャラクターの返答生成、チャット履歴の書き込みは行いません。

`POST /data/user/tts/utterances` に JSON を送信します。パスは `/v1/api` からの相対パスで、上記の HMAC 署名を使います：

```json
{
  "request_id": "e14eec98-5f67-445f-b605-bcc0b23f8f75",
  "slot": 1,
  "text": "リビングの照明を消しました。"
}
```

`GET /data/user/tts/utterances/{utterance_id}` で照会します。取り消しは `POST /data/user/tts/utterances/{utterance_id}/cancel`、本文は空です。

202 は受付のみを示し、`data.utterance_id` を取得します。状態は `queued`、`synthesizing`、`playing`、`cancelling`、終端状態は `completed`、`failed`、`cancelled` です。合成と端末再生の両方の完了で completed になり、エラーは `data.error` に入ります。text は前後の空白除去後 1–16000 文字、JSON 本文は最大 128 KiB です。現在のスロットとキャラクターの声・言語・音量を使います。端末の起動・ロック解除が必要です。新しい読み上げは前景セッションを中断し、新しい会話も読み上げを中断できます。再生待ちキューはありません。

同じ認証情報・request_id・内容の再送は元のタスクを返します（200）。内容の競合は409です。request_id と最大256件のキャッシュはチャットと共用、終端後30分保持、再起動で消去されます。認証失効、ユーザー・キャラクター変更、履歴消去により停止または参照不可になる場合があります。タスクは最大180秒です。音声ファイルやストリームは返しません。

```sh
python examples/local_api.py http://DEVICE_IP:8081 "リビングの照明を消しました。" --credentials credentials.json --tts --timeout 180
```

`playing` は現在の再生ストリームの端末再生開始通知に基づきます。アニメーションが有効な場合、Talking アニメーションをランダムに1つ選びます。取り消し受付は202、取り消し中・取り消し済みは200、完了・失敗済みは409です。クライアントの待機がタイムアウトしても、端末のタスクは自動で取り消されません。

## 6. その他の機能と結果確認

正確なパス、フィールド、応答は全 API リファレンスと OpenAPI を参照してください。

| 分類 | 機能 |
| --- | --- |
| 接続・ペアリング | 疎通、ペアリング、キャンセル |
| 端末・状態 | システム情報、権限、端末・会話状態、AI 言語 |
| キャラクター・能動的対話 | 属性、スロット、モデル・衣装アップロードとエクスポート、読み込み、リセット、能動的対話設定 |
| 会話・履歴 | ターン、履歴照会・削除、画像 |
| 記憶・日記 | 照会と編集 |
| アラーム・メモ | リソース管理とアラーム有効化・無効化 |
| 音楽 | アップロード、ファイル管理、再生、状態 |
| 音声・サウンド | 権限、独立テキスト読み上げ、固定文の試聴、制御、音声クローン |
| ライト | 作品、プレビュー、コンパイル、再生、適用、全体スイッチ |
| AI サービス | サービス一覧、接続設定とテスト、能力ルート、システムプロンプト、構成・バックエンド切替、最新の実行エラー |

ペアリングで機能権限が増えることはありません。`/data/user/system` の `capabilities` を確認してください。各操作の権限は `x-capabilities` に記載しています。実験室の API ドキュメントページと AI サービス設定は、すべての機能タグで利用できます。既存のペアリング署名認証は引き続き適用されます。

成功応答が保存やキュー投入を意味し、動作完了を意味しない API もあります。会話はターン、モデルのアップロード・適用・エクスポートはタスクまたは読み込み状態、音楽は再生状態、試聴は `/data/user/tts_audition/state` を照会します。試聴は `voice_id` と任意の `group_key` のみ受け付け、現在の AI 言語の固定文を使用します。任意の読み上げ文は指定できません。

モデルファイルは最大 200 MiB、音声クローン用ファイルは最大 10 MiB。音楽の一括要求は最大 200 MiB、64 ファイルです。通常の署名本文と圧縮チャンクは最大 8 MiB。フォーム全体にはフィールド分の容量も必要です。詳細は各 API の制約を確認してください。

任意のリアルタイム状態チャネルは `/v1/ws/events` です。署名付き GET ハンドシェイクを使用し、クエリに `sig`、`timestamp`、`nonce`、`payload_mode=empty`、`body_hash`、`user_id` と、空でない場合の `session_id` を指定します。署名対象パスはクエリなしの `/v1/ws/events` です。状態スナップショット用であり、会話生成テキストのストリーミングではありません。認証失効やユーザー変更などで接続が閉じます。

`ready` の受信後、`{"type":"subscribe","request_id":"sub-1","data":{"events":["chat_history","chat_turn"]}}` を送信し、成功の `result` を待ちます。`events` は購読集合全体を置き換え、空配列はすべて解除します。再接続後は再購読して HTTP で状態を復元します。公開仕様には Release ビルドと RELEASE タグの両方で有効な機能のみを含みます。OpenAPI の既定の説明は英語です。

## 7. 問題解決とフィードバック

| 現象・コード | 対応 |
| --- | --- |
| 接続できない | IP、ポート、LAN 疎通を確認し、まず ping を実行 |
| `401 REQUEST_EXPIRED` | 時刻を合わせ、新しいタイムスタンプと署名を生成 |
| `401 NONCE_REUSED` | ポーリング・再試行を含む全要求で新しい nonce を使用 |
| `401 LOCAL_SESSION_EXPIRED` | 新しい認証情報ファイル名で再ペアリング |
| `401 LOCAL_USER_MISMATCH` | 現在のユーザーを確認し、bootstrap とペアリングを再実行 |
| `401 LOCAL_SECRET_MISMATCH` / `INVALID_SIGNATURE` | 元の鍵、署名パス、本文バイト、ヘッダーを確認 |
| `403` | 応答のエラーと端末権限を確認。ペアリングで権限は回避不可 |
| `409 SLOT_MISMATCH` | 現在のスロットを再取得 |
| `409 DEVICE_BLOCKED` / `AMBIENT_MODE_BLOCKED` | `reason` を確認し、スリープ・ロック解除と現在モードの会話可否を確認 |
| `413` / `415` | サイズ、Content-Type、入出力タイプ、WAV 形式を確認 |
| `503` | エラーコードに従い、サービス準備状態やターン上限を確認 |

OpenAPI の `code.enum` に会話・TTS・アラーム・メモ・クローン録音のローカルHTTPエラーを列挙しています。`x-error-codes.http` は機械用一覧、`x-error-codes.task` は既知の非同期エラーで、提供元が拡張する場合があります。HTTP 202 後の失敗は `data.error` を確認してください。許可されないペアリング元は `403 PAIRING_SOURCE_NOT_ALLOWED`、冪等キーの内容競合は `409 REQUEST_ID_CONFLICT`、メモが存在しない場合は `404 MEMO_NOT_FOUND` です。

`x-error-codes` は AI サービス、モデル処理、キャラクターリセット、記憶、音楽の既知のエラーも含みます。日記の旧形式コードは `legacy_error` に列挙し、応答の `error` に入ります。非同期エラーには `NO_SPEECH` があり、`DEVICE_INTERRUPTED.reason=vad_start` は音声開始検出を表します。reason は拡張可能です。

エラー構造は機能ごとに異なります。HTTP ステータスを先に確認し、各 API の `code`、`message`、`reason` などを読みます。書き込み操作を無条件に再試行しないでください。

報告には端末バージョン、`manifest.json` のソースコミット、発生時刻と端末のタイムゾーン、メソッド・パス、HTTP ステータス、機密情報を除去した応答、該当する `request_id` / `turn_id`、端末の現象を含めてください。`local_secret`、認証情報ファイル、署名ヘッダーは送らないでください。

管理 API、デバッグ音声、診断ログ、内部保守は本公開契約の対象外です。本パッケージはソース上の契約を確認したものです。ビルド、実機 API 呼び出し、クラウド E2E は個別に検証が必要です。
