# ユニバーサルなコードを書く

サーバサイドの描画について、さらに見ていく前に、"ユニバーサル"なコード(サーバーとクライアントの両方で動作するコード)を記述するときの制約について考えてみましょう。ユースケースとプラットフォームの API の相違により、異なる環境で実行したコードの動作は全く同じものにはなりません。ここでは、サーバサイドの描画を行う上で、知っておく必要がある重要な項目について説明します。

## サーバー上でのデータリアクティビテイ

クライアントだけで実行するアプリでは、全てのユーザーがブラウザでアプリケーションの新しいインスタンスを使用します。サーバサイドの描画でも、同じ振る舞いが必要とされます: すなわち、複数のリクエストに跨った状態の汚染がないよう各リクエストは新しく独立したアプリケーションのインスタンスが必要になります。

実際の描画プロセスは決定的であることが求められるので、サーバー上でデータを"プリフェッチ"することもあります。これは描画を開始する時、アプリケーションの状態は既に解決済みであることを意味します。つまり、サーバー上では、データがリアクティブである必要はないので、デフォルトで無効になっています。データをリアクティブにしないことで、データをリアクティブなオブジェクトに変換する際のパフォーマンスコストを無視できます。

## コンポーネントのライフサイクルフック

動的な更新がないので、ライフサイクルフックのうち、`beforeCreate` と `created` のみが SSR 中に呼び出されます。つまり、 `beforeMount` や `mounted` などの他のコンポーネントサイクルフックは、クライアントでのみ実行されます。

注意すべきもう一つは、`beforeCreate` と `created` において、例えば  `setInterval` でタイマーを設定するような、グローバルな副作用を引き起こすコード避けるべきです。クライアントサイドのみのコードでは、タイマーを設定してから `beforeDestroy` または `destroyed` したりすることができます。しかしながら、SSR 中に破棄フックは呼び出されないため、タイマーは永遠に残ります。 これを回避するために、代わりに副作用コードを `beforeMount` または `mounted` に移動してください。

## プラットフォーム固有の API にアクセスする

ユニバーサルコードでは、プラットフォーム固有の API へのアクセスは想定されていないので、`window` や `document` といったブラウザ環境のグローバル変数を直接使用すると、Node.js ではエラーが発生します。

サーバーとクライアントでコードを共有するものの、タスクが使用する API がプラットフォームによって異なる場合は、プラットフォーム固有の実装を ユニバーサル な API の内部でラップするか、それを行うライブラリを使用することをお勧めします。例えば、[axios](https://github.com/mzabriskie/axios) は、サーバーとクライアントの両方に同じ API を提供する HTTP クライアントです。

ブラウザ  API を利用する際の一般的なアプローチは、クライアントだけで実行されるライフサイクルの中で遅延的にアクセスすることです。

サードパーティライブラリがユニバーサルに使用することを考慮していない場合、それをサーバサイドによって描画されるアプリケーションに統合することは難しいので注意してください。グローバル変数のいくつかをモックすることで動かすことができるようになる *かも*しれませんが、それはハックであり、他のライブラリの環境検出のコードを妨げる恐れがあります。 

## カスタムディレクティブ

ほとんどの カスタムディレクティブ は直接 DOM を操作するため、SSR 中にエラーが発生します。これを回避するには、2つの方法があります:

1. 抽象化の仕組みとしてコンポーネントを使用し、カスタムディレクティブの代わりに仮想 DOM レベル（例えば、render 関数を使用すること)で実装することをお勧めします。
2. コンポーネントに簡単に置き換えができないカスタムディレクティブの場合、サーバーレンダラを生成する際の [`directives`](./api.md#directives) オプションを使用して、そのオプションの "サーバーサイドのバージョン" を用意することで回避できます。
