# PHP7製のオリジナルフレームワークを作ろう！

## 1. ClassLoaderクラスを作ろう

ClassLoaderクラスはオートロードに関する処理をまとめたクラス。
- PHPにオートローダークラスを登録する処理（register()メソッド） 
- オートロードが実行された際にクラスファイルを読み込む処理（loadClass()メソッド）

## 2. bootstrap.php を作ろう

作成したClassLoaderをオートロードに登録するための設定ファイルです。

現段階では明示的にrequireを用いてクラスファイルを読み込みます。

registerDir()で coreディレクトリと modelsディレクトリをオートロードの対象ディレクトリに設定し、register() でオートロードに登録します。