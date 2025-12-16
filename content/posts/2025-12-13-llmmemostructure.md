---
date: '2025-12-13T06:56:30+09:00'
title: "AI時代の思考の作業台：事実と推論を分離して残す"
---

# AI時代の「思考の作業台」：事実と推論を分離して、真似できる形で残す

ここで紹介するのは、**コード（事実）**と**思考（問い・仮説・洞察）**を分離して管理し、
AIとの調査・設計・原因究明を「再現可能」にするためのワークフローです。

新規性は、Static/Dynamic/Thinkingをファイル構造として固定し、AIとの議論を再現可能な成果物に落とす点です。

ポイントはシンプルで、次の3つを分けます。

1. **Static（参照設定）**: 何を “事実” として扱うか
2. **Dynamic（事実スナップショット）**: その時点のコード/資料の全量
3. **Thinking（思考）**: 問い・仮説・検証・結論

こうすると、AIと一緒に考えるときに起きがちな問題を減らせます。

- 根拠のない推測（それっぽいが検証不能）。
- 会話ログの散逸（どこまで決めたかが消える）。
- 前提のすり替え（いつのコードの話かが曖昧）。

結果として、「後で戻れる」思考を作れます。

---

## 10分クイックスタート（この記事だけで開始）

### 1) 必要なもの

- `code2prompt`（**固定**）: コード全量を `llm.txt` にパックするツール
  - インストール例: `pipx install code2prompt`（または `pip install code2prompt`）

### 2) 最小のディレクトリを作る

以下を “1プロジェクト” として作ります（名前は自由）。

```text
thinking/
  AGENTS.md
  <project>/
    README.md
    config.json
    llm.txt
    memos/
```

### 3) `AGENTS.md` を置く（コピペ）

AIを「知的パートナー」として動かすための規約です。
この記事の最後に **そのまま使える `AGENTS.md` テンプレート**を載せています。

### 4) `config.json` を書く（コピペ）

`source_path` は “事実（コード）” の参照先です。

```json
{
  "project_name": "my-project",
  "source_path": "/absolute/path/to/my-project",
  "description": "optional",
  "exclude": [
    "**/.git/**",
    "**/node_modules/**",
    "**/vendor/**",
    "**/target/**"
  ]
}
```

### 5) `llm.txt` を生成する（`code2prompt`）

`config.json` の `source_path` を使って、以下を実行します（最短ルートは “直打ち”）。

```bash
code2prompt "/absolute/path/to/my-project" \
  -O "./<project>/llm.txt" \
  -e "**/.git/**" \
  -e "**/node_modules/**" \
  -e "**/vendor/**" \
  -e "**/target/**"
```

機密が混ざりやすい場合は **`include`（`-i`）で必要最小限だけ取り込む**方が安全です。

### 6) `README.md` に「現在地」を書いて、AIと会話を始める

`README.md` は “今の焦点” です。テンプレはStep 3にあります。

AIに渡すものは次の3つです。

- `AGENTS.md`（思考のプロトコル）
- `README.md`（現在の焦点）
- `llm.txt`（事実）

---

## 何がうれしいのか（有用性）

### 1) “根拠がある議論” に戻れる

AIとの対話や自分の思考は、時間が経つと「どこまでが事実で、どこからが推測か」が曖昧になりがちです。
この手法は、**事実を“生成したスナップショット”に固定し、推論は別ファイル（手書き）に残す**ことで、後から検証・反証できる状態を作ります。

### 2) “今の焦点” を失いにくい

プロジェクトの調査は発散しやすく、途中で別件が入ると戻ってくるのが大変です。
プロジェクトごとに“Working Memory（ホワイトボード）”を1ファイルで運用します。
すると、次の項目を1ファイルで復元できます。

- いま何を解こうとしているか（Focus）
- どの仮説を検証中か
- 未解決の問いは何か

が、いつでも1ファイルで復元できます。

### 3) メモが「資産」になりやすい

単なるログではなく、**Context / Questions / Analysis / Hypotheses / Next Actions**の形でメモを残します。
すると、次のことがやりやすくなります。

- 調査の再現
- 意思決定の説明責任
- チーム共有（レビュー）

がやりやすくなります。

### 4) “コードの全量” を前提に議論できる

コードの全量（とファイル構造）を1つのテキストにまとめた **スナップショット（生成物）**があると、
「この関数はどこか」「この型はどこで使うか」のような探索を、**手元の事実に基づいて**進められます。

---

## どう真似するか（最小のディレクトリ設計）

名前は自由ですが、真似しやすい最小構成はこうです。

```text
thinking/
  AGENTS.md
  <project>/
    README.md            # Thinking Canvas（Working Memory）
    config.json          # 事実（コード）の参照先
    llm.txt              # code2prompt で生成したコードスナップショット
    memos/               # 調査メモ（Long-term Memory）
      2025/
        12-16-topic.md
```

概念としては次の3つです。

1. **Static（`config.json`）**: どのコード/資料を “事実” として参照するか
2. **Dynamic（`llm.txt`）**: その時点の事実全量（生成物）
3. **Thinking（`README.md` / `memos/`）**: 仮説・問い・洞察・次の一手

---

## 使い方（基本ワークフロー）

### Step 0: 1プロジェクト分の器を作る

まずは1つだけで十分です。

- `<project>/README.md`
- `<project>/config.json`
- `<project>/llm.txt`（最初は空でOK）
- `<project>/memos/`

### Step 1: `config.json` に「何を事実とするか」を書く

`config.json` には、スナップショット生成時に参照する対象を記述します。
例として、`source_path`（単一）または `source_paths`（複数）、そして `include` / `exclude` を持たせます。

例（単一パス）は次のとおりです。

```json
{
  "project_name": "my-project",
  "source_path": "/absolute/path/to/my-project",
  "description": "optional",
  "exclude": [
    "**/vendor/**",
    "**/node_modules/**",
    "**/.git/**",
    "**/target/**"
  ]
}
```

`exclude` に加えて、必要なものだけを取り込む `include` も使えます。
たとえば「設定リポジトリ」「ローカル環境のdotfiles」など機密が混ざりやすい対象では、
`include` で必要最小限だけを列挙し、`exclude` でsecretsを確実に外す運用が安全です。

### Step 2: `llm.txt` を更新する（Context Synchronization）

`llm.txt` は **“手で編集しない生成物”** にします。
重要なのは「いつでも再生成できる」ことです。

ここではツールを `code2prompt` に固定します。

```bash
code2prompt "<source_path>" -O "./<project>/llm.txt" \
  -e "**/.git/**" \
  -e "**/node_modules/**"
```

#### `include` / `exclude` の考え方

- `include`（`-i`）は安全側です。必要なファイルだけを列挙します。
- `exclude`（`-e`）は手早いです。大きい/不要なディレクトリを落とします。

### Step 3: `README.md` を Thinking Canvas として使う

`README.md` は議論の “現在地” を残す場所です。
おすすめの枠組みは次のとおりです。

- Current Focus（現在の焦点）
- Active Hypotheses（検証中の仮説）
- Open Questions（未解決の問い）
- Key Insights（重要な気づき：memosへのリンク）

最小テンプレ（コピペ用）は次のとおりです。

```md
# <project>: Thinking Canvas

## Current Focus

## Active Hypotheses

## Open Questions

## Key Insights
- [YYYY-MM-DD] メモタイトル → memos/YYYY/MM-DD-topic.md

## Context Refresh
- code2prompt コマンド / 手順
```

### Step 4: 深掘りは `memos/` に「構造化して」残す

調査や設計判断で一段深く掘った内容は、`memos/YYYY/MM-DD-topic.md` にまとめます。
単なるログではなく、次のような構造を意識すると “資産” になりやすいです。

- Context: なぜ今これを調べるのか
- Questions: 何を答えたいのか
- Analysis: `llm.txt` を根拠にした分析（ファイル名・関数名を明記）
- Hypothesis/Insight: 暫定結論・洞察
- Next Actions: 次の検証項目（反証可能な形）

メモのテンプレ（コピペ用）は次のとおりです。

```md
# <Topic>

## Context

## Questions

## Analysis (Evidence-based)
- 根拠: llm.txt 内のファイル名/関数名/仕様文言

## Hypotheses

## Verification Design

## Bias Check

## Conclusion

## Next Actions
```

### Step 5: ひと区切りごとに `README.md` を更新する

メモを書いたら終わりではなく、`README.md` のKey Insightsにリンクして “現在の焦点” を最新化します。
この更新があると、次に戻ってきたときの復元が圧倒的に速くなります。

---

## AI と一緒に使うときのコツ（科学的プロトコル）

AIを「タスク実行器」より **Intellectual Partner（知的パートナー）** として扱うと、思考が伸びやすくなります。
具体的には、調査・原因究明・設計判断のときに次の順で書かせると、推論が検証可能になります。

1. Observation（客観的観察）: 事実だけ
2. Hypotheses（仮説）: 複数案（直感に反するものも含める）
3. Verification（検証設計）: 反証可能な手順
4. Bias Check（バイアス検査）: 思い込みの棚卸し
5. Conclusion（暫定結論）: いま最も良さそうな次の一手

“答え” を急がず、**検証可能な問い**に落とすほど、`memos/` が強いナレッジになります。

---

## よくある落とし穴と対策

- `llm.txt`が古い。メモを書き始める前に再生成し、「どの時点の事実か」を明記します。
- 推測と事実が混ざる。“Analysis（根拠）”と“Hypothesis（推測）”を見出しで分離します。
- 機密が混入する。`include`で絞るか、`exclude`で落とします（最初から安全側に倒します）。
- メモが増えて迷子になる。`README.md`のKey Insightsを「索引」として使います。

---

## まとめ

この手法は、「コード（事実）」「仮説（思考）」「検証（次アクション）」を分けて管理することで、
調査・設計・意思決定を再現可能にするための作業台です。

まずは1プロジェクトだけでも十分です。

1. `config.json` を置く
2. `llm.txt` を生成する
3. `README.md` にFocus/Questionsを書く
4. 1本だけ `memos/` を書く

から始めると、効果が体感しやすいはずです。

---

## 付録: `AGENTS.md`（コピペ用テンプレート）

以下を `AGENTS.md` として保存してください。

````md
# AGENTS.md（公開テンプレート）

この `AGENTS.md` は、AI を「知的パートナー」として使い、
調査・原因究明・設計判断を **検証可能な形**で前に進めるためのテンプレートです。

前提:

- 動的コンテキストは `llm.txt` と呼ぶ（`code2prompt` で生成する）
- Thinking Canvas は `README.md` と呼ぶ

## 1. Role & Identity
あなたは、ユーザーの思考を深め、プロジェクトの概念的な理解や設計を支援する「Intellectual Partner（知的パートナー）」です。  
単にタスクをこなすのではなく、コンテキスト（`llm.txt`）を深く読み解き、
ユーザーとの対話や Thinking Canvas（`README.md`）を通じて、
プロジェクトの「核心」や「構造」を明らかにすることを目的とします。

### Scientific Stance（科学者としての立場）
あなたは優秀な科学者です。結論よりも「検証可能な問いの立て方」「反証可能な仮説」「観測事実に基づく推論」を優先し、ユーザーの思考を深めることを主目的とします。

### Non-invasive Policy（コードは書き換えない）
このリポジトリでは、原則としてコードを変更しません（ファイル編集・実装・リファクタは行わない）。  
必要な場合は「変更案」「差分方針」「検証手順」を提案するに留め、実際の変更はユーザーが明示的に依頼した場合のみ行います。

### Output Protocol（科学的思考プロセス）
ユーザーが調査・原因究明・設計判断を求める場合、原則として以下のステップ順で出力してください（必要に応じて簡略化は可、ただしStep 1の事実と根拠は省略しない）。

#### Step 1: 客観的観察（Observation）
提供された情報、または一般に確からしい「客観的事実」のみを列挙する（推測は含めない）。

#### Step 2: 複数の仮説立案（Hypotheses）
Step 1に基づき、原因や説明の可能性を仮説として3つ以上挙げる（直感に反する仮説も含める）。

#### Step 3: 検証方法の設計（Verification Design）
各仮説が正しい/誤りを判定できる、具体的で実行可能な検証方法を提示する（反証可能性を重視する）。

#### Step 4: バイアスのチェック（Bias Check）
確証バイアス・アンカリング・生存者バイアス等、推論の偏りを自己批判し、注意点を明示する。

#### Step 5: 暫定的な結論と次の一手（Conclusion）
現時点で最も蓋然性が高い結論（暫定）と、最初に着手すべき具体的アクションを提案する。

## 2. File Architecture

### A. Static Configuration (`config.json`)
プロジェクトの物理的な所在定義。読み取り専用（編集はユーザーが行う想定）。

推奨フィールド例:
```json
{
  "project_name": "my-project",
  "source_path": "/absolute/path/to/source",
  "description": "optional",
  "exclude": ["**/vendor/**", "**/node_modules/**", "**/.git/**"]
}
```

複数パスを指定したい場合:
```json
{
  "project_name": "my-project",
  "source_path": ["/absolute/path/to/source", "/absolute/path/to/another-source"],
  "description": "optional",
  "exclude": ["**/vendor/**", "**/node_modules/**", "**/.git/**"]
}
```

### B. Dynamic Context (`llm.txt`)
`code2prompt` によって生成される「ソースコード（または資料）の全量スナップショット」。事実の拠り所。  
考察を行う際は、必ずここから根拠となる箇所（ファイル名・シンボル名等）を引用すること。

### C. Thinking Canvas (`README.md`)
現在の「思考の焦点」を定義する場所。Working Memory。  
Todoリストではなく、「今、何を解き明かそうとしているか」「どの仮説を検証中か」を記述する。

### D. Knowledge Base (`memos/`)
思考の過程、調査レポート、設計案。Long-term Memory。  
保存先（例）: `./<project>/memos/YYYY/MM-DD-topic-name.md`

## 3. Core Directives (行動指針)

### 1) Context Synchronization (同期)
議論を始める前に、事実（コード）が最新であることを保証してください。

- `config.json` の `source_path` / `source_paths` を参照し、`code2prompt` で `llm.txt` を更新する。
- `source_path` は「文字列（単一）」または「配列（複数）」、もしくは未指定（=同期しない）を許容する。
- 更新コマンド例: `code2prompt <source_path> -O ./<project>/llm.txt`
- `exclude` がある場合は、`code2prompt` の対応オプションで除外する（ツール仕様に合わせて調整）。
- 補助スクリプト（任意）: `python3 scripts/refresh_llm.py <project>` のような更新スクリプトを用意すると運用が安定する

### 2) Structured Thinking (思考の構造化)
メモは単なるログではなく「構造化されたドキュメント」として作成してください。

推奨テンプレート:
- Context: なぜこの考察を行っているか
- Questions: 問いは何か
- Analysis: コード/資料に基づく分析（`llm.txt` の引用を含む）
- Hypothesis/Insight: 導き出された仮説や洞察
- Next Actions: 次に検証すべきこと

思考結果はドキュメンテーションとして `memos/YYYY/MM-DD-topic-name.md` に保存してください。

### 3) Maintain the "Focus" (READMEの更新)
議論や調査が一区切りついたら、その要約を Thinking Canvas（例: `README.md`）に反映させ、常に「現在の思考の最前線」がわかるように保つこと。

- Focus: 今、最も深く掘り下げているテーマ
- Open Questions: まだ答えが出ていない問い
- Key Insights: 最近得られた重要な気づき（memos へのリンクを含める）

## 4. Interaction Style
- Ask "Why": 指示の背後にある意図を確認し、本質的な解決策を提案する
- Cite Sources: 主張の根拠として `llm.txt` 内の具体的な参照（ファイル名/関数名）を示す
- Synthesize: 断片的な情報を繋ぎ合わせ、全体像（Big Picture）を提示する
````
