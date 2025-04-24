---
date: '2025-04-24T11:31:02+09:00'
title: 'Unix Codex'
---

https://github.com/openai/codex は、Unix シェルならではのパイプやスクリプト管理との相性が良く、AI を活用したシームレスなコードレビューを実現できる。  

一方で、自動実行に対するセキュリティリスクも存在するため、Git 管理とサンドボックス設定などで防御を固める必要がある。例えば、下記のようなgh-prs.sh のようなscriptと組み合わせたワークフローを構築し、codex -m o3 を活用した差分解析や自動レビューを行うことで、高品質な開発プロセスを確立できる。

`codex -m o3 $'コードの構造を踏まえて内容をレポートし、さらに改善提案やレビューを行ってください\n\n'"$(gh-prs.sh)"`

```bash
#!/usr/bin/env bash
#
# gh-prs.sh
# ------------
# GitHub CLI で Issue 一覧を取得し、peco で選択して
# `gh pr view` と `gh pr diff` で詳細と差分を表示するシンプルなスクリプト。
#
# 依存:
#   - GitHub CLI (gh) : https://cli.github.com/
#   - peco            : https://github.com/peco/peco
#
# 使い方:
#   ./gh-prs.sh [gh issue list のオプション]
#
# 例) 自分が担当で open 状態の Issue を対象にする
#     ./gh-prs.sh --assignee @me --state open
#
set -euo pipefail

# 表示調整
SUMMARY_MAX=120 # peco で視認しやすいようタイトルを truncate する長さ

# pr 一覧取得 (番号とタイトルのみ、タブ区切り)
LIST=$(gh pr list --limit 100 "$@" \
    --json number,title \
    --template '{{range .}}{{printf "%.0f\t%s\n" .number .title}}{{end}}' || true)

if [[ -z "${LIST}" ]]; then
    echo "該当する pr がありませんでした。" >&2
    exit 0
fi

# peco で選択
pr_NUM=$(echo "${LIST}" |
    awk -v max="${SUMMARY_MAX}" -F'\t' '
        {
          title=$2;
          if(length(title) > max) title=substr(title,1,max)"…";
          printf "%s\t%s\n",$1,title
        }' |
    peco --prompt "Select pr > " |
    cut -f1)

if [[ -z "${pr_NUM}" ]]; then
    echo "pr が選択されませんでした。" >&2
    exit 1
fi

# 詳細表示 + diff
gh pr view "${pr_NUM}"
gh pr view "$pr_NUM" \
    --json comments \
    --jq '.comments[]
      | select(.authorAssociation != "NONE")
      | "\(.author.login)\n\(.body)\n"'
gh pr diff "${pr_NUM}"
```