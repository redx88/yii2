パフォーマンスチューニング
==========================

> Note|注意: この節はまだ執筆中です。

あなたのウェブアプリケーションのパフォーマンスは二つの部分に基づいています。
一つはフレームワークのパフォーマンスであり、もう一つはアプリケーションそのものです。
Yii は、そのままの状態でも、あなたのアプリケーションのパフォーマンスを劣化させる影響がかなり小さいものですが、本番環境のためには、さらに微調整することが可能です。
アプリケーションに関しては、ベストプラクティスのいくつかを提供すると共に、それを Yii に適用する方法を例示します。

環境を準備する
--------------

PHP アプリケーションを走らせるための環境を正しく構成することは本当に重要です。
最大のパフォーマンスを得るためには、

- 常に最新の安定した PHP バージョンを使うこと。
PHP は、メジャーリリースのたびに、顕著なパフォーマンスの改善とメモリ使用の削減がなされています。
- PHP 5.4 以下には [APC](http://ru2.php.net/apc)、または、PHP 5.5 以上には [Opcache](http://php.net/opcache) を使うこと。
このことは、非常に良いパフォーマンス強化をもたらします。

フレームワークを本番用に準備する
--------------------------------

### デバッグモードを無効化する

アプリケーションを本番環境に配備する前に行うべき最初のことは、デバッグモードの無効化です。
Yii のアプリケーションは、`index.php` において定数 `YII_DEBUG` が `true` と定義されていると、デバッグモードで走ります。
従って、デバッグモードを無効にするために、以下のコードが `index.php` になければなりません。

```php
defined('YII_DEBUG') or define('YII_DEBUG', false);
```

デバッグモードは開発段階において非常に役に立ちますが、いくつかのコンポーネントがデバッグモードにおいて追加の負荷を発生させるため、パフォーマンスを劣化させます。
例えば、メッセージロガーは、ログに記録されるすべてのメッセージについて、追加のデバッグ情報を記録します。

### PHP opcode キャッシュを有効にする

PHP opcode キャッシュを有効にすると、すべての PHP アプリケーションで、顕著にパフォーマンスが向上し、メモリ使用量が削減されます。
Yii も例外ではありません。
[PHP 5.5 OPcache](http://php.net/manual/ja/book.opcache.php) と [APC PHP 拡張](http://php.net/manual/ja/book.apc.php) の両方でテストされています。
どちらのキャッシュも、PHP 中間コードを最適化して、入ってくるリクエストごとに PHP スクリプトを解析するために時間を消費することを回避します。

### ActiveRecord のデータベーススキーマキャッシュを有効にする

アプリケーションがアクティブレコードを使用している場合は、スキーマキャッシュを有効にして、データベーススキーマを解析するための時間を節約すべきです。
そうするためには、アプリケーションの構成情報 `config/web.php` において、`Connection::enableSchemaCache` プロパティを `true` に設定します。

```php
return [
    // ...
    'components' => [
        // ...
        'db' => [
            'class' => 'yii\db\Connection',
            'dsn' => 'mysql:host=localhost;dbname=mydatabase',
            'username' => 'root',
            'password' => '',
            'enableSchemaCache' => true,

            // スキーマキャッシュの持続時間
            // 'schemaCacheDuration' => 3600,

            // 使用されるキャッシュコンポーネントの名前。デフォルトは 'cache'。
            //'schemaCache' => 'cache',
        ],
        'cache' => [
            'class' => 'yii\caching\FileCache',
        ],
    ],
];
```

`cache` [アプリケーションコンポーネント](structure-application-components.md) が構成されていなければならないことに注意してください。

### アセットを結合して最小化する

アセットは、典型的には JavaScript と CSS ですが、結合して圧縮することが出来ます。
これにより、ページの読み込みにかかる時間をわずかながら削減して、アプリケーションのエンドユーザにより良いユーザ体験をもたらすことが出来ます。

これをどのようにすれば達成できるかについて学ぶためには、ガイドの [アセット](structure-assets.md) の節を参照してください。

### セッションのためにより良いストレージを使用する

デフォルトでは、PHP はセッションを処理するためにファイルを使います。
開発と小さなプロジェクトではそれでも構いませんが、リクエストを並列処理するとなると、データベースのような別のストレージに変更する方が良いでしょう。
そうするためには、`config/web.php` によってアプリケーションを構成します。

```php
return [
    // ...
    'components' => [
        'session' => [
            'class' => 'yii\web\DbSession',

            // デフォルトの 'db' 以外の DB コンポーネントを使用したい場合は
            // 以下を設定する
            // 'db' => 'mydb',

            // デフォルトの session テーブルをオーバーライドするためには
            // 以下を設定する
            // 'sessionTable' => 'my_session',
        ],
    ],
];
```

`CacheSession` を使って、セッションをキャッシュに保存することが出来ます。
キャッシュストレージの中には、memcached のように、セッションデータが失われないことを保証しないものもあり、予期せぬログアウトを引き起こす場合があることに注意してください。

サーバに [Redis](http://redis.io/) がある場合は、それをセッションのストレージに使用することを強く推奨します。

アプリケーションを改善する
--------------------------

### サーバ側のキャッシュテクニックを使う

キャッシュの節で説明されているように、Yii はウェブアプリケーションのパフォーマンスを著しく改善することが出来るキャッシュのソリューションをいくつか提供しています。
データの生成に長い時間を要するものがある場合は、データキャッシュの手法を使用して、データを生成する頻度を削減することが出来ます。
ページのある部分が比較的静的なものであり続ける場合は、フラグメントキャッシュの手法を使用して、その部分のレンダリングの頻度を削減することが出来ます。
あるページ全体が比較的静的なものであり続ける場合は、ページキャッシュの手法を使用して、ページ全体のレンダリングのコストを節約することが出来ます。


### HTTP キャッシュを利用して、処理時間と帯域を節約する

HTTP キャッシュを利用すると、処理時間および帯域やリソースを著しく節約することが出来ます。
HTTP キャッシュは、`ETag` または `Last-Modified` ヘッダをアプリケーションのレスポンスで送信することによって実装することが出来ます。
ブラウザが HTTP の仕様に従って実装されていれば (ほとんどのブラウザがそうです)、コンテントは以前の状態と異なる場合にだけ取得されます。

正しいヘッダを作成することは手間のかかる仕事ですので、Yii は [[yii\filters\HttpCache]] というコントローラフィルタの形でショートカットを提供しています。
これを使うことはとても簡単です。
コントローラの中で `behaviors` メソッドを以下のように実装することが必要です。

```php
public function behaviors()
{
    return [
        'httpCache' => [
            'class' => \yii\filters\HttpCache::className(),
            'only' => ['list'],
            'lastModified' => function ($action, $params) {
                $q = new Query();
                return strtotime($q->from('users')->max('updated_timestamp'));
            },
            // 'etagSeed' => function ($action, $params) {
                // return // egat のシードをここで生成
            //}
        ],
    ];
}
```

上記のコードでは、`etagSeed` または `lastModified` のどちらかを使うことが出来ます。
両方を実装する必要はありません。
目的は、コンテントを取得してレンダリングするよりも安価な方法を使って、コンテントが修正されたかどうかを判断することです。
`lastModified` はコンテントが最後に修正されたときの UNIX タイムスタンプを返さなければなりません。
一方 `etagSeed` は `ETag` ヘッダの値を生成するために使われる文字列を返さなければなりません。


### データベースの最適化

データベースからのデータ取得がウェブアプリケーションのパフォーマンスの主たるボトルネックになることがよくあります。
[キャッシュ](caching.md#Query-Caching) の使用によってパフォーマンスの劣化を緩和することは出来ますが、問題を完全に解決することは出来ません。
データベースが膨大なデータを抱えている場合、キャッシュされたデータが無効化されたときに最新のデータを取得するためのコストは、データベースとクエリが適切に設計されていないと、法外なものになり得ます。

データベースのインデックスを上手に設計しましょう。
インデックスを付けると SELECT クエリを非常に速くすることが出来ます。ただし、INSERT、UPDATE、または DELTE のクエリは遅くなることがあります。

複雑なクエリに対しては、PHP コードの中からクエリを発行して DBMS にクエリを解析するように繰り返して求める代わりに、データベースビューを作成することを推奨します。

アクティブレコードを使い過ぎないでください。
アクティブレコード は OOP 流にデータをモデリングするには便利ですが、クエリ結果の各行を表すために一つまたは複数のオブジェクトを作る必要があるため、パフォーマンスを現実に低下させます。
膨大なデータを扱うアプリケーションでは、より低レベルの DAO や データベース API を使うのが良い選択でしょう。

最後にもう一つ大事なことですが、SELECT クエリで LIMIT を使ってください。
こうすることで、大量のデータが返されて、PHP のために確保されたメモリを使い尽くすということがなくなります。

### asArray を使う

読み出し専用のページにおいて、メモリと処理時間を節約する良い方法に、アクティブレコードの `asArray` メソッドを使うという方法があります。

```php
class PostController extends Controller
{
    public function actionIndex()
    {
        $posts = Post::find()->orderBy('id DESC')->limit(100)->asArray()->all();
        return $this->render('index', ['posts' => $posts]);
    }
}
```

ビューにおいては、`$posts` の個々のレコードのフィールドを配列としてアクセスしなければなりません。

```php
foreach ($posts as $post) {
    echo $post['title'] . "<br>";
}
```

`asArray` が指定されていなくても配列記法を使用してフィールドにアクセスすることが出来ますが、その場合は AR オブジェクトを扱っていることに注意してください。

### Composer オートローダを最適化する

現代的な PHP アプリケーションでは、オートローダの性能が非常に重要です。
``` php composer.phar dumpautoload -o ``` を使って Composer のオートローダを最適化するのが最大の性能を引き出す方法です。
ただし、最適化を適用するのはアプリケーション開発者の責務です。

php composer.phar dumpautoload -o


### バックグラウンドでデータを処理する

ユーザのリクエストに素早く応答したい場合、リクエストの重い部分は、それについて即座にレスポンスを返す必要がなければ、後から処理することが出来ます。

これを達成する一般的な方法が二つあります。クロンジョブによる処理と、専用のキューです。

最初のケースでは、後から処理したいデータを、データベースなどの持続的ストレージに保存する必要があります。
そして、クロンジョブによって定期的に実行される [コンソールコマンド](tutorial-console.md) がデータベースを検索して、データがあれば処理します。

このソリューションでたいていの場合は OK ですが、一つ大きな欠点があります。
データベースを検索するまでは処理すべきデータの有無を知ることが出来ません。
そのため、データベースをかなり頻繁に検索するか、または、各処理の間に若干の遅延を生じさせるかのどちらかになります。

この問題は、キューやジョブサーバ (RabbitMQ、ActiveMQ、Amazon SQS、その他いろいろ) によって解決することが出来ます。
この場合は、持続的ストレージにデータを書き込む代りに、キューやジョブサーバによって提供される API を通じてデータをキューに入れます。
処理はたいていはジョブハンドラのクラスに渡されます。
キューに入れられたジョブは、先行するジョブが全て完了した直後に実行されます。

### 何をしても効果がない場合

何をしても効果がない場合は、何がパフォーマンスの問題を解決するかについての思い込みを排することです。
代りに、何でも変更する前には、常にコードをプロファイルにかけてください。
次のツールが役に立つでしょう。

- [Yii のデバッグツールバーとデバッガ](tool-debugger.md)
- [XDebug プロファイラ](http://xdebug.org/docs/profiler)
- [XHProf](http://www.php.net/manual/en/book.xhprof.php)
