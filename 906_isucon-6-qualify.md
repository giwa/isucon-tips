# ISUCON6 qualify

## Regulation
[isucon6 official](http://isucon.net/archives/48396740.html)

## Theme: Microservice Anti-pattern
http://isucon.net/archives/48697611.html

### Application

はてなキーワード、 (?:匿名)? ダイアリーを模したブログとWikiの中間の様なアプリケーションです。キーワード自動リンク機能がついています。また、はてなスターのようなお気に入りを付けられる様な機能もついていました。記事の投稿時にはスパムチェックをおこなっており、一部の禁止ワードや、アダルトサイトへのリンクが含まれている場合には投稿できないようになっています。

### Archtecture

初期状態で以下の3種類のアプリケーションが起動しており、それぞれが通信を行なっていました。
- isuda (はてなキーワード・はてな (?:匿名)? ダイアリーを模したアプリケーション)
- isutar (はてなスターを模したサブアプリケーション)
- isupam (スパムチェッカー/Go製のバイナリのみ提供)

今回の問題のテーマは「マイクロサービスアンチパターン」でした
isudaとisutarは相互に依存しあっているというアンチパターンで、isupamはソースコードが紛失してしまっているという設定でした。

## Bench

> 今回、予選問題を作るにあたって以下の様な狙いがありました。

> 余りキャッシュが有効にならない問題にしたかった
> 多くの予選参加者が無条件失格にはならずにスコアが残せるような作りにしたかった

-> 失敗レスポンスを大幅に許容してもキャッシュで押し切れる様になってしまった

## Score

> この初期スコアのばらつきは、各言語の正規表現エンジンの性質によるところが大きいです。問題を作成していて特に困ったのがGoとPHPで、Goの正規表現は想像を絶する遅さであり、

## Scenario

ベンチマーカーには以下の様なシナリオが設定されていました。

- 正しくログインが行えるか/不正なログインを弾けるか
- スパムコンテンツを適切に弾いているか
- スターを正しく付けられるか/不正なスターが付けられないようになっているか
- トップページから深いページに遡っても適切なレスポンスが返せているか
- ランダムなキーワードにアクセスして正しくレスポンスを返せるか
- キーワードを新たに追加できるか。その場合トップページに反映されるとともに新たなキーワードリンクが適切に設定されるか
- 既存のキーワードを更新できるか。その場合トップページに正しく反映されるか



## 小ネタ
今回、各言語実装で、DB接続時に SET SESSION sql_mode='TRADITIONAL,NO_AUTO_VALUE_ON_ZERO,ONLY_FULL_GROUP_BY' というクエリを発行していますが、これは kamipo TRADITIONALです。皆さん、kamipo TRADITIONALを使いましょう。

今回の問題のログインユーザーは、PerlのCPANモジュールの Acme::CPANAuthors::Japanese からリストを作成しています。

http://www.songmu.jp/riji/entry/2015-07-08-kamipo-traditional.html


## やること
### Monothilic
> 今回のケースだと、isudaとisutarが相互に依存しており無駄な通信も多いため、この2つを結合してモノリシックにしてしまうのが良いでしょう。マイクロサービスを捨ててモノリシックにするのが正解と言うのは皮肉が効いていて面白いのではないでしょうか。

TODO: isucon6通過者の実装例

### SQL
> キーワードリンクを作る際に毎回テーブルを全件走査して正規表現を組み立てているのは無駄なので、 SELECT * をやめて、 keyword のみの取得にしつつ、そこの処理は1リクエストに付き高々1回にしてしまうこともやると良いでしょう。

```
# before
keywords = db.xquery(%| select * from entry order by character_length(keyword) desc |)

# after
keywords = db.xquery(%| select keyword from entry order by character_length(keyword) desc |)
```

https://github.com/isucon/isucon6-qualify/blob/ba21fa19573deba630f34ebe470141dff6a67273/webapp/go/isuda.go#L96-L99



```go
rows, err := db.Query(fmt.Sprintf(
		"SELECT keyword FROM entry ORDER BY updated_at DESC LIMIT %d OFFSET %d",
		perPage, perPage*(page-1),
	))
```

### Keyword link

> 今回の問題の本丸は、キーワードリンク作成の高速化です。ここに関しては、正直に言うと余り模範解答を考えていませんでした。想定解答としては、キーワード更新時に正規表現を作り直して、memcachedなりオンメモリにキャッシュしてしまうというものです。

> Goでオンメモリに正規表現をキャッシュしてしまえばかなり強いのではないかと思っていたのですが、 Go の正規表現エンジンが想定以上に遅く、その戦略は有効になりませんでした。ただ、 strings.Replacer が有効だったことは盲点でした。

> キーワード投稿時にLIKE検索を使ってキャッシュのpurgeをおこなう戦略も有効でこれも盲点でした。初期データのデータ量を絞らざるを得なかったからこそ有効だった手段ですが、想定外の解法であり脱帽です。

TODO: isucon6通過者の実装例

#### N+1
[htmlify](https://github.com/isucon/isucon6-qualify/blob/ba21fa19573deba630f34ebe470141dff6a67273/webapp/go/isuda.go#L307-L342)



### スパムチェッカーisupam

> スパムチェッカーのisupamに関しては、これは狙いとしては完全に目眩ましであり、ボトルネックにはならないところなので手をいれるところではなく、気にしたら負け、くらいの感覚でした。

