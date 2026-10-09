# edu-deploy-workflows

教育用リポジトリ（Next.js の静的サイト）を、`main` への push で FTP サーバーへ自動公開する共通ワークフローです。

- アップロード先: FTP の `digital/<フォルダー名>/`
- 公開 URL: `http://e.gmobb.jp/omaesensei/digital/<フォルダー名>/`
- `<フォルダー名>` は省略するとリポジトリ名になります

## 新しい教材リポジトリで使う手順

### 1. `next.config.ts` をルートフォルダーを環境変数から読む形にする

```ts
import type { NextConfig } from "next";

const basePath = process.env.NEXT_PUBLIC_BASE_PATH ?? "";

const nextConfig: NextConfig = {
  output: "export",
  distDir: "out",
  basePath,
  assetPrefix: basePath,
};

export default nextConfig;
```

### 2. `.github/workflows/deploy-ftp.yml` を追加する

```yaml
name: Deploy to FTP server

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: renly24/edu-deploy-workflows/.github/workflows/ftp-deploy.yml@main
    secrets: inherit
    # フォルダー名をリポジトリ名と変えたいときだけ指定する
    # with:
    #   project-dir: my-folder-name
```

### 3. Secrets を登録する

リポジトリの Settings → Secrets and variables → Actions → New repository secret で、次の 3 つを登録します。

| Name | 値 |
|---|---|
| `FTP_HOST` | FTP サーバーのホスト名 |
| `FTP_USER` | FTP のユーザー名 |
| `FTP_PASS` | FTP のパスワード |

### 4. `main` に push する

Actions タブの「Deploy to FTP server」が緑になれば公開完了です。実行結果の Summary に公開 URL が表示されます。
`digital/index.html`（教材一覧ページ）へのリンク追加は手作業で行ってください。

## ワークフローがしていること

1. `NEXT_PUBLIC_BASE_PATH=/omaesensei/digital/<フォルダー名>` でビルド
2. ビルド結果にルートフォルダーが反映されているか確認（`next.config.ts` の設定漏れを検出）
3. 名前に `..` を含むファイルを改名し、参照も書き換える（FTP サーバーがこうした名前を拒否するため）
4. FTPS でアップロード（変更分のみ。`--delete` はアップロード先フォルダーの中だけに効き、他の教材には触れない）
5. 公開 URL が 200 を返すか確認

## 困ったとき

- **ログインで証明書エラーになる**: サーバーの証明書が更新された可能性があります。`ftp-deploy.yml` の `FTP_CERT_FINGERPRINT` を新しい指紋に書き換えてください（全教材に反映されます）。
- **画面が崩れる / CSS が読み込まれない**: `next.config.ts` が手順 1 の形になっているか確認してください。
- **Secrets が読めないエラー**: 呼び出し側に `secrets: inherit` があるか、3 つの Secrets がそのリポジトリに登録されているか確認してください。
