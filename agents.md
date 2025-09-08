# Agents Guidelines

以下は、このリポジトリで文書（特に `content/posts` 配下の Markdown）を作成・更新するときのルールです。

## Lint（Textlint）必須

- 文章を作成・更新したら、必ず Textlint を実行し、エラー/警告がなくなるまで修正する。
- 実行例: `npx textlint content/posts/2hoppeerswap.md`
- 対象ファイルは作成/更新したファイルに置き換えて実行すること（例: `npx textlint content/posts/<your-post>.md`）。
- 複数ファイルをまとめて確認したい場合は、必要に応じてパスを調整して実行すること（例: `npx textlint "content/posts/**/*.md"`）。

参考: https://agents.md/

