# ロリポップへ「ワンボタン」アップロード設定

GitHub の `word/word/` を、ボタン1つでロリポップへ一括アップロードします。
FTPのパスワードは GitHub の Secrets に安全に保存され、コードや会話には残りません。

## 1. ロリポップのFTP情報を確認

ロリポップのユーザー専用ページ →「ユーザー設定」→「アカウント情報」で次を確認:

- FTPサーバー（例: `ftp.lolipop.jp` など、表示されたホスト名）
- FTPアカウント（FTP・WebDAVアカウント）
- FTPパスワード（FTP・WebDAVパスワード）

## 2. GitHub に Secrets を3つ登録

リポジトリの **Settings → Secrets and variables → Actions → New repository secret** で登録:

| 名前 | 値 |
|------|----|
| `FTP_SERVER`   | ロリポップのFTPサーバー（ホスト名） |
| `FTP_USERNAME` | FTPアカウント |
| `FTP_PASSWORD` | FTPパスワード |

## 3. 保存先パスを確認

`.github/workflows/deploy.yml` の `server-dir: /word/word/` が、実サイトで
`https://(ドメイン)/word/word/` に対応するサーバー上の場所と一致しているか確認。
違う場合だけ書き換えてください（多くの場合このままでOK）。

## 4. 実行（ワンボタン）

GitHub の **Actions タブ → 「Deploy to Lolipop」→ 右上の「Run workflow」** を押すだけ。
`word/word/` のHTMLがロリポップへアップロードされます。
（main に push した時も自動で実行されます。自動が不要なら deploy.yml の `push:` ブロックを削除）

## 注意

- **`data/*.json` は消さないでください。** デプロイは「`word/word/` の中身をサーバーに同期」する動作のため、
  過去バッチのデータが消えると、サーバー側の該当ページも削除されます（追記式で運用すれば常に全ページが揃います）。
- `protocol: ftps` で失敗する場合は `ftp` に変更して再実行。
- 初回は数ページで試し、ロリポップ上で正しく表示されるか確認してから本格運用を。
