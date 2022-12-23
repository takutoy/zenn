---
title: "Semgrep ルール作成チュートリアル (DOM-Based XSS)"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["security", "semgrep", "sast"]
published: false
---

Semgrep は標準で多数のルールを使うことができますが、ルールをカスタマイズしたり新しく作ったりしたくなる時があります。

例えば自社開発プロダクトである脆弱性が見つかった時、
- 同様の脆弱性が他のプロダクトにもないか調べたい
- 今後同様の脆弱性が作りこまれたら早く検知したい

このようなとき、脆弱性を発見するルールを作成できるとプロダクトのセキュリティを維持する助けになります。

この記事では DOM-Based XSS を検出するルール作成を通して、ルールの作成に必要な機能やオプションを理解します。

## 類似ルールを探す

Semgrep Registry には Semgrep の製作者である r2c やコミュニティが作成したルールがたくさんあります。

自分が作成しようとしているルールが既に作られていたり、似たルールがあるかもしれないのでまずは検索しましょう。

今回は DOM-Based XSS を検出するルールを作りたいので [javascript を dom xss で検索](https://semgrep.dev/r?q=dom+xss&lang=JavaScript) してみます。

検索の結果 `javascript.browser.security.dom-based-xss.dom-based-xss` というルールが見つかったので、これをベースに作成していきます。

```yaml : dom-based-xss.yaml のパターン部
pattern-either:
  - pattern: document.write(<... document.location.$W ...>)
  - pattern: document.write(<... location.$W ...>)
```

`document.write()` を検出するだけのシンプルなルールのようです。

## テストケースを考える

ルールを書き始める前に、まずはテストケースを作成しておくとよいです。

- 検出したいパターン
- 検出したくないパターン

をそれぞれいくつか用意しておきましょう。

```js : dom-based-xss.js
const qs = window.location.search;
const hash = window.location.hash;

// ok
document.write("<p>ok</p>");

// ng
document.write(qs);
document.write(hash);
```

テストケースは最初から全パターンを網羅する必要はなく、ルールを作りながらテストケースも追加していくと良いです。

ちなみに、今のルールで実行してもまだ正しく検出できません。

```
$ semgrep --config dom-based-xss.yaml dom-based-xss.js
Scanning 1 file.

Ran 1 rule on 1 file: 0 findings.
```

## Taint tracking

XSS のようなインジェクション攻撃には source と sink という考え方があります。
攻撃コードを仕込む箇所を source, 攻撃コードが実行される個所を sink と言います。

Semgrep には、信頼できない source が脆弱な関数 sink に到達するかを分析する、Taint tracking という機能があります。
Taint tracking を使うことで、見逃しや誤検知を減らすことができることがあります。

### Taint mode

Taint tacking を使用するには、mode を taint に設定し、`pattan-sources` と `pattern-sinks` を書きます。

```yaml : dom-based-xss.yaml
rules:
- id: dom-based-xss
  mode: taint
  message: dom-xss
  languages:
  - javascript
  - typescript
  severity: ERROR
  pattern-sources:
  - pattern: window.location
  pattern-sinks:
  - pattern-either:
    - pattern: document.write(...)
```

- `mode: taint`
- `pattan-sources` に DOM-XSS の source である `window.location`
- `pattan-sinks` に DOM-XSS の sink である `document.write(...)`

をそれぞれ設定しています。

このように設定した場合、taint tracking では下記のように分析されます。

- `const qs = window.location.search;`
  - `window.location` は汚染されている
  - `window.location.search` も汚染されている
  - 定数 `qs` も汚染されている
- `document.write(qs);`
  - 汚染された `qs` が sink で実行された 

このルールをさきほどのテストケースに実行してみると、検出できるようになったことが確認できます。

```bash
$ semgrep --config dom-based-xss.yaml dom-based-xss.js 
Scanning 1 file.

Findings:

  dom-based-xss2.js 
     dom-based-xss  
        dom-xss     

          5┆ document.write(qs);
          ⋮┆----------------------------------------
          6┆ document.write(hash);


Ran 1 rule on 1 file: 2 findings
```

### Source と Sink を充実させる

DOM-XSS の source には `window.location` 以外にもありますし、sink には `document.write()` 以外にもあります。

たとえば 
[portswigger の blog](https://portswigger.net/blog/introducing-dom-invader#:~:text=in%20the%20sink.-,List%20of%20sources%20and%20sinks,-Whilst%20developing%20DOM) では、11 の source と 86 の sink を紹介しています。
全ての source と sink をに対応すると記事が長くなるので、それぞれ5つずつ選びました。

主な source
```
- location
- location.href
- location.hash
- location.search
- document.URL
```

主な sink
```
- document.write()
- document.writeln()
- jQuery.html()
- element.innerHTML
- location.href
```

これらの source と sink をルールに書くとこうなります。

```yaml : dom-based-xss.yaml
rules:
- id: dom-based-xss
  mode: taint
  message: dom-xss
  languages:
  - javascript
  - typescript
  severity: ERROR
  pattern-sources:
  - pattern-either:
    - pattern: location
    - pattern: window.location
    - pattern: document.location
    - pattern: document.URL
  pattern-sinks:
  - pattern-either:
    - pattern: document.write($PAYLOAD)
    - pattern: document.writeln($PAYLOAD)
    - pattern: $JQ.html($PAYLOAD)
    - pattern: $ELEMENT.innerHTML = $PAYLOAD
    - pattern: location.href = $PAYLOAD
```

注意点
- `location` を設定すれば、`location.href` `location.hash` `location.search` も自動的に source となる
- `location` には、`window.location` と `document.location` があるため追加している

source と sink の追加に合わせて、テストケースを変更します。

```js : dom-based-xss.js
const qs = window.location.search;
const hash = document.location.hash;
const query = location.search;
const url = document.URL;

// ok
document.write("<p>ok</p>");

// ng
document.write("ng" + qs);
document.writeln("ng" + hash);

// ng
$("div.test").html(query)

// ng
const e1 = document.createElement('p');
e1.innerHTML = url;

// ng
location.href = qs
```

テストケースを追加したらルールを実行しましょう。5つのNGケースがあるので、5つ検出されるはずです。

```bash
$ semgrep --config dom-based-xss.yaml dom-based-xss.js
Scanning 1 file.

Findings:

  dom-based-xss.js
     dom-based-xss
        dom-xss

         10┆ document.write("ng" + qs);
          ⋮┆----------------------------------------
         11┆ document.writeln("ng" + hash);
          ⋮┆----------------------------------------
         14┆ $("div.test").html(query)
          ⋮┆----------------------------------------
         18┆ e1.innerHTML = url;
          ⋮┆----------------------------------------
         21┆ location.href = qs


Ran 1 rule on 1 file: 5 findings.
```

ちゃんと検出されました。

### Propagator

taint tracking では一部の関数が使われた際、追跡が途切れてしまうことがあります。

例えば次のようなケースはDOM-XSSが発生しますが、Semgrepでは検出されません。

```js
// ng
arr = [];
arr.push(url);
document.write(arr.join(' '));
```

これは Semgrep は `arr` が `url` により汚染されることを知らないためです。そこで `propagators` の出番です。`propagators` は次のように設定します。

```yaml
pattern-propagators:
- pattern: $ARR.push($E)
  from: $E
  to: $ARR
```

これにより先ほどのテストケースも検出されるようになります。なお、`push` 以外にも、`shift` や `unshift`, `concat` なども同様に設定が必要です。

### Sanitizer

汚染された変数が適切にサニタイズされている場合は DOM-XSS は発生しません。

たとえば DOMPurify を使ってサニタイズしている場合は DOM-XSS は発生しませんが、検出されてしまいます。

```js
// ok
const sanitized = DOMPurify.sanitize(qs)
document.write(sanitized);
```

そこで `sanitizers` を設定すると、変数がサニタイズされたとして追跡を中断させることができます。

```yaml
pattern-sanitizers:
- pattern: DOMPurify.sanitize(...)
```

### taint tracking まとめ

ここまでで taint mode を使ってDOM-Based XSSを検出するルールを作成しました。

taint mode では次の設定を使用しました。

- mode: taint
- pattern-sources
- pattern-sinks
- pattern-propagators
- pattern-sanitizers

完成したYAMLファイルとテストコードは次の通りです。

```yaml : dom-based-xss.yaml
rules:
- id: dom-based-xss
  mode: taint
  message: dom-xss
  languages:
  - javascript
  - typescript
  severity: ERROR
  pattern-sources:
  - pattern-either:
    - pattern: location
    - pattern: window.location
    - pattern: document.location
    - pattern: document.URL
  pattern-sinks:
  - pattern-either:
    - pattern: document.write($PAYLOAD)
    - pattern: document.writeln($PAYLOAD)
    - pattern: $JQ.html($PAYLOAD)
    - pattern: $ELEMENT.innerHTML = $PAYLOAD
    - pattern: location.href = $PAYLOAD
  pattern-propagators:
  - pattern: $ARR.push($E)
    from: $E
    to: $ARR
  pattern-sanitizers:
  - pattern: DOMPurify.sanitize(...)
```

```js : dom-based-xss.js
const qs = window.location.search;
const hash = document.location.hash;
const query = location.search;
const url = document.URL;

// ok
document.write("<p>ok</p>");

// ng
document.write("ng" + qs);
document.writeln("ng" + hash);

// ng
$("div.test").html(query);

// ng
const e1 = document.createElement('p');
e1.innerHTML = url;

// ng
location.href = qs;

// ng
arr = [];
arr.push(url);
document.write(arr.join(' '));

// ok
const sanitized = DOMPurify.sanitize(qs)
document.write(sanitized);
```

実行結果

```bash
$ semgrep --config dom-based-xss.yaml dom-based-xss.js
Scanning 1 file.

Findings:

  dom-based-xss.js
     dom-based-xss
        dom-xss

         10┆ document.write("ng" + qs);
          ⋮┆----------------------------------------
         11┆ document.writeln("ng" + hash);
          ⋮┆----------------------------------------
         14┆ $("div.test").html(query);
          ⋮┆----------------------------------------
         18┆ e1.innerHTML = url;
          ⋮┆----------------------------------------
         21┆ location.href = qs;
          ⋮┆----------------------------------------
         26┆ document.write(arr.join(' '));


Ran 1 rule on 1 file: 6 findings.
```

## 他の言語にうめこまれた javascript を抽出

Semgrep の既定では、HTML に埋め込まれた javascript はスキャンされません。

次のテストコードを考えます。

```html : dom-based-xss.html
<html>
    <body>
        <script>
const qs = window.location.search;
const hash = document.location.hash;

// ok
document.write("<p>ok</p>");

// ng
document.write("ng" + qs);
document.writeln("ng" + hash);
        </script>
    </body>
</html>
```

このテストコードに先ほど作成したルールを実行してみます。

```bash
$ semgrep --config dom-based-xss.yaml dom-based-xss.html
Nothing to scan.

Ran 1 rule on 0 files: 0 findings.
```

検出できませんでした。

このような、言語の中に埋め込まれている別の言語を検出したいは `extract` モードを使う必要があります

### HTML から抽出する

HTML から javascript を抽出するルールは次のようになります。

```yaml : extract-html-to-javascript.yaml
rules:
- id: extract-html-to-javascript
  mode: extract
  languages:
    - html
  pattern: <script>$...SCRIPT</script>
  extract: $...SCRIPT
  dest-language: javascript
```

extract モードを使うには、下記5つ設定が必要です。

- mode: extract
- languages
- pattern
- extract
- dest-language 

このルールを使うことで、HTML内のjavascriptを検知することができるようになります。

```bash
$ semgrep --config dom-based-xss.yaml --config extract-html-to-javascript.yaml dom-based-xss.html
Scanning 1 file.

Findings:

  dom-based-xss.html
     dom-based-xss
        dom-xss

         11┆ document.write("ng" + qs);
          ⋮┆----------------------------------------
         12┆ document.writeln("ng" + hash);

Ran 2 rules on 1 file: 2 findings.
```

※注意、extract ルールは、通常のルールの「後」に設定する必要があります。extractルールを「前」に設定してしまうと検出できません。

```bash
$ semgrep --config extract-html-to-javascript.yaml --config dom-based-xss.yaml dom-based-x
ss.html
Scanning 1 file.

Ran 2 rules on 1 file: 0 findings.
```

### ERB から抽出する

ついでに Ruby on Rails で使われる ERB から抽出するルールも紹介します。

```yaml : extract-erb-to-javascript.yaml
rules:
- id: extract-erb-to-javascript
  mode: extract
  languages:
    - generic
  options:
    generic_ellipsis_max_span: 500
  pattern: ...<script>$...SCRIPT</script>
  extract: $...SCRIPT
  dest-language: javascript
  paths:
    include:
      - "*.erb"
```

注意点は2つあります

1点目：
ERBは対応 langage でないため `generic` を使用し、拡張子 `.erb` のファイルを対象としています。

2点目：
generic では既定で抽出したテキストの11行目以降は省略されます。そのため、`<script>` タグの本文が10行を超える場合、正しく抽出できません。そこでオプション`generic_ellipsis_max_span` を設定し、100行まで抽出できるようにしています。(値はパフォーマンスに影響するので調整してください)

## まとめ

Semgrep で DOM-Based XSS を検出するルールを作成する過程を通して、下記の機能を紹介しました。

- taint mode
  - source
  - sink
  - propagator
  - sanitizer
- extract mode
  - pattern
  - extract
  - dest-language
- option
  - generic_ellipsis_max_span

自作ルールを作るときの参考にしてもらえれば！

最終的に出来たもの
https://github.com/takutoy/my-semgrep-rules/tree/master/javascript/browser/security

試行錯誤の記録
https://zenn.dev/takutoy/scraps/6c0f9c20bf1d86