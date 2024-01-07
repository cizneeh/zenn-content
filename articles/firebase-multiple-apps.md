---
title: 'Firebase Admin SDKでエミュレータと本物のプロジェクトのFirestoreに同時に接続する'
emoji: '🧪'
type: 'tech'
topics: [Firebase]
published: true
---

# この記事について

あまり無いユースケースかもしれませんが、タイトルの通り「Node.js の`firebase-admin-sdk`で、エミュレータの firestore と本物の Firebase プロジェクトの firestore を同時に扱いたい」ということが先日ありました。

ワークアラウンド的な方法にはなりますが、実現出来たのでその紹介です。

Firebase をすでに使っている人向けの記事であり、Firebase emulator や Admin SDK の説明は省略します。

## 使用する環境・バージョン

```json
{
  "engines": {
    "node": "18"
  },
  "dependencies": {
    "firebase-admin": "^11.11.1"
  }
}
```

## 結論

以下のように、エミュレータに接続したい app の初期化のタイミングで（コード中で）、 `FIRESTORE_EMULATOR_HOST`に値をセットすることで同時に接続できます。

```ts
const realApp = initializeApp(
  {
    projectId: 'real-project',
  },
  'real-app',
)
// 本物のプロジェクトのfirestore
const realFirestore = getFirestore(realApp)

process.env.FIRESTORE_EMULATOR_HOST = '127.0.0.1:8080'
const emuApp = initializeApp(
  {
    projectId: 'emulator',
  },
  'emulator-app',
)
// エミュレータのfirestore
const emuFirestore = getFirestore(emuApp)
```

## 詳細

### 複数 app の初期化

まず、複数の app の初期化については以下の公式ドキュメントに記載があります。

https://firebase.google.com/docs/admin/setup?hl=ja#initialize-multiple-apps

`initializeApp`は通常引数の config のみを指定して呼び出すことが多い（それが default の app になる）ですが、第 2 引数に app 名を指定することで、複数の app を初期化することが出来ます。

そして`getFirestore`などは引数無しで呼び出すとデフォルトの app に対して firestore を取得しますが、引数に特定の app インスタンス を渡すことでその app に対しての firestore を取得することが出来ます。

```ts
// 第2引数のapp名は無しで、configのみで初期化するとデフォルトのappになる
initializeApp(defaultAppConfig)
// getFirestore()などを引数無しで呼び出すと、そのデフォルトのappに対してfirestoreを取得する
const defaultFirestore = getFirestore()

// 第2引数にapp名を指定して、別のconfigとともに初期化することも出来る
const otherApp = initializeApp(otherAppConfig, 'other-app')
// getFirestore()などの引数にappを渡すと、そのappに対してfirestoreを取得できる
const otherFirestore = getFirestore(otherApp)
```

### エミュレータへの接続

https://firebase.google.com/docs/emulator-suite/connect_firestore?hl=ja#admin_sdks

そして Firestore のエミュレータに接続したい場合、以下のように環境変数`FIRESTORE_EMULATOR_HOST`に値をセットすると勝手に繋ぎにいってくれます。

```ts
export FIRESTORE_EMULATOR_HOST="127.0.0.1:8080"
```

ただし、この環境変数が設定されていると、すべての`getFirestore()`呼び出しでエミュレータと接続する firestore インスタンスを取得しようとしてしまいます。

片方は本物のプロジェクト、もう片方はエミュレータに接続する方法が無いか調べたところ、全く同じことをやってくれていた issue がありました。

https://github.com/firebase/firebase-admin-node/issues/776#issuecomment-1322803407

本物のプロジェクトに接続する firestore インスタンスを取得した後に、ランタイムで環境変数を設定してエミュレータに接続する firestore インスタンスを取得するという方法です。

```ts
const realApp = initializeApp(
  {
    projectId: 'real-project',
  },
  'real-app',
)
// 本物のプロジェクトのfirestore
const realFirestore = getFirestore(realApp)

process.env.FIRESTORE_EMULATOR_HOST = '127.0.0.1:8080'
const emuApp = initializeApp(
  {
    projectId: 'emulator',
  },
  'emulator-app',
)
// エミュレータのfirestore
const emuFirestore = getFirestore(emuApp)
```

注意として、`FIRESTORE_EMULATOR_HOST`は app の初期化時ではなく`getFirestore()`の呼び出し時に参照されるようなので、環境変数のセット前に`getFirestore()`まで呼び出す必要があります。

ランタイムで環境変数を書き換えるのでプロダクションコード中では使いたくないですが、テストコードや簡単なスクリプトなどで役に立つかもしれません。

## まとめ

そんなに需要があるユースケースでは無い気がしますが、一応本物プロジェクトとエミュレータに同時に接続出来るよという話でした。

試していないですが、firestore 以外のサービスでも出来るかもしれません。
