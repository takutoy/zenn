---
title: "Semgrepでセキュリティコードレビューを効率化する"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rails", "security", "sast", "semgrep"]
published: true
---

静的アプリケーションセキュリティテスト(SAST)ツールである Semgrep を使って、セキュリティコードレビューを効率化する方法を紹介する。

- Semgrepとは
- Semgreoの使い方
- Semgrepのルールの作り方
- Semgrepの導入

なお本記事ではサンプルコードとして Ruby on Rails のコードを使うが、Semgrep は Ruby だけでなく、C#, Java, Javascript, PHP, Python 等[様々な言語をサポート](https://semgrep.dev/docs/supported-languages/)している。

## Semgrep とは

Semgrep はコードを（文字列ではなく）構文で検索できるツールである。Semgrep = Semantic な Grep だとか。

例えば `User.find(params[:id])` のようなコードを検出することを考える。（ちなみにこれは Ruby on Rails において [Insecure Direct Object Reference (IDOR)](https://portswigger.net/web-security/access-control/idor) の可能性があるコードである。）

下記コードにおいて a, b, c は全て等価であるが、全てをマッチさせる正規表現パターンを作ることは非常に困難である。

```ruby : find1.rb
# a
User.find(params[:id])

# b
User.find params[:id]

# c
User
  .find(params[:id])
```

対して Semgrep ではコードの構文で検索できるため、a, b, c 全てにマッチさせるパターンを容易に作ることができる。

## Semgrepの使いかた

### インストール

Semgrep のインストールは [Getting started with Semgrep CLI](https://semgrep.dev/docs/getting-started/#running-semgrep-locally) を参照。

pip を使う場合は次のコマンドでインストールできる。

```bash
python3 -m pip install semgrep
```

動作確認

```bash
$ semgrep --version
```

次ようにバージョンが表示されたらインストールは成功している。

```
1.2.1
```

### CLIの基本

もっとも基本的なコマンドは下記である。

```bash
semgrep -e '{パターン}' -l {言語} {ファイル名} 
```

次のコードを `find1.rb` に保存し、Semgrep で検索してみよう。

```ruby : find1.rb
# a
User.find(params[:id])

# b
User.find params[:id]

# c
User
  .find(params[:id])
```

コマンドは次のとおり。

```bash
$ semgrep -e '$M.find(params[:id])' -l ruby find1.rb
```

すると a, b, c の3つとも検出できたことが確認できる。

```
Scanning 1 file.

Findings:

  find1.rb
          2┆ User.find(params[:id])
          ⋮┆----------------------------------------
          5┆ User.find params[:id]
          ⋮┆----------------------------------------
          8┆ User
          9┆   .find(params[:id])


Ran 1 rule on 1 file: 3 findings.
```

このように、文字列としては異なっていても、構文的に等価であれば簡単にマッチさせることができるのが Semgrep の特徴だ。

### ルールをYAMLに定義する

パターンを毎回コマンドで叩くのはめんどくさい。SemgrepではパターンをYAMLファイルで定義することができる。

先ほどのパターンをYAMLで定義するとこのようになる。`idor1.yaml` に保存しよう。

```yaml : idor1.yaml
rules:
  - id: idor
    languages:
      - ruby
    pattern: $M.find(params[:id])
    message: check authorization
    severity: WARNING
```

YAMLファイルを使って検索するコマンドは次の通り。

```bash
$ semgrep --config idor1.yaml find1.rb
```

コマンド `semgrep -e '$M.find(params[:id])' -l ruby find1.rb` と同様の結果が得られる。

```
Scanning 1 file.

Findings:

  find1.rb
     idor
        check authorization

          2┆ User.find(params[:id])
          ⋮┆----------------------------------------
          5┆ User.find params[:id]
          ⋮┆----------------------------------------
          8┆ User
          9┆   .find(params[:id])


Ran 1 rule on 1 file: 3 findings.
```

## ルールの作り方

実は先ほど作成したルールは使い物にならない。次のコードを考えてみよう。(後で使うので `find2.rb` として保存しておこう)

```ruby : find2.rb
# a : NG
Article.find(params[:id])

# b : NG
Article.find params[:id]

# c : NG
Article
  .find(params[:id])

# d : NG
id = params[:id]
Article.find(id)

# e : NG
id = params[:id].to_i
Article.find(id)

# f : OK
current_user.articles.find(params[:id])
```

IDORの脆弱性を発見したいとき、 a-e は検出したいが、f は検出したくない。

`$M.find(params[:id])` というパターンで検索するとどうなるだろう。

```bash
$ semgrep -e '$M.find(params[:id])' -l ruby find2.rb
```

```
Scanning 1 file.

Findings:

  find2.rb
          2┆ Article.find(params[:id])
          ⋮┆----------------------------------------
          5┆ Article.find params[:id]
          ⋮┆----------------------------------------
          8┆ Article
          9┆   .find(params[:id])
          ⋮┆----------------------------------------
         20┆ current_user.articles.find(params[:id])


Ran 1 rule on 1 file: 4 findings.
```

a, b, c は検知するものの、d, f を見逃し、e を誤検知してしまう。

|#|コード|想定する結果|実際の結果|OK/NG|
|:--|:--|:--|:--|:--|
|a|`Article.find(params[:id])`|検出される|検出される|OK|
|b|`Article.find params[:id]`|検出される|検出される|OK|
|c|`Article`<br>`  .find(params[:id])`|検出される|検出される|OK|
|d|`id = params[:id]`<br>`Article.find(id)`|検出される|検出されない|NG|
|e|`id = params[:id].to_i`<br>`Article.find(id)`|検出される|検出されない|NG|
|f|`current_user.articles.find(params[:id])`|検出されない|検出される|NG|

見逃しと誤検知は、どちらも自動化・効率化の敵である。どうにかしたい。

### 誤検知をどうにかする

まずは誤検知である f をどうにかしよう。

```ruby
# f : OK
current_user.articles.find(params[:id])
```

誤検知を減らすには、パターンを厳密にすればよい。Semgrep の metavariable という仕組みを使うと、パターンに制約を加えることができる。

先ほど特に説明なくパターン `$M.find(params[:id])` と記載したが、実はこの `$M` が metavariable だ。

下記のルールでは、metavariable `$M` に正規表現 `^[A-Z]` （大文字アルファベットで始まる） にマッチするという制約を追加している。

```yaml : idor2.yaml
rules:
 - id: idor2
   languages:
     - ruby
   patterns:
     - pattern: $M.find(params[:id])
     - metavariable-regex:
         metavariable: $M
         regex: ^[A-Z]
   message: check authorization
   severity: WARNING
```

このルールを `idor2.yaml` に保存して、次のコマンドを実行してみよう。

```bash
$ semgrep --config idor2.yaml find2.rb
```

```
Scanning 1 file.

Findings:

  find2.rb
     idor2
        check authorization

          2┆ Article.find(params[:id])
          ⋮┆----------------------------------------
          5┆ Article.find params[:id]
          ⋮┆----------------------------------------
          8┆ Article
          9┆   .find(params[:id])


Ran 1 rule on 1 file: 3 findings.
```

すると、大文字で始まっていない f のコードが検出されなくなったことが確認できる。

|#|コード|想定する結果|実際の結果|OK/NG|
|:--|:--|:--|:--|:--|
|a|`Article.find(params[:id])`|検出される|検出される|OK|
|b|`Article.find params[:id]`|検出される|検出される|OK|
|c|`Article`<br>`  .find(params[:id])`|検出される|検出される|OK|
|d|`id = params[:id]`<br>`Article.find(id)`|検出される|検出されない|NG|
|e|`id = params[:id].to_i`<br>`Article.find(id)`|検出される|検出されない|NG|
|f|`current_user.articles.find(params[:id])`|検出されない|検出されない|OK|

なお、semgrep では metavariable 以外にも様々なパターンが利用できる、詳細は [Pattern syntax](https://semgrep.dev/docs/writing-rules/pattern-syntax/) を参照。

### 見逃しをどうにかする

次は見逃していた d と e を検出できるようにする。見逃しに対応するには大きく2つの方法がある。

1. パターンを緩くする
2. データフロー分析を使う

パターンを緩くするとは、例えばパターンを `$M.find(...)` （※ `...` は任意のパラメータにマッチする）のように、より多くのコードにマッチするように変更すれば見逃しを減らすことはできる。ただしこれでは誤検知も増やしてしまいかねない。

なので、ここでは2つめ方法であるデータフロー分析を使おう。Semgrep のデータフロー分析には `Constant propagation` と `Taint tracking` という2つの手法がある。

それぞれの方法を用いて d と e のコードを検出できるようにする。

```ruby
# d : NG
id = params[:id]
Article.find(id)

# e : NG
id = params[:id].to_i
Article.find(id)`
```

### Symbolic propagation

Symbolic propagation（変数伝播？）は、定数や変数を追跡できるデータフロー分析だ。

Symbolic propagation を使うには、 `symbolic_propagation` オプションを有効にすればよい。

```yaml : idor3.yaml
rules:
 - id: idor3
   languages:
     - ruby
   options:
     symbolic_propagation: true
   patterns:
     - pattern: $M.find(params[:id])
     - metavariable-regex:
         metavariable: $M
         regex: ^[A-Z]
   message: check authorization
   severity: WARNING
```

Symbolic propagation により Semgrep は `id = params[:id]` から `Article.find(id)` が `Article.find(params[:id])` と等価であると判断するようになる。

上記のルールを `idor3.yaml` に保存して次のコマンドを実行してみよう。

```bash
$ semgrep --config idor3.yaml find2.rb
```

```
Scanning 1 file.

Findings:

  find2.rb
     idor3
        check authorization

          2┆ Article.find(params[:id])
          ⋮┆----------------------------------------
          5┆ Article.find params[:id]
          ⋮┆----------------------------------------
          8┆ Article
          9┆   .find(params[:id])
          ⋮┆----------------------------------------
         13┆ Article.find(id)


Ran 1 rule on 1 file: 4 findings.
```

結果、d も検出されるようになったことが確認できる。

|#|コード|想定する結果|実際の結果|OK/NG|
|:--|:--|:--|:--|:--|
|a|`Article.find(params[:id])`|検出される|検出される|OK|
|b|`Article.find params[:id]`|検出される|検出される|OK|
|c|`Article`<br>`  .find(params[:id])`|検出される|検出される|OK|
|d|`id = params[:id]`<br>`Article.find(id)`|検出される|検出される|OK|
|e|`id = params[:id].to_i`<br>`Article.find(id)`|検出される|検出されない|NG|
|f|`current_user.articles.find(params[:id])`|検出されない|検出されない|OK|

### Taint tracking

Taint tracking（汚染追跡？）では、信頼できないデータ(source)が脆弱な関数(sink) に到達するか、という分析ができる。

Taint tracking を使うには taint モードを使用し、source と sink のパターンを作成する。

今回の例では `params[:id]` が source, `$M.find(...)` が sink となる。

```yaml : idor4.yaml
rules:
  - id: idor
    languages:
      - ruby
    options:
      symbolic_propagation: true
    mode: taint
    pattern-sources:
    - pattern: params[:id]
    pattern-sinks:
    - patterns:
      - pattern: $M.find(...)
      - metavariable-regex:
          metavariable: $M
          regex: ^[A-Z]
    message: check authorization
    severity: WARNING
```

このルールでは次のようにコードが分析される。

1. `params[:id]` (source) は汚染されている
2. `params[:id].to_i` も当然汚染されている
3. `id = params[:id].to_i` により `id` も汚染された
4. 汚染された `id` が `$M.find()` (sink) で使用された ⇒ アウト！

上記のルールを `idor4.yaml` に保存して次のコマンドを実行してみよう。

```bash
$ semgrep --config idor4.yaml find2.rb
```

```
Scanning 1 file.

Findings:

  find2.rb
     idor
        check authorization

          2┆ Article.find(params[:id])
          ⋮┆----------------------------------------
          5┆ Article.find params[:id]
          ⋮┆----------------------------------------
          8┆ Article
          9┆   .find(params[:id])
          ⋮┆----------------------------------------
         13┆ Article.find(id)
          ⋮┆----------------------------------------
         17┆ Article.find(id)


Ran 1 rule on 1 file: 5 findings.
```

これでようやくすべてのコードを正しく検出することができるようになった。

|#|コード|想定する結果|実際の結果|OK/NG|
|:--|:--|:--|:--|:--|
|a|`Article.find(params[:id])`|検出される|検出される|OK|
|b|`Article.find params[:id]`|検出される|検出される|OK|
|c|`Article`<br>`  .find(params[:id])`|検出される|検出される|OK|
|d|`id = params[:id]`<br>`Article.find(id)`|検出される|検出される|OK|
|e|`id = params[:id].to_i`<br>`Article.find(id)`|検出される|検出される|OK|
|f|`current_user.articles.find(params[:id])`|検出されない|検出されない|OK|

なお Taint tracking では source と sink の他、sanitizer パターンを指定してサニタイズ関数が使われていたら検出しないようにする等、より高度なルールを作ることもできる。

データフロー分析の詳細は [Data-flow analysis engine overview](https://semgrep.dev/docs/writing-rules/data-flow/data-flow-overview/)、Taint tracking の詳細は [Taint tracking](https://semgrep.dev/docs/writing-rules/data-flow/taint-mode/) を参照。


## Semgrepの導入

ここまでで Semgrep で何が出来るかを説明した。次は実用のためのヒントとして、どのような使い方ができるかを紹介する。

### semgrep-rules

[semgrep-rules](https://github.com/returntocorp/semgrep-rules) には semgrep の製作者である r2c やコミュニティが作成したルールがある。

ルールは自分で1から作ることもできるが、まずはこのリポジトリにあるルールを使ってみて、自分たちの組織に合わせて少しずつカスタマイズしていくのが良いだろう。

ちなみに今回作成した idor.yaml よりも良くできてるルールもある。 ⇒ [check-unscoped-find.yaml](https://github.com/returntocorp/semgrep-rules/blob/develop/ruby/rails/security/brakeman/check-unscoped-find.yaml)

ルールは言語ごとにディレクトリが分かれているので、必要なルールを探すのが面倒な場合はディレクトリをまるごとコピーしてしまってもよい。

### リポジトリ全体をスキャンする

Semgrep は複数のソースコードを複数のルールで検索することもできる。

例として、semgrep-rules を Railsgoat という脆弱性満載のWebアプリケーションに実行させてみよう。

semgrep-rules と Railsgoat は github から入手できる。

```bash
$ git clone https://github.com/returntocorp/semgrep-rules.git
$ git clone https://github.com/OWASP/railsgoat.git
```

実行方法と結果は次の通り。

```bash
$ semgrep --config semgrep-rules/ruby/ railsgoat/

Scanning across multiple languages:
    <multilang> | 18 rules × 264 files
           ruby | 92 rules × 117 files

  100%|███████████████████████████████████████████████████████████████████████████████████████████████████|381/381 tasks

Findings:

  railsgoat/app/controllers/admin_controller.rb
     semgrep-rules.ruby.rails.security.brakeman.check-unscoped-find
        Found an unscoped `find(...)` with user-controllable input.  If the ActiveRecord model being
        searched against is sensitive, this may lead to Insecure Direct Object Reference (IDOR)
        behavior and allow users to read arbitrary records.

         29┆ @user = User.find_by_id(params[:admin_id].to_s)
          ⋮┆----------------------------------------
         35┆ user = User.find_by_id(params[:admin_id])
...
```

Semgrepのルールを育てつつ、プロダクトのリポジトリを定期的にスキャンして脆弱性を発見するといった使い方ができそうだ。

### 分析結果を利用する

Semgrep の実行結果は標準出力だけでなく JSON や SARIF 等でも出力できるので、分析結果を他のツールに統合することもできる。

```bash
$ semgrep --config semgrep-rules/ruby/ railsgoat/ --sarif -o result.json
$ cat result.json
{
  "$schema": "https://docs.oasis-open.org/sarif/sarif/v2.1.0/os/schemas/sarif-schema-2.1.0.json",
  "runs": [
    {
      "invocations": [
...
```

### CIに統合する

ルールが洗練されてきたら Semgrep を CI に組み込むのも良いかもしれない。詳細は [Getting started with Semgrep in continuous integration (CI)](https://semgrep.dev/docs/semgrep-ci/overview/) を参照。

## まとめ

本記事では次のことを紹介した。

- Semgrepとは
- Semgreoの使い方
- Semgrepのルールの作り方
- Semgrepの導入

Semgrep を使えばセキュリティコードレビューを大幅に効率化できる可能性がある。試してみてはいかが？

## 付録

### 類似ツール

Ruby on Rails で使える静的解析ツールをいくつか紹介しておく。

[Rubocop](https://github.com/rubocop/rubocop) は Ruby の静的コード解析＋フォーマッタで、コーディングスタイルの一貫性を保つために使われる、いわゆるlinterである。脆弱なコードを検出するルールもあるにはあるが、あくまでlinterなのでセキュリティを主目的に使うものではない。

[Brakeman](https://brakemanscanner.org/) は Ruby on Rails で使えるSASTツール。Rails用とだけあってそこそこ良い精度だと思う。ただしカスタムルールを作るのは難しそう。

[CodeQL](https://codeql.github.com/) は超高機能なSASTツールだが、基本的にプライベートなプロジェクトには使えない。オープンソースプロジェクトであれば導入候補になりうる。
