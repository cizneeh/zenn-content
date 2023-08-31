---
title: "Firebase Functionsのテストを書こう"
emoji: "🧪"
type: "tech"
topics: [Firebase, Test]
published: false
publication_name: "terass_dev"
---

# Cloud Functions for Firebaseのテストを書く

こんにちは。[株式会社TERASS](https://terass.com/)で主にフロントエンドエンジニアをしている越前です。

弊社ではCloud Functions for Firebaseをよく利用しているのですが、functionのテストを書くにあたって`firebase-functions-test`という公式のテスト用SDKが便利だったので紹介します。

https://firebase.google.com/docs/functions/unit-testing?hl=ja

https://www.npmjs.com/package/firebase-functions-test

`firebase-functions-test`とfirebaseのエミュレータを使ってテストを書いて、モックなどをなるべく使わない本物に近い環境のテストをローカルで実行しよう！という話です。

## firebase functionのテストの難しいところ

今回は例として`onUpdateUser`という、usersコレクションのドキュメントが更新された時に発火する、firestoreトリガーのfunctionのテストを書いていきます。

ドキュメントには`__state`というフィールドがあり、これが`default`から`updated`に変更された際に、外部のAPIにリクエストを送るような関数です。

```ts
export const onUpdateUser = region('asia-northeast1')
  .firestore.document('users/{id}')
  .onUpdate(async (change) => {
    const before = change.before.data()
    const after = change.after.data()
    // stateがupdatedになっていたらAPIに投げる
    if (before.__state === 'default' && after.__state === 'updated')
      await callApi(after)

    // stateをdefaultに戻す
    await change.after.ref.update({
      __state: 'default',
    })
  })
``` 

特別なことをせずとも、firebaseのエミュレーターを起動して、テストコード中でfirestoreに書き込んで、functionが実行された結果をチェックして…のようにテストを書くことも一応できます。

しかし、その場合テストランナーとは別のプロセスでfunctionが勝手に実行されるため、テストコード内でfunctionの終了をいい感じに待つ手段がありません。

```zsh
# firestoreとfunctionsのエミュレータを起動 
firebase emulators:start --only functions,firestore
```

```ts
test('テスト', async () => {
  // firestoreのデータを更新する
  getFirestore().collection('users').doc('user1').update({
  // 更新するデータ
  })

  // functionが発火するが、その終了を待つ手段が無い！
})
```

`firebase-functions-test`は以下のようにテスト対象のfunctionをラップして、テストコード内でfunctionを明示的に実行できます。

```ts
const wrapped = tester.wrap<Change<QueryDocumentSnapshot<User>>>(onUpdateUser)
await wrapped(changeData)
```


## 前準備

テストを動かすための準備です。

まず、firestoreはローカルのエミュレータで動かしておきます。なお、functionはテストコード内で実行するため、エミュレータとして起動しておく必要はありません。

```zsh
firebase emulators:start --only firestore
```

`firebase-functions-test`は、オンラインモードとオフラインモードの2つの方法で使うことができます。

https://firebase.google.com/docs/functions/unit-testing?hl=ja#initializing

簡単にいうと、オンラインモードはfirestoreへの書き込みなどが実際に行われる（実際にfirebaseとのやり取りが行われる）テストで、オフラインモードはfirebaseとのやりとりをすべてスタブして行うテストです。

「オンラインモード」とはいっても、やりとりをする対象は本物のfirebaseプロジェクトではなくエミュレータでも大丈夫です。公式でもこちらが推奨されているので、基本的にエミュレータを併用してオンラインモードでテストするのが良いと思われます。


## テストを書く

ここからは実際にテストを書いていきます。テストフレームワークはvitest、言語はTypeScriptです。

```zsh
yarn add -D firebase-functions-test vitest
```

まずSDKを初期化します。

```ts
import functionTest from 'firebase-functions-test'

// ここでパラメータを指定して初期化すると、オンラインモードでのテストになる
export const tester = functionTest({
  projectId: '<project-id>',
})
```

`wrap()`でテスト対象のfunctionをラップします。その際、型パラメータとして`onUpdateUser`のコールバックの引数の型を渡すこともできます。

```ts
const wrapped = tester.wrap<Change<QueryDocumentSnapshot<User>>>(onUpdateUser)
```

そして、このwrapされたfunctionをテストコード内で実行して、結果を確認するという流れです。

この`wrapped`を実行する際に、functionが実行される際の引数(`onUpdate`の場合は`Change`)を渡すことが出来るのですが、そういったfirestore関連のデータを作るためのAPIも（`makeDocumentSnapshot`や`makeChange`）もSDKに用意されています。

それらを使って以下のようにテストを書くことができます。

```ts
// エミュレータと通信出来るようにポートを指定する
beforeAll(() => {
  vi.stubEnv('FIRESTORE_EMULATOR_HOST', '127.0.0.1:8080')
})

test('__stateがdefaultのままのとき、APIリクエストが飛ばない', async () => {
  // APIコールをモック
  const apiSpy = vi.spyOn(api, 'callApi')

  // 変更前のデータのスナップショットを作る
  const beforeData: User = {
    __state: 'default'
    // 他のデータ
  }
  const beforeSnap = tester.firestore.makeDocumentSnapshot(
    beforeData,
    `user/user1`,
  ) as QueryDocumentSnapshot<User>

  // 変更後のデータのスナップショットを作る
  const afterData = {
    __state: 'default'
    // 他のデータ
  }
  const afterSnap = tester.firestore.makeDocumentSnapshot(
    afterData,
    `user/user1`,
  ) as QueryDocumentSnapshot<User>

  // 変更前後のスナップショットからchangeを作る
  const change = await tester.makeChange<QueryDocumentSnapshot<User>>(
    beforeSnap,
    afterSnap,
  )
  
  // changeを渡してfunctionを実行
  // functionの終了をawait出来る！
  await wrapped(change)

  // APIリクエストが飛ばないことを確認する
  expect(apiSpy).not.toHaveBeenCalled()
})
```

しかし、このテストを実行すると以下のエラーが出ます。

```shell
NOT_FOUND: no entity to update
```

`makeDocumentSnapshot`の第二引数ではドキュメントのパスを渡すのですが、ここでは`users/user1`など適当なドキュメントのidを指定しているのが原因です。

`makeDocumentSnapshot`はsnapshotを作るだけであり、実際にfirestoreにデータを書き込むわけではありません。なので、`onUpdateUser`の中の以下の箇所で、snapshotの参照先のドキュメントが無いよというエラーになります。

```ts
// stateをdefaultに戻す
// ここで、change.after.refのドキュメントは実際には存在しない
await change.after.ref.update({
  __state: 'default',
})
``` 

なので、`makeDocumentSnapshot`を呼ぶ前に、実際にドキュメントを作成し、そのidをパスとして渡してあげます。

データからsnapshotやchangeを作ったりするもろもろの処理はテスト中に何回も行うので、以下のようなヘルパー関数を用意しておくことにします。

```ts
// 任意のデータからsnapshotとchangeを作って返すヘルパー
const makeUserChange = async (
  tester: FeaturesList,
  beforeData: User,
  afterData: User,
) => {
  // 実際にfirestoreエミュレータにデータを書き込む
  const userId = (await db.user.add(beforeData)).id
  
  // パスにそのidを指定してスナップショットを作る
  const beforeSnap = tester.firestore.makeDocumentSnapshot(
    beforeData,
    `user/${userId}`,
  ) as QueryDocumentSnapshot<User>

  const afterSnap = tester.firestore.makeDocumentSnapshot(
    afterData,
    `user/${userId}`,
  ) as QueryDocumentSnapshot<User>

  const change = tester.makeChange<QueryDocumentSnapshot<User>>(
    beforeSnap,
    afterSnap,
  )
  
  // 変更前と変更後のスナップショット、そしてchangeを返す
  return { beforeSnap, afterSnap, change }
}
```

これで変更対象のドキュメントがfirestoreのエミュレータに実際にある状態でテストが行うことができます。（エミュレータのuiを開いてテストを実行すると、実際に書き込まれているのが確認できます）。

```ts
test('__stateがdefaultのままのとき、APIリクエストが飛ばない', async () => {
  // APIコールをモック
  const apiSpy = vi
    .spyOn(api, 'callApi')

  const beforeData: User = {
    __state: 'default'
    // 他のデータ
  }
  const afterData = {
    __state: 'default'
    // 他のデータ
  }
  const { change } = await makeUserChange(tester, beforeData, afterData)
  await wrapped(change)

  // APIリクエストが飛ばないことを確認
  expect(apiSpy).not.toHaveBeenCalled()
})
```

## firestoreのクリーンアップと直列での実行

このテストでは実際にfirestore（のエミュレータ）に実際に書き込んでテストを行いますが、やはりテストごとにデータを毎回消去したいところです。

firestoreのsdkにはコレクションのデータをすべて消すAPIはありませんが、エミュレータに限っては、それを行うための以下のようなRESTエンドポイントがローカルに用意されています。

```
http://HOST:PORT/emulator/v1/projects/PROJECT_ID/databases/(default)/documents
```

https://cloud.google.com/firestore/docs/emulator#clear_emulator_data

afterEachでこのエンドポイントにリクエストしてあげれば、エミュレータのデータを毎回消去することができます。

```ts
afterEach(async () => {
  // テスト後にfirestoreのデータを削除する
  await fetch(
    `http://${process.env.FIRESTORE_EMULATOR_HOST}/emulator/v1/projects/${process.env.GCLOUD_PROJECT}/databases/(default)/documents`,
    { method: 'DELETE' },
  )
})
```

また、テストが並列で実行されると、別のテストで書き込まれたデータが影響してしまうため、直列で実行するようにします。vitestの場合は、以下のように設定ファイルで`threads:false`を指定します。

```ts:vitest.config.ts
import { configDefaults, defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    exclude: [...configDefaults.exclude],
    threads: false,
  },
})
```

https://vitest.dev/config/#threads

テストの実行速度が遅くならないか心配ではありましたが、自分は今のところこれで困ったことはありません。

## 最終形

最終的なテストファイルが以下です。

```ts:onUpdateUser.test.ts
import functionTest from 'firebase-functions-test'
import { afterAll, afterEach, beforeAll, vi } from 'vitest'
import * as api from '../../services/api'

export const tester = functionTest({
  projectId: '<project-id>',
})

beforeAll(() => {
  vi.stubEnv('FIRESTORE_EMULATOR_HOST', '127.0.0.1:8080')
})

afterAll(() => {
  vi.unstubAllEnvs()
})

afterEach(async () => {
  // テスト後にfirestoreのデータを削除する
  await fetch(
    `http://${process.env.FIRESTORE_EMULATOR_HOST}/emulator/v1/projects/${process.env.GCLOUD_PROJECT}/databases/(default)/documents`,
    { method: 'DELETE' },
  )
})

test('__stateがdefaultのままのとき、APIリクエストが飛ばない', async () => {
  // APIコールをモック
  const apiSpy = vi
    .spyOn(api, 'callApi')

  const beforeData: User = {
    __state: 'default'
    // 他のデータ
  }
  const afterData = {
    __state: 'default'
    // 他のデータ
  }
  const { change } = await makeUserChange(tester, beforeData, afterData)
  await wrapped(change)

  // APIリクエストが飛ばないことを確認
  expect(apiSpy).not.toHaveBeenCalled()
})

test('__stateがdefault -> updatedに更新されたとき、APIリクエストが飛ぶ', async () => {
  // APIコールをモック
  const apiSpy = vi.spyOn(api, 'callApi')

  const beforeData: User = {
    __state: 'default'
    // 他のデータ
  }
  const afterData = {
    __state: 'updated'
    // 他のデータ
  }
  const { afterSnap, change } = await makeV1ImportChange(
    tester,
    beforeData,
    afterData,
  )
  await wrapped(change)

  // APIリクエストが飛ぶことを確認
  expect(apiSpy).toHaveBeenCalled()

  // stateがdefaultに戻ることを確認
  const data = (await afterSnap.ref.get()).data()
  expect(data?.__state).toBe('default')
})

```

実際には、`afterEach`などに書かれている処理は別のファイルにおいて、フレームワークの設定でsetupファイルなどとして指定すると良いでしょう。

## コマンド

エミュレータの起動とテストのコマンドが別だと不便なので、`emulators:exec`を使った以下のようなコマンドを用意しておくと便利です。

```json:package.json
  "scripts": {
    "test": "yarn firebase emulators:exec --only firestore 'yarn vitest'",
  },
```

https://firebase.google.com/docs/emulator-suite/install_and_configure?hl=ja#startup

## 他のトリガーのテスト

今回はfirestoreトリガーのfunctionのテストを書きましたが、authトリガーやhttp関数のテストも書くことができます。ただし、`beforeCreate`などのBlocking Functionにはどうやら対応していないようでした。

## まとめ

`firebase-functions-test`とfirebaseのエミュレータを利用すれば、モックなどをなるべく使わず本番に近い環境でのテストをローカルで実行することができて便利という話でした。

この記事がどなたかの参考になれば幸いです。
