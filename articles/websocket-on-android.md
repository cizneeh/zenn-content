---
title: "AndroidでWebSocket通信"
emoji: "📱" 
type: "tech"
topics: ["Android", "Kotlin"]
published: true
---

# WebSocketとは？
初めにピア間で通信を確立し、その後はそのコネクション上で低コストな双方向通信が出来るプロトコルです。

HTTPの拡張のような形のプロトコルで、利用するポート番号も同じなので経路上のネットワーク機器によるトラブルに見舞われることも少ないという特徴があります。

参考
- [RFC 6455 - The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)
- [WebSocket サーバーの記述 - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/WebSockets_API/Writing_WebSocket_servers)
- [WebSocketについて調べてみた。 - Qiita](https://qiita.com/south37/items/6f92d4268fe676347160)

# AndroidでWebSocketがしたい
仕事でAndroidアプリでWebSocketを扱うことになって色々調べていたのですが、どうやらHTTPクライアントとして既に利用していた[OkHttp3](https://square.github.io/okhttp/)がWebSocketもサポートしていることが分かったので、今回はこれを利用します。

OkHttpの基本的な使い方は以下の記事などが分かり易いと思います。
[OkHttp（基本的なGET・POST） - Qiita](https://qiita.com/naoi/items/8d493f00b0bbbf8a666c)

- OkHttpClientのインスタンスを作成
- Requestオブジェクトを作成

まではHTTPクライアントと同じです。

Androidアプリ-サーバー間で最低限のWebSocket通信ができるところまでを実装します。

# Android側の実装

## build.gradleに依存関係を追加
OkHttp3を追加します。
記事執筆時点で最新のバージョンを追加してます。

```gradle:app/build.gradle
dependencies {

    implementation 'androidx.core:core-ktx:1.7.0'
    implementation 'androidx.appcompat:appcompat:1.4.1'
    implementation 'com.google.android.material:material:1.5.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.3'
    testImplementation 'junit:junit:4.+'
    androidTestImplementation 'androidx.test.ext:junit:1.1.3'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.3.0'
    
    // 追加
    implementation 'com.squareup.okhttp3:okhttp:4.9.1'
}
``` 

## WebSocket作成
デモ用なので、MainActivityに全てべた書きします。

`OkHttpClient`の[newWebSocket()](https://square.github.io/okhttp/4.x/okhttp/okhttp3/-web-socket/-factory/new-web-socket/)でWebSocketを作成します。

```kotlin
fun newWebSocket(request: Request, listener: WebSocketListener): WebSocket
```
Requestで接続先のURLを指定します。
第2引数のlistenerには、[WebSocketListener](https://square.github.io/okhttp/4.x/okhttp/okhttp3/-web-socket-listener/)という抽象クラスを実装したクラスのインスタンスを渡します。

`WebSocketListener`には`onOpen()`, `onMessage()`など特定のイベント発生時に呼び出されるメソッドが用意されているので、それらをオーバーライドして実装します。

今回はその実装を含めたWebSocketクライアント用のクラスを作成し、`listener`として`this`を渡しています。
なお、`newWebSocket()`は非同期でWebSocket接続を開始し、即`WebSocket`をreturnします。
接続が成功/失敗したときは、`onOpen()`と`onFailure`で通知を受け取ることが出来ます。

最終的な実装は以下です。

```kotlin:MainActivity.kt
package com.example.websocket_android

import androidx.appcompat.app.AppCompatActivity
import android.os.Bundle
import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.Response
import okhttp3.WebSocket
import okhttp3.WebSocketListener
import okio.ByteString

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val webSocketClient = WebSocketClient()
        webSocketClient.send("Hello from Android")
    }
}

class WebSocketClient() : WebSocketListener() {
    private val ws: WebSocket

    init {
        val client = OkHttpClient()

        // 接続先のエンドポイント
        // localhostとか127.0.0.1ではないことに注意
        val request = Request.Builder()
            .url("ws://10.0.2.2:8080")
            .build()

        ws = client.newWebSocket(request, this)
    }

    fun send(message: String) {
        ws.send(message)
    }

    override fun onOpen(webSocket: WebSocket, response: Response) {
        println("WebSocket opened successfully")
    }

    override fun onMessage(webSocket: WebSocket, text: String) {
        println("Received text message: $text")
    }

    override fun onMessage(webSocket: WebSocket, bytes: ByteString) {
        println("Received binary message: ${bytes.hex()}")
    }

    override fun onClosing(webSocket: WebSocket, code: Int, reason: String) {
        webSocket.close(1000, null)
        println("Connection closed: $code $reason")
    }

    override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
        println("Connection failed: ${t.localizedMessage}")
    }
}
```

接続先のURLは`localhost`とか`127.0.0.1`にしたくなりますが、エミュレータで実行した場合それはローカルのマシンではなくエミュレータ自身を指すので接続できません。
Androidのエミュレータでは`10.0.2.2`がローカルマシンにブリッジされるようになっているので、それを指定します。
エミュレータのネットワークについては、[公式ドキュメント](https://developer.android.com/studio/run/emulator-networking?hl=ja)が詳しいです。

:::message
[OkHttpのドキュメント](https://square.github.io/okhttp/4.x/okhttp/okhttp3/-ok-http-client/#okhttpclients-should-be-shared)にて、OkHttpClientはシングルトンにしてアプリ内で共有することが推奨されています。それぞれのインスタンスがコネクションとスレッドプールを持つので、インスタンスを複数作るとコストがかかりすぎるようです。  

今回はそこまでしていませんが、実際に使うときはシングルトンにしましょう。
:::

## ネットワークの設定
ネットワーク通信を許可する設定と、暗号化なしの通信(wssではなくws)を許可する設定をマニフェストファイルに書く必要があります。

```xml:AndroidManifest.xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.websocket_android">
<!--    インターネット接続を許可-->
    <uses-permission android:name="android.permission.INTERNET" />
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.Websocketandroid"
        android:usesCleartextTraffic="true">
        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```



# サーバー側
Node.jsのWebSocket実装ライブラリである[ws](https://www.npmjs.com/package/ws)を使います。
最低限の実装です。

```shell
npm install ws
```

```js:server.js
const WebSocketServer = require('ws').Server

const wss = new WebSocketServer({
  port: 8080,
})

// connectionイベントは、ハンドシェイクが完了し、コネクションが開いたときに発火される
// イベントリスナーは、そのそれぞれのコネクションを表すwsを引数にして呼ばれる
wss.on('connection', ws => {
  
  // 適当にメッセージを送信
  ws.send('Hello from server')
  
  // メッセージを受信したら、それをログ出力
  ws.on('message', data => {
    console.log(`Received message: ${data}`)
  })
})
```

# 結果
サーバーを起動した状態でAndroidアプリを動かすと、以下のログが確認できます。

アプリ側
```
I/System.out: WebSocket opened successfully
I/System.out: Received text message: Hello from server
```

サーバー側
```
Received message: Hello from Android
```

# 接続状態を管理する
さて、無事接続が確認できましたが、WebSocketはHTTPとは違いステートフルなプロトコルです。
つまり、通信上のそれぞれのやりとりは独立しておらず、コネクションの状態はこちらで管理する必要があります。

## PingとPong

WebSocketには、標準でPingとPongという疎通確認用のコントロールフレームが定義されています。
Ping Pongは慣習として使われるメッセージとかではなく、[RFC](https://datatracker.ietf.org/doc/html/rfc6455#section-5.5)で定義された仕様です。
なので、どのWebSocketライブラリにも実装されているはずです。

RFCにある通り、Pingを受け取った側は、必ずPongを返さなければいけません。
> Upon receipt of a Ping frame, an endpoint MUST send a Pong frame in response.

なのでどのライブラリでもPongは勝手に送り返すようになっているはずですが、Pingの送り方に関してはライブラリごとに実装が異なります。

OkHttp3にはPingを自分で送るためのAPIは用意されておらず、初めにOkHttpClientを作成するときに[Pingを送るインターバルを設定]()し、後はライブラリ側で勝手に送ってくれるようです。
（ちなみにwsには自分でPingを送るための関数が用意されています。）

そしてサーバーからPingが返ってこなかった場合、`onFailure()`が呼ばれるようになっています。

## AndroidからPingを送ってみる

`OkHttpClient`作成時にpingIntervalを設定するよう、先ほどのコードを修正します。

```kotlin:MainActivity.kt
 init {
        // Pingを送るインターバルを設定したでclientを作成
        val client = OkHttpClient.Builder().pingInterval(5, TimeUnit.SECONDS).build()

        // 接続先のエンドポイント
        // localhostとか127.0.0.1ではないことに注意
        val request = Request.Builder()
            .url("ws://10.0.2.2:8080")
            .build()

        ws = client.newWebSocket(request, this)
    }
```

## サーバー側で確認
サーバーは、Pingが来たときにその旨を表示するようにします。

```js
wss.on('connection', ws => {
  // 適当にメッセージを送信
  ws.send('Hello from server')

  // メッセージを受信したら、それをログ出力
  ws.on('message', data => {
    console.log(`Received message: ${data}`)
  })

  // Pingを受け取ったときに発火するイベント
  ws.on('ping', () => {
    console.log('Received Ping. Sending Pong back...')

    // pongは、こちらで何もしなくても勝手に送り返される
  })
})
```

サーバーを再起動してAndroidアプリを実行すると、サーバー側で以下のように5秒ごとにログが表示され、Pingが送られてきていることが分かります。

``` 
Received Ping. Sending Pong back...
Received Ping. Sending Pong back...
Received Ping. Sending Pong back...
```

### 余談: pingの挙動のカスタマイズについて
OkHttp3でPingの挙動をいじれないことは[GitHubのIssue](https://github.com/square/okhttp/issues/3197)でも取り上げられており、カスタマイズしたいという要望もあるみたいですが、開発チームはそのようなAPIを提供する予定はない」と解答しています。
理由としては、同じくpingのAPIを提供しない[ブラウザのAPI](https://developer.mozilla.org/ja/docs/Web/API/WebSockets_API)に合わせているからだそうです。

# さいごに
WebSocketは通信してみるだけなら簡単ですが、接続状態の監視や再接続の処理などを考える始めると難しく感じます。このあたりの知見が欲しい…。

AndroidとKotlinについては経験が浅いので、間違い等ありましたらご指摘ください。
この記事が何かの参考になれば幸いです。

# 参考
- [WebSocket - OkHttp - OkHttp](https://square.github.io/okhttp/4.x/okhttp/okhttp3/-web-socket/)
- [okhttp/WebSocketEcho.java at master · square/okhttp · GitHub](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/WebSocketEcho.java)
- [WebSocket ping logic is not customizable · Issue #3197 · square/okhttp · GitHub](https://github.com/square/okhttp/issues/3197)
- [AndroidでWebsocket | 南国マン](https://nangokuman.com/archives/151)