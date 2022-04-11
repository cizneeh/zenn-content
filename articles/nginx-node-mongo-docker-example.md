---
title: "Dockerでnginx + Node.js + MongoDBの環境を用意する"
emoji: "✨"
type: "tech"
topics: ["docker", "nginx", "nodejs", "mongodb"]
published: true
---

# Dockerでnginx + Node.js + MongoDBの環境を用意する

タイトルの通り、nginx(リバースプロキシ)、Node.js、MongoDBの環境をDocker + Docker Composeで用意します。  
全て公式のイメージを使用し、基本的に公式のガイドに従って準備します。

バージョン
```shell
❯ docker -v                      
Docker version 20.10.10, build b485636

❯ docker-compose -v              
docker-compose version 1.29.2, build 5becea4c
```

## ディレクトリ構成

```
.
├── docker-compose.yml
├── mongo
│   ├── init.js # DB初期化用スクリプト
│   └── mongo-data # データ保存用ディレクトリ（バインドマウントする）
├── nginx
│   ├── Dockerfile
│   └── conf
│       └── proxy.conf
└── server
    ├── Dockerfile
    ├── node_modules
    ├── package-lock.json
    ├── package.json
    └── src # アプリケーションコード
```
## MongoDB
[公式イメージ](https://hub.docker.com/_/mongo)を利用します。
Dockerfileは用意せず、公式イメージをそのまま使います。

```yml:docker-compose.yml(mongo部分だけ)
  mongo:
    image: mongo
    container_name: mongo
    environment:
      MONGO_INITDB_DATABASE: admin
    volumes:
      - ./mongo/init.js:/docker-entrypoint-initdb.d/init.js:ro
      - ./mongo/mongo-data:/data/db
    # Start mongo with authentication enabled
    command: [mongod, --auth]
```
MongoDBはデフォルトだと認証が無効になっているので、`command: [mongod, --auth]`で認証を有効にして起動しています。

また、以下のような初期化用のスクリプトを用意します。

```js:mongo/init.js
db.createUser({
  user: 'echizen',
  pwd: 'password',
  roles: [
    {
      role: 'readWrite',
      db: 'nginx-node-mongo-docker-example',
    },
  ],
})
```

この初期化用のスクリプトをコンテナ内の`/docker-entrypoint-initdb.d/`にマウントしています。

どういうことかというと、この公式イメージのコンテナは初めて立ち上がったとき、`/docker-entrypoint-initdb.d/`以下にある`.js`もしくは`.sh`ファイルを実行してくれるようになっています。  
また、スクリプトを実行する際に使用するデータベースを`MONGODB_INITDB_DATABASE`という環境変数で指定できます。

ここでは`MONGODB_INITDB_DATABASE`に`admin`(デフォルトの認証情報用DB)を指定しているので、そこに`echizen`というユーザーが作成されます。

また、`MONGODB_INITDB_ROOT_USERNAME`と`MONGODB_INITDB_ROOT_PASSWORD`という環境変数を指定することで、初期化時にroot権限のユーザーを`admin`データベースに作ることもできます。詳しくは[公式イメージ](https://hub.docker.com/_/mongo)のページを参照。

## Node.js

こちらも[公式イメージ](https://hub.docker.com/_/node)を利用します。

### アプリケーションコード

expressと、[MongoDB公式のNode.jsドライバー](https://www.npmjs.com/package/mongodb)を利用します。([APIドキュメント](https://mongodb.github.io/node-mongodb-native/4.5/))

```shell
npm init -y
npm install express
npm install mongodb
```

[ES Modules](https://nodejs.org/api/esm.html#enabling), [Top level await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await#top_level_await)を使いたいので、`package.json`の`type`フィールドに`module`を指定します。

```json:server/json
{
  "name": "server",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "express": "^4.17.3",
    "mongodb": "^4.5.0"
  }
}
```

アプリケーションコードを用意します。

```js:server/src/app.js
import express from 'express'
import { MongoClient } from 'mongodb'

const PORT = 8080
const HOST = '0.0.0.0'

const app = express()

app.get('/', async (req, res) => {
  const result = await collection.find({}).toArray()
  res.json(result)
})

app.listen(8080, HOST, () => {
  console.log(`Listening on port ${PORT}`)
})

const URL = 'mongodb://echizen:password@mongo/nginx-node-mongo-docker-example?authSource=admin'
const client = new MongoClient(URL)

try {
  await client.connect()
  console.log('Succesfully connected to mongo')
} catch (e) {
  console.error(e)
}
const db = client.db()

// Prepare initial data
const doc1 = { name: 'Echizen', age: 24 }
const doc2 = { name: 'Bob', age: 32 }

const collection = db.collection('test-collection')
await collection.insertMany([doc1, doc2])
```
URLで`authSource=admin`を指定して、先ほどの初期化用スクリプトで作成されるユーザーで認証しています。

### Docker化

次に、[公式のガイド](https://nodejs.org/ja/docs/guides/nodejs-docker-webapp/)を参考にDocker化します。

```Dockerfile:server/Dockerfile
FROM node:16

# アプリケーションディレクトリを作成する
WORKDIR /usr/src/app

# キャッシュを利用するために、package.jsonとpackage-lock.jsonのみをコピーし、
# 依存関係を先にインストール
COPY package*.json ./
RUN npm install

# アプリケーションコードをコンテナにコピー
COPY . .

EXPOSE 8080
CMD [ "node", "src/app.js" ]
```

`node_modules`等を無視するために、上の`Dockerfile`と同じディレクトリに`.dockerignore`を以下の内容で作成します。

```dockerignore:server/.dockerignore
node_modules
npm-debug.log
```

## nginx

こちらも[公式イメージ](https://hub.docker.com/_/nginx)を使って、[nginx公式のガイド](https://www.nginx.co.jp/blog/deploying-nginx-nginx-plus-docker/)に従って準備します。

### 設定ファイル 

デフォルトではコンテナ内に`/etc/nginx/nginx.conf`と`/etc/nginx/conf.d/default.conf`という二つが用意され、`nginx.conf`から`default.conf`を読み込むようになっていますが、`default.conf`を削除してリバースプロキシ用の設定ファイルを代わりに用意します。

リクエストは全て`server`(Node.jsが動いているコンテナ)に飛ばします。

```nginx:nginx/conf/proxy.conf
server {
  listen 80;
  location / {
    proxy_pass http://server:8080/;
  }
}
```

### Dockerfile

```Dockerfile: nginx/Dockerfile
FROM nginx:1.21

# デフォルトの設定を削除
RUN rm /etc/nginx/conf.d/default.conf

# 作成した設定ファイルをコピー
COPY conf/proxy.conf /etc/nginx/conf.d
```

## docker-compose.yml

最後に`docker-compose.yml`を準備します。

```yml:docker-compose.yml
services:
  nginx:
    build: ./nginx
    container_name: nginx
    depends_on:
      - server
    ports:
      - 8080:80

  server:
    build: ./server
    container_name: server
    depends_on:
      - mongo

  mongo:
    image: mongo
    container_name: mongo
    environment:
      MONGO_INITDB_DATABASE: admin
    volumes:
      - ./mongo/init.js:/docker-entrypoint-initdb.d/init.js:ro
      - ./mongo/mongo-data:/data/db
    # Start mongo with authentication enabled
    command: [mongod, --auth]
```

nginx以外は外にポートを公開していません。

docker-composeで作ったコンテナはデフォルトですべて`<親ディレクトリ名>_default`というネットワークに所属し、コンテナ名で通信できます。

## 動かしてみる

```shell
❯ docker-compose up -d
```

```shell
❯ curl http://localhost:8080/ | jq .
[
  {
    "_id": "625103c4f9d2309deaad919e",
    "name": "Echizen",
    "age": 24
  },
  {
    "_id": "625103c4f9d2309deaad919f",
    "name": "Bob",
    "age": 32
  },
]
```

## 最後に

今回のソースコードは全て[GitHub](https://github.com/cizneeh/nginx-node-mongo-docker-example)に置いてあります。

まだまだ勉強中なので、「こうした方がいい」等あったら是非教えてください。
この記事が何かの参考になったら幸いです。

## 参考
- [LukeMwila/multi-container-nginx-react-node-mongo](https://github.com/LukeMwila/multi-container-nginx-react-node-mongo)
- [【MongoDB】MongoDBのDockerコンテナ起動時に認証用ユーザーを作成する - Qiita](https://qiita.com/homoluctus/items/038dc08fca6405813e0b)
