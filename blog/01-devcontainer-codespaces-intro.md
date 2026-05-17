製造ビジネステクノロジー部の小林です。

GitHub Copilot を使っていますが、最近次のような点が気になりました。

- コードが外部に漏れないか
- メンバーが野良の MCP サーバーを使い始めたらどうするか
- 誰がいつどこに通信したか把握できるか

これらを本格的に解決するには GitHub Codespaces の組織ポリシーによる権限制限が有効かと思います。その前段として開発環境と Copilot の設定をコードで統一することが第一歩になると考え、まず Dev Containers を試してみました。

## 前提

本記事での動作確認環境は以下のとおりです。

| 項目                | バージョン           |
| ------------------- | -------------------- |
| マシン              | MacBook              |
| macOS               | Tahoe 26.3.1         |
| Docker Desktop      | （執筆時点の最新版） |
| VS Code             | （執筆時点の最新版） |
| Dev Containers 拡張 | v0.401.0             |

## Dev Containers とは

Dev Containers（Development Containers）は、開発環境を Docker コンテナとして定義する仕組みです。リポジトリに `.devcontainer/devcontainer.json` を置くだけで、チーム全員が同じ環境で開発できます。

Dev Containers の詳細は下記の記事をご覧ください。
https://zenn.dev/yamato_snow/articles/fcb3cf8cf0ad03

設定ファイルの配置は以下のとおりです。

```
プロジェクト/
└── .devcontainer/
    ├── devcontainer.json   ← 環境定義ファイル
    └── Dockerfile          ← （任意）カスタム環境
```

VS Code の「Dev Containers」拡張と組み合わせて使うと、エディタの見た目・操作感はほぼそのままに、実行環境だけをコンテナに閉じ込められます。

### VS Code「Dev Containers」拡張機能

Microsoft 公式の拡張機能で、3900 万以上のインストール実績があります。

![スクリーンショット 2026-05-18 1.18.43](https://devio2024-media.developers.io/image/upload/f_auto/q_auto/v1779034735/2026/05/18/bnlfj5epes1svnjw9gsx.png)

インストールすると VS Code にリモートエクスプローラーパネルが追加され、起動中のコンテナの詳細をサイドバーから確認できます。

![スクリーンショット 2026-05-18 1.19.50](https://devio2024-media.developers.io/image/upload/f_auto/q_auto/v1779034813/2026/05/18/yathk3zd2kwuofbrq5j8.png)

**属性**

![スクリーンショット 2026-05-18 1.20.55](https://devio2024-media.developers.io/image/upload/f_auto/q_auto/v1779034897/2026/05/18/in9rknzaupi4f6cesd4j.png)

| 項目           | 内容の例                                                        |
| -------------- | --------------------------------------------------------------- |
| ワークスペース | ローカルのプロジェクトディレクトリのパス                        |
| 名前           | コンテナ名（自動生成）                                          |
| イメージ       | `mcr.microsoft.com/devcontainers/typescript-node:4-24-bookworm` |
| 構成           | `.devcontainer/devcontainer.json` のパス                        |

**マウント**

| 種別                   | 項目               | 内容                                  |
| ---------------------- | ------------------ | ------------------------------------- |
| マウントをバインドする | ソース             | ローカルのプロジェクトディレクトリ    |
|                        | 宛先               | `/workspaces/devcontainer-codespaces` |
|                        | I/O パフォーマンス | OS をブリッジすることで削減           |
| ボリュームマウント     | ボリューム         | `vscode`                              |
|                        | 宛先               | `/vscode`                             |

コンテナがどのイメージで動いているか、どのディレクトリをマウントしているかをコマンドなしで把握できるため、環境の確認やトラブルシュートがしやすいですね。

https://code.visualstudio.com/docs/devcontainers/containers#_quick-start-open-a-git-repository-or-github-pr-in-an-isolated-container-volume

### devcontainer.json に定義できること

- 使用する Docker イメージ（Node.js、Python、Java など）
- 自動インストールする VS Code 拡張機能
- VS Code の設定（フォーマッター、Copilot の設定など）
- コンテナ起動時に実行するコマンド（`npm install` など）
- ポートフォワード設定

## Dev Containers を使うと嬉しいこと

### 開発環境がコードで管理できる

`devcontainer.json` をリポジトリに置くだけで、チーム全員が同じ環境で開発を始められます。「自分の PC では動くのに」という問題が起きにくくなります。

新しいメンバーがジョインしたときも、リポジトリを clone してコンテナを起動するだけでセットアップ完了です。README に「まず Node.js をインストールして…」と書く必要がなくなります。

### OS・マシンの差異を吸収できる

Mac・Windows が混在するチームでも、コンテナの中は全員同じ Linux 環境です。「Windows だと改行コードが…」「自分だけパスが通らない…」といったトラブルを減らせます。

### AI ツールの設定も統一できる

Copilot の設定（機密ファイルの除外・Agent Mode の挙動など）を `devcontainer.json` に書いておけば、個人の設定に依存せず全員に適用されます。チームで AI ツールを使い始めるときに、最低限の設定を揃えた状態でスタートできます。

## やってみた

### devcontainer.json を作る

```json:devcontainer.json
{
  "name": "Copilot Dev Environment",
  "image": "mcr.microsoft.com/devcontainers/typescript-node:4-24-bookworm",
  "customizations": {
    "vscode": {
      "extensions": [
        "GitHub.copilot",
        "GitHub.copilot-chat",
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode",
        "GitHub.vscode-github-actions",
        "shd101wyy.markdown-preview-enhanced"
      ],
      "settings": {
        // Agent Mode が VS Code タスクを自動実行する前に確認ダイアログを出す
        "github.copilot.chat.agent.runTasks": false,
        // 拡張機能の自動更新を無効化
        "extensions.autoUpdate": false,
        // 機密ファイルを Copilot の補完対象から除外
        "github.copilot.enable": {
          "*": true,
          ".env": false,
          "*.pem": false,
          "*.key": false
        }
      }
    }
  },
  "postCreateCommand": "npm install && gh extension install github/gh-copilot",
  "forwardPorts": [3000],
  "portsAttributes": {
    "3000": {
      "label": "App",
      "onAutoForward": "notify",
      "visibility": "private"
    }
  },
  "remoteUser": "node"
}
```

各プロパティの意味はこちらです。

| プロパティ                           | 内容                                                                                                                |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `name`                               | コンテナの表示名。VS Code のリモートエクスプローラーやステータスバーに表示される                                    |
| `image`                              | 使用する Docker イメージ。Microsoft 公式の Node.js 24 + TypeScript 環境（Debian 12）で、Dockerfile 不要で始められる |
| `customizations.vscode.extensions`   | コンテナ起動時に自動インストールされる拡張機能。Copilot・ESLint・Prettier・GitHub Actions を全員に統一する          |
| `github.copilot.chat.agent.runTasks` | `false` にすることで、Agent Mode が VS Code タスク（tasks.json）を自動実行する前に確認ダイアログを出す              |
| `extensions.autoUpdate`              | `false` にすることで、拡張機能の自動更新を止め、意図しないバージョン変更を防ぐ                                      |
| `github.copilot.enable`              | `.env`・`*.pem`・`*.key` など機密ファイルを Copilot の補完対象から除外する                                          |
| `portsAttributes`                    | ポート 3000 をデフォルトで `private` にする。Codespaces で動かす場合に外部公開を防ぐ                                |
| `remoteUser`                         | `node`（一般ユーザー）で実行する。`root` で動かす必要がなければ非 root が無難                                       |
| `postCreateCommand`                  | コンテナ起動後に自動実行するコマンド。`npm install` で依存関係を入れつつ `gh copilot` 拡張もインストールする        |

なお、`extensions` はあくまで「起動時のプリインストールリスト」であり、リスト外の拡張機能の追加インストールを禁止する機能ではありません（後述）。

### Dev Containers を起動する

`.devcontainer/devcontainer.json` を置いたリポジトリを VS Code で開くと、右下に通知が表示されます。

＜キャプチャ：「コンテナーで再度開く」通知＞

「コンテナーで再度開く（Reopen in Container）」をクリックするとコンテナのビルドが始まります。通知が出ない場合はコマンドパレット（`Cmd+Shift+P`）から `Dev Containers: Reopen in Container` を実行してください。

初回はベースイメージのダウンロードがあるため数分かかります。ビルドが完了すると VS Code がコンテナ内で再起動し、ステータスバー左下に `開発コンテナー: Copilot Dev Environment` と表示されます。

＜キャプチャ：ステータスバーの表示＞

あわせて `postCreateCommand` が自動実行され、`npm install` と `gh copilot` 拡張のインストールが行われます。

＜キャプチャ：postCreateCommand の実行ログ＞

起動後はリモートエクスプローラーパネルからコンテナの状態を確認できます。

＜キャプチャ：リモートエクスプローラー＞

## 実際に試してわかったこと

### `extensions` は「プリインストールリスト」であって「allowlist」ではない

`devcontainer.json` の `extensions` に列挙した拡張機能は、起動時に自動インストールされるものです。ここに書いていない拡張機能のインストールをブロックする機能はありません。

実際に試すと、リストに含まれていない `Vim`

＜キャプチャ＞

### ブラウザ言語設定に応じた拡張機能も自動インストールされる

`devcontainer.json` に含めていないにもかかわらず、ブラウザが日本語設定だったため `Japanese Language Pack` が自動インストールされました。

<キャプチャ>

拡張機能のインストール自体をブロックするには、VS Code の Extension Management ポリシー（GitHub Enterprise で管理するか、VS Code 自体のポリシー設定）が必要です。

### `postCreateCommand` が exit code 127 で失敗する

コンテナ起動時に下記のエラーが出ることがありました。

```
postCreateCommand from devcontainer.json failed with exit code 127.
Skipping any further user-provided commands.
```

exit code 127 は「コマンドが見つからない」を意味します。`postCreateCommand` に `gh extension install github/gh-copilot` を書いていましたが、ベースイメージ `mcr.microsoft.com/devcontainers/typescript-node` には `gh`（GitHub CLI）が含まれていないため失敗していました。

その結果、コンテナ内で `copilot` を実行すると次のように表示されました。

```
Cannot find GitHub Copilot CLI
Install GitHub Copilot CLI? ['y/N']
```

**対処法**：`features` で GitHub CLI をコンテナビルド時にインストールするよう追加します。`features` はベースイメージへのアドオンで、`postCreateCommand` より先に実行されるため、`gh` が使える状態でコマンドが実行されます。

```json
{
  "features": {
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },
  "postCreateCommand": "npm install && gh extension install github/gh-copilot"
}
```

修正後にコマンドパレット（`Cmd+Shift+P`）から `Dev Containers: Rebuild Container` を実行すると、`postCreateCommand` が正常に完了します。

### コンテナのターミナルからローカルのターミナルを開きたいとき

Dev Containers で開いた VS Code のターミナルはコンテナ内に接続されます。ローカル（Mac）のターミナルを開きたい場合は、コマンドパレット（`Cmd+Shift+P`）で `Terminal: Create New Terminal (Local)` を実行するとローカル側のターミナルが起動します。

コンテナ内とローカルを行き来しながら作業するときに便利です。

### おまけ：ローカルと Codespaces の使い分け

|                | ローカル VS Code          | Codespaces                 |
| -------------- | ------------------------- | -------------------------- |
| 必要なもの     | Docker Desktop + 拡張機能 | ブラウザだけ               |
| 起動速度       | 速い                      | やや遅い（クラウドビルド） |
| オフライン作業 | ✅ できる                 | ❌ できない                |

`devcontainer.json` はローカルでも Codespaces でも共通で使えるため、まずローカルで試してから Codespaces に移行するという進め方もスムーズです。

Codespaces の詳細についてはこちらをご覧ください。

https://zenn.dev/yuhei_fujita/articles/github-codespaces-introduction

## まとめ

Dev Containers を試してみて、開発環境と Copilot の設定をリポジトリで管理できるのは素直に便利でした。新しいメンバーが clone してコンテナを起動するだけで同じ状態になり、Copilot の設定も個人任せにならずに済みます。

一方で、最初に挙げた懸念（通信の監視・野良 MCP のブロック・証跡監査）は Dev Containers だけでは解決できません。これらは Codespaces の組織ポリシーと組み合わせて初めて実現できる部分です。

次回は Codespaces を使って組織レベルの統制まで実践してみます。

## 参考リンク

- [開発コンテナーの概要 - GitHub Docs](https://docs.github.com/ja/codespaces/setting-up-your-project-for-codespaces/adding-a-dev-container-configuration/introduction-to-dev-containers)
- [Dev Containers とは？Docker を使った開発環境構築の決定版【図解で完全理解】 - Zenn](https://zenn.dev/yamato_snow/articles/fcb3cf8cf0ad03)
- [いつでもどこでも VS Code が利用できる GitHub Codespaces - Zenn](https://zenn.dev/yuhei_fujita/articles/github-codespaces-introduction)
- [devcontainer.json リファレンス](https://containers.dev/implementors/json_reference/)
