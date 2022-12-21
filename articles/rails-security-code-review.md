---
title: "Railsアプリケーションのセキュリティコードレビューガイド"
emoji: "🕵🏻‍♀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rails", "security", "sast"]
published: true
---

※本記事は以前[Qiitaに投稿した記事](https://qiita.com/takutoy/items/e1867b51a6c8a15a93c8)を改変したものです。

## 本ガイドについて

Ruby on Rails 製 Webアプリケーションのセキュリティテストをするためのガイドです。本ガイドは脆弱なRailsアプリである [RailsGoat](https://github.com/OWASP/railsgoat) を題材に、自動および手動によるセキュリティコードレビューの手法を解説します。

### 対象読者

Ruby on Rails でプロダクト開発している組織のセキュリティチームを想定していますが、セキュリティに関心がある開発者やテスターにとっても役立つかもしれません。

### 前提知識・スキル

- 情報セキュリティに関する基本的な用語を知ってること
- 基本的なWebアプリケーションの脆弱性について知ってること
- Ruby on Rails のコードが読めること
- Linux のコマンドを叩けること

なお本ガイドではセキュリティコードレビューを静的アプリケーションセキュリティテスト(SAST)と同じ意味で扱います。

## セキュリティコードレビューの準備

静的アプリケーションセキュリティテストはソースコードをツールで解析したり、目視でレビューしたりすることにより脆弱性を発見します。

静的テストを実行するにあたり、Rails アプリによくあるプログラミングミスを把握しておいたり、コードを読みやすい環境を整えておいたりしておくと効率よくテストを進められます。

### Railsコードレビューの前提知識

Railsアプリの開発者とセキュリティテスターにとって [Securing Rails Applications (Ruby on Rails Guides)](https://guides.rubyonrails.org/security.html) は必読書です。Rails アプリによくあるセキュリティの問題とその対応方法が充実しています。

Securing Rails Applications に基づいてコードレビューするだけでも多くの脆弱性をカバーできると思います。読んだことない人はいますぐ読みましょう！英語版より更新は遅れますが、[日本語版](https://railsguides.jp/security.html)もあります。

### コードレビュー環境

効率的にセキュリティコードレビューするために環境を整えましょう。小規模プロジェクトならソースコードリポジトリをWebブラウザで眺めてもよいですが、ある程度の規模なら高機能なテキストエディタやIDEを使うことお勧めします。

本ガイドでは [Visual Studio Code](https://code.visualstudio.com/) を使います。インストール等についてはリンク先を参照してください。

### Ruby

Ruby を実行する環境がない場合、[Rubyのインストール](https://www.ruby-lang.org/ja/documentation/installation/) を参照してインストールしてください。

### RailsGoat

本ガイドでは脆弱なRailsアプリである [RailsGoat](https://github.com/OWASP/railsgoat) にセキュリティコードレビューを実施します。リンク先からソースコードを入手しておいてください。

## Brakemanを使った自動セキュリティコードレビュー

Rails には [Brakeman](https://brakemanscanner.org/) というソースコードの静的解析ツールがあります。Brakeman は自動かつ高速にソースコードを検査できるため、セキュリティテストでは重宝します。

本節では Brakeman を使ってコードの欠陥を検出し、その結果を目視で精査します。

### Brakeman のインストール

Brakeman は gem でインストールできます。インストールの詳細は[Brakeman Quickstart](https://brakemanscanner.org/docs/quickstart/)を参照してください。

```
gem install brakeman
```

### Brakeman の使い方

Brakeman を使うには、Railsアプリケーションのソースコードがあるディレクトリに移動して `brakeman -A` コマンドを実行します。 `-A` は追加の検査をしてくれるオプションです。付けて困ることはないと思うので、基本的には `-A` をつけておいてよいと思います。

brakeman の実行例：

```shell
user@sectest:~$ cd railsgoat
user@sectest:~/railsgoat$ brakeman -A
Loading scanner...
Processing application in /home/user/railsgoat
Processing gems...

===略===

== Warnings ==

Confidence: High
Category: Cross-Site Scripting
Check: CrossSiteScripting
Message: Unescaped cookie value
Code: cookies[:font]
File: app/views/layouts/application.html.erb
Line: 12

Confidence: High
Category: Dangerous Send
...

```

実行結果は標準出力に吐き出されますが、`-o` オプションを付けるとファイルに出力でき、`-o filename.html` や `-o filename.csv` のように使います。CSVファイルはスプレッドシートに貼り付けやすいのでおすすめです。

```shell
user@sectest:~/railsgoat$ brakeman -A -o result.csv -o result.html
===略===
user@sectest:~/railsgoat$ more result.csv
Confidence,Warning Type,File,Line,Message,Code,User Input,Check Name,Warning Code,Fingerprint,Link
High,Cross-Site Scripting,app/views/layouts/application.html.erb,12,Unescaped cookie value,cookies[:font],,CrossSiteScripting,2,febb2
1e45b226bb6bcdc23031091394a3ed80c76357f66b1f348844a7626f4df,https://brakemanscanner.org/docs/warning_types/cross-site_scripting/
...
```

HTML出力した結果

![](/images/rails-static-security-testing/1.png)

出力されたCSVファイルをスプレッドシートにインポートした例：

![](/images/rails-static-security-testing/2.png)

このスプレッドシートは精査の記録に役立ちます。

### Brakeman の結果を精査する

Brakeman のアラートはすべてが脆弱性というわけではなく、誤報の場合もあります。特にインジェクション系の脆弱性は誤報が多いです。誤報だらけのレポートは役に立たないので精査しましょう。

`Confidence` は誤報度合いの参考にはなりますが、`High` で誤報のこともあれば、逆に `Weak` でもホンモノのこともあります。アラートが多くてすべてを見切れない場合を除き、すべてのアラートを精査することをお勧めします。

精査は `File` と `Line` が指し示しているソースコードを参照しながら、パラメータである `Code` と `User Input` が安全な値であるかを目視で確認していきます。

安全であるかの判断方法はいろいろあると思いますが、例を次に挙げます。

#### パラメータが検証されている場合

Brakeman はパラメータが検証されていたりサニタイズされている場合でもアラートを出すことがあります。この場合、アラートは False Positive と判断できます。

ただし、検証やサニタイズの妥当性は確認しておきましょう。もし安全である確信が持てない場合は、動的アプリケーションセキュリティテストで有効性を確認するのも良いでしょう。

#### パラメータの作成者が信頼できる場合

パラメータの作成者が信頼できるとき、そのアラートは False Positive と判断できる場合があります。Webアプリケーションで処理されるパラメータの参照元や作成者は様々です。

パラメータの参照元のと作成者例：

![](/images/rails-static-security-testing/3.png)

例えば Brakeman はクロスサイトスクリプティングのアラートを挙げており、実際に入力値がサニタイズされていない場合であっても、コンテンツ作成者が職務上パラメータにスクリプトを埋め込むことを許可されていれば問題なしと判断できるでしょう。

ただし、パラメータの作成者が信頼できるからと言って、パラメータの検証やサニタイズが無意味かというとそうでもありません。作成時点ではパラメータが安全な値であっても、使用されるまでの間に改ざんされていた場合、その値は安全とは言えません。
![](/images/rails-static-security-testing/4.png)

パラメータの参照元データが何らかの脆弱性によって改ざんされることが想定される場合、検証やサニタイズはリスクを緩和します。

### 精査結果の記録

先に出力したCSVファイルを整形し、精査結果を追記すると良いです。

![](/images/rails-static-security-testing/5.png)

RailsGoat はわざと脆弱性に作られたアプリなのでほぼすべてホンモノっぽいです。一部、セキュリティにかかわる設定不備を指摘しているものもあり、実害に発展しうるか現時点では判断をつけがたいアラートはとりあえず？としてます。

### Brakeman の注意点

Brakeman はとても便利なツールですが、次の3点を認識しておく必要があります。

#### 検出されたアラートの中には誤報(False Positive)が含まれる

Brakeman が報告するのはあくまで「疑わしいコード」なので、実際は問題がないコードが検出されることもあります。

本当に問題があるコードなのかを判断するために、結果を目視で精査する必要があります。

#### Brakeman では検出できない脆弱性がある

Brakeman はすべての脆弱性に対応しているわけではありません。また、対応していても漏れなく検出してくれるわけではありません。Brakeman が何も報告しなかった＝脆弱性ゼロとは思わないでください。

様々な脆弱性を網羅的にテストするには、Brakeman だけでなく目視によるコードレビューや動的アプリケーションセキュリティテストで補完する必要があります。

Brakeman が対応している脆弱性の種類は次のページにあります：[Brakeman Warning Types](https://brakemanscanner.org/docs/warning_types/)

#### 誤報でなかったとしても、実際に悪用可能かは分からない

Brakeman が検出するのはソースコード上の問題なので、Web アプリケーションとして動くときにどのような挙動をするかは、実際に動かしてみるまで分かりません。

その問題が実際に悪用可能であるかを確かめるには、動的アプリケーションセキュリティテスト(DAST)が必要です。DASTについてはここでは扱いません。

### Brakeman まとめ

Brakeman を使ってソースコードにあるセキュリティ上の欠陥を検出しました。

Brakeman は自動かつ高速にソースコードを検査できますが、すべての脆弱性を発見できるわけでもなく、また、誤報もあります。

そのため、Brakemanの結果を目視で精査したり、網羅できていない脆弱性を別のテストで補完する必要があります。

## 手動によるセキュリティコードレビュー

Brakeman は脆弱性の発見にとても役立ちますが、Brakemanでは発見できない脆弱性もあります。特にアプリ固有の脆弱性を発見するにはソースコードを手動でレビューすることも必要です。

コードをなんとなく眺めて脆弱性を発見できることもありますが、あらかじめレビューの観点やルールを定めておくと、再現性のあるテストを効率よく実行できるようになります。

セキュリティコードレビューに標準的なルールというものはなく、またコードには組織や開発チームのクセみたいなものもありますので、テストしながらチームの中で育てていくのが良いと思います。

本節では筆者のオレオレルールを3つ紹介しますので参考にしてください。

### Mass Assignment / 不適切な Strong Parameters の設定

#### 概要

Update 系の処理で使われる Strong Parameters が適切に設定されていない場合、攻撃者によってデータを改ざんされる可能性があります。

あからさまな Mass Assignment は Brakeman でも検出できますが、アラートが出ないパターンもあるので目視でもチェックしておくとよいです。

#### ルール

コントローラー全体を `.permit` で検索し、Strong Parameters で permit している属性が必要最小になっていることを確認しましょう。

必要最小とは、「アクションを呼び出すユーザが変更できる属性」であることです。これを厳密に判断するにはアプリケーションの仕様を理解しておく必要がありますが、直感的には「Web画面で編集できる項目以外にpermitされている属性があればNG」と判断してよいと思います。

#### RailsGoat の場合

RailsGoat には`.permit`にマッチするコードが4か所ありました。

![](/images/rails-static-security-testing/6.png)

1つ目：

```ruby
params.require(:message).permit(:creator_id, :message, :read, :receiver_id)
```

permitのパラメータはユーザに変更されても問題なさそうな属性です。大丈夫そうですね。

補足：このパラメータは `create` メソッドで使われるため問題ないですが、もし `update` で使われる場合はNGかもしれません。更新時に `creator_id` を変更できるのは変ですよね。

2つ目：

```ruby
params.require(:schedule).permit(:date_begin, :date_end, :event_desc, :event_name, :event_type)
```

同様に大丈夫そう。

3つ目：

```ruby
params.require(:user).permit!
```

`permit!`はすべての属性をpermitしちゃいます。高確率でダメです。

4つ目：

```ruby
params.require(:user).permit(:email, :admin, :first_name, :last_name)
```

`admin` が非常に怪しいです。もしこれが管理者権限フラグなら権限昇格できてしまうかもしれません。

※ちなみに3つ目と4つ目のパターンはBrakemanでも検出できます

検索結果の `Open in editor` をクリックすると、該当する行の前後n行も合わせて確認できます。

![](/images/rails-static-security-testing/7.png)

これをスプレッドシートに貼り付けるとレビューの記録を付けることができます。

![](/images/rails-static-security-testing/8.png)

### 権限昇格 (IDOR)

#### 概要

あるWebサイトの `My Account` をひらいたら URL が `http://example.com/users/1234` だった時を想像してください。URL の `1234` は自分のユーザIDと推測できます。この `1234` を `1` に変えてアクセスしたらどうなるでしょうか。

もし他の人のアカウント情報が参照できたら、そのサイトはアクセス制御に問題があります。そしてこのような攻撃手法を Insecure Direct Object Reference (IDOR) と言い、権限昇格につながる脆弱性です。参照だけでなく、更新、削除についても同様です。

アクセス制御はアプリケーション固有の仕様となるため、Brakemanで検知することはできません。つまりソースコードから権限昇格の脆弱性を検出するには、目視によるレビューが必要です。

#### ルールの例

コントローラを正規表現 `params\[:.*id\]` で検索し、リソースにアクセス制御が必要な場合、アクションまたはクエリにアクセス権が含まれているかを確認します。

わかりづらいので具体例を挙げます。ブログ記事(article)を修正できるのは自分だけ、という仕様を想像しつつ次のパターンAとパターンBのコードを観察してください。

A. 認可が含まれていないクエリ：

```ruby
def update:
  @article = Articles.find(params[:article_id])
  @article.update!(article_params)
  redirect_to @article
end
```

B. 認可が含まれているクエリ：

```ruby
def update:
  @article = @current_user.articles.find(params[:article_id])
  @article.update!(article_params)
  redirect_to @article
end
```

Aの場合 `article_id` パラメータを改ざんすれば他人の記事を更新できてしまいます。対してBは他人の `article_id` を入れても `@article` は nil となるため安全です。

#### RailsGoat の場合

RailsGoat には該当するコードが10か所ありました。

![](/images/rails-static-security-testing/9.png)

いくつかのパターンがあるので、それぞれ見てみます。

1つ目のパターン：IDOR

```ruby
class MessagesController < ApplicationController
# ...省略...
  def show
    @message = Message.where(id: params[:id]).first
  end
```

IDパラメータを別の番号変えると、他人のメッセージも見れてしまいそうです。NGです。（もし他人のメッセージも見れるという仕様であればOKです）

2つ目のパターン：IDORかつSQLインジェクションっぽい

```ruby
class UsersController < ApplicationController
# ...省略...
  def update
    message = false

    user = User.where("id = '#{params[:user][:id]}'")[0]
```

IDパラメータを別の番号変えると他人のユーザ情報を更新できてしまいそうです。さらにプレースホルダを使ってないので、SQLインジェクションもできそうですね。

このように本来の観点とは異なる問題が見つかることもあります。

3つ目のパターン：IDORっぽいけど違う

```ruby
class AdminController < ApplicationController
# ...省略...
  def get_user
    @user = User.find_by_id(params[:admin_id].to_s)
    arr = ["true", "false"]
    @admin_select = @user.admin ? arr : arr.reverse
  end
```

一見怪しいですが、問題ないかもしれません。AdminControlerという名前から、このアクションは管理機能に見えます。管理者はすべてのユーザ情報を参照できるという仕様であれば、IDORではありません。

ただしこのアクションが本当に管理者からのみ呼ばれる仕様なのかは確認しましょう。

4つ目のパターン：IDORではないが別の問題

```ruby
def admin_param
  params[:admin_id] != "1"
end
```

クエリではないため IDOR ではありません。しかしクエリパラメータ `admin_id` が `1` の時を特別に扱っていることから、別の問題がありそうです。

このようにレビュー中に気になったことも記録しておくとよいでしょう。作業記録の例：

![](/images/rails-static-security-testing/10.png)

#### TIPS

実際のセキュリティテストでも、この例のようにレビュー中気になることが次々と出てくることはよくあります。

すぐに済むならその場で確認しても良いですが、あまり深追いし過ぎると自分が今何をやってるかわからなくなってきます。そのため、確認事項があってもその場ではメモするにとどめておき、実際に確認するのは作業途中のレビューが片付いてからにしたほうが良いです。

### レースコンディション

#### 概要

レースコンディションとは、同一のリソースに同時にアクセスした場合に想定外の動作が発生する事象をいいます。

レースコンディションには様々なパターンが有りますが、ここでは現在時刻をもとにIDやファイル名を決定している場合に問題が発生するケースを挙げます。

#### ルール

ソースコード全体を正規表現 `Time\.(now|current|zone\.now)` で検索し、IDやファイル名などを現在時刻だけで作成している場合、NGと判断します。

#### RailsGoat の場合

RailsGoat には1つありました。

![](/images/rails-static-security-testing/11.png)

```ruby 
def self.make_backup(file, data_path, full_file_name)
  if File.exist?(full_file_name)
    silence_streams(STDERR) { system("cp #{full_file_name} #{data_path}/bak#{Time.zone.now.to_i}_#{file.original_filename}") }
  end
end
```

現在時刻と `original_filename` を組み合わせたファイル名でバックアップを作成しているようです。可能性は低いですが、`original_filename` が同一のリクエストが同時に2回以上実行されたら、1回目に作成されたバックアップファイルが2回目以降に作成されるバックアップファイルで上書きされてしまいそうです。

次にこのメソッドがどこから呼ばれるかを突き止めます。`make_backup` メソッドを呼出してるメソッドを呼出してるメソッド...と辿った結果、`BenefitFormsController` の `upload` メソッドに行きつきました。

![](/images/rails-static-security-testing/12.png)

`upload` はWebサイトの利用者から呼ばれるアクションなので、顕在化する可能性があると判断できます。

### コードレビューのまとめ

独自のルールに基づいたコードレビューの例を3つ挙げました。

1. `.permit` で検索し、不適切な Strong Parameters 設定を発見する
2. `params\[:.*id\]` で検索し、権限昇格(IDOR)の問題を発見する
3. `Time\.(now|current|zone\.now)`で検索し、ID重複によるレースコンディションを発見する

上記以外にもいろいろなルールが考えられますので、セキュリティチームでレビュールールを育てていくとよいです。

今回は予めルールを定めたうえでレビューしましたが、ルールは無くても勘や思い付きに頼って脆弱性を探索することもできます。特に未知の脆弱性を発見するには、レビュアーの経験に基づく探索的なテストも必要です。

## まとめ

本記事では Rails アプリケーションの静的アプリケーションテストの手法として、Brakemanを使った自動化されたセキュリティコードレビューと、目視によるセキュリティコードレビューの手法を紹介しました。

また、紹介の中で様々なテスト手法を挙げ、セキュリティコードレビューだけでは網羅できない領域があることにも触れました。

- 静的テスト・動的テスト (SAST vs DAST)
- 手動テスト・自動テスト (Brakeman vs 目視)
- 手順が定められたテスト・アドホックなテスト (ルール vs 勘)

これらは独立しているわけではなく、それぞれが補完しあうテスト手法です。セキュリティテストに限らずソフトウェアテスト全般に言えることですが、費用対効果の高いテストをするには、複数のテスト手法を組み合わせることが重要です。

## 課題

本ガイドでは Brakemanでは対応していない脆弱性を検出するために文字列・正規表現を使ってコードを抽出し目視チェックをしましたが、実際には文字列や正規表現のマッチングでは抽出が難しい場合もあります。

また、ソースコードを毎回検索して目視チェックするというのもあまり効率的ではありません。

そこで次回は[Semgrep](https://github.com/returntocorp/semgrep)というツールを使って、ルールの作成とセキュリティコードレビューを効率化する方法を紹介します。