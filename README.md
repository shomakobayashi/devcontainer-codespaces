# GitHub Copilot セキュリティ対策 検証リポジトリ

DevelopersIO ブログ「GitHub Copilot のセキュリティ対策として Codespaces + Dev Containers 構成を試してみた」の検証用リポジトリです。

## 構成

```
.
├── .devcontainer/
│   └── devcontainer.json   # Dev Container 設定（拡張機能 allowlist 等）
├── src/
│   └── index.js
└── README.md
```

## このリポジトリで検証していること

- Dev Container による拡張機能の allowlist 化
- Copilot Agent Mode の自動承認強制 OFF
- 機密ファイル（.env, *.key 等）の Copilot 除外設定
- Codespaces のポート可視性制御（private 強制）

## Codespaces で開く

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/YOUR_ORG/YOUR_REPO)
