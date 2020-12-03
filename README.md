# PHP7製のフレームワークを作ろう！

## 1. ClassLoaderクラスを作ろう

ClassLoaderクラスはオートロードに関する処理をまとめたクラス。
- PHPにオートローダークラスを登録する処理（register()メソッド） 
- オートロードが実行された際にクラスファイルを読み込む処理（loadClass()メソッド）

## 2. bootstrap.phpを作ろう

作成したClassLoaderをオートロードに登録するための設定ファイルです。

現段階では明示的にrequireを用いてクラスファイルを読み込みます。

registerDir()で coreディレクトリと modelsディレクトリをオートロードの対象ディレクトリに設定し、register() でオートロードに登録します。

## 3. Requestクラスを作ってURLを制御しよう

HTTPメソッド(GET/POST)の判定や $_GET、$_POST などの値の取得、リクエストされたURLの取得、サーバのホスト名やSSLでのアクセスかどうかと行った判定などの機能の実装をしているクラスです。

isPost() メソッドで、HTTPメソッドが POST かどうかの判定を行います。また、getGet() メソッドや getPost() メソッドでそれぞれ $_GET、$_POST 変数からの値の取得を行います。

getHost() メソッドで、サーバのホスト名の取得をします。isSsl() メソッドでは HTTPS でのアクセスかどうかの判定を行います。getRequestUri() メソッドではリクエストされたURLのホスト部分以降を返します。

getBaseUrl() メソッドでベースURL(ホスト部分より後ろから、フロントコントローラまでの値)を取得し、getPathInfo() メソッドで PATH_INFO(ベースURLより後ろの値)を取得します。これらの値を利用してURLの制御を行って行きます。

## 4. Routerクラスを作ってルーティングを制御しよう

Routerクラスはルーティング定義配列とPATH_INFOを受け取り、ルーティングパラメータを特定する役割をもちます。

compileRoutes() メソッドで、受け取ったルーティング定義配列中の動的パラメータ指定を正規表現で扱える形式に変換します。resolve() メソッドで、変換済みのルーティング定義配列とPATH_INFOマッチングを行い、ルーティングパラメータの特定を行います。特定できなかった場合は False を返します。

## 5. Responseクラスを作ろう

Responseクラスはレスポンスを表すクラスで、HTTPヘッダとHTMLなどのコンテンツを返すのが主な役割です。

setContent() メソッドで、$contentプロパティにHTMLなどの実際にクライアントに返す内容を格納します。setStatusCode() メソッドで、$status_codeプロパティ、$status_textプロパティにHTTPのステータスコードを格納します。setHttpHeader() メソッドで、$http_headersプロパティにHTTPヘッダを格納します。send() メソッドで各プロパティに設定された値を元にレスポンスの送信を行います。

## 6. PDOによるデータベースアクセス

DbManagerクラスでは、PHPに標準で付属するデータベース抽象化ライブラリであるPDO(PHP Data Object)クラスを使って、データベースへのアクセスを管理します。

DbRepositoryクラスは、実際にはデータベース上のテーブルごとにこのクラスを継承したクラスを作成し、そこにデータベースアクセス処理を記述して行きます。userテーブルがあれば、UserRepositoryクラスを作成する、と行った具合です。

DbManagerクラスとDbRepositoryクラスの関係性としては、DbManagerクラスの内部にテーブルごとのRepositoryクラスを保持するようにします。DbManagerクラスのインスタンスが $db_manager 変数に入っているとして、`$db_manager->get('User')` のようにして UserRepository クラスを取得するようにします。

##### DbManagerクラスの使い方
```
$db_manager = new DbManager();
$db_manager->connect('master', array(
    'dsn'      => 'mysql:dbname=mydb;host=localhost',
    'user'     => 'myuser',
    'password' => 'mypass',
));
$db_manager->getConnection('master');
$db_manager->getConnection();  #=> master が返ってくる
```
##### DbRepositoryクラスと接続情報のマッピング

DbManagerクラス内、getConnection() には引数の指定がなければ最初に作成したコネクションを取得するようにしてある。それ以外のものを使用する場合に備え、getConnectionForRepository()メソッドの実装も行っている。

$repository_connection_mapプロパティにテーブルごとのRepositoryクラスと接続名の対応を格納し、getConnectionForRepository()メソッドでRepositoryクラスに対応する接続を取得しようとした際に、$repository_connection_mapプロパティに設定されているものは getConnection()メソッドに接続名を指定し、そうでなければ最初に作成したものを取得するという機能になる。

##### Repositoryクラスの管理

DbRepositoryクラスは後ほど記述するが、１度インスタンスを作成したら、それ以降インスタンスを生成する必要は無いように実装してあり、全てのインスタンスをDbManagerクラスで管理できるようにしている。それらは全て `$repositories` に格納している。

##### DbRepositoryクラス

DbRepositoryクラスはデータベースへのアクセスを行うクラスで、テーブルごとにDbRepositoryクラスの子クラスを作成するようにします。

各Repositoryクラスには、例えばuserテーブルであればUserRepositoryクラスを定義し、userテーブルへレコードの新規作成を行う `insert()`メソッドや id というカラムを元にデータを取得する `fetchById()`メソッドなどを必要に合わせて作成していくことを想定しています。

それぞれのメソッド内部ではSQLを実行することになりますが、SQLの実行時に頻繁に出てくるような処理をDbRepositoryに抽象化しておきます。

##### DbRepository内でのPDOクラスについて

__construct()メソッドとsetConnection()メソッドはDbManagerクラスからPDOクラスのインスタンスを受け取って内部に保存するメソッドです。ここで受け取ったPDOクラスのインスタンスに対してSQL文を実行します。

プリペアドステートメントについても補足しておきます。
```
$name = $_POST['name'];
$sql = "INSERT INTO user (name) VALUES('" . $name . "')";
```
上記の例はSQLに脆弱性があります。ユーザの入力に対しては下記の例のように適切に入力値をエスケープする必要があります。
```
$sql = 'INSERT INTO user (name) VALUES(:name)';
$stmt = $con->prepare($sql);

$params = array(':name' => $_POST['name']);
$stmt->execute($params);
```

