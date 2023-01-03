---
title: "Markdownで書いたノートをNotionのデータベースに移行する"
emoji: "🐥"
type: "tech"
topics: [JavaScript, Notion]
published: true
---

# やりたいこと

自分は技術的な勉強をするときや本を読むときなどMarkdownファイルにメモを取ってローカルに保存していたのですが、それだとPCを買い替えた時とかに移行がめんどくさいし、別の端末からもメモを確認したい時もあります。

[Notion API](https://developers.notion.com/)を使って大量のMarkdownのメモ達をNotionのデータベースに移行することができたので、この記事ではそのメモ書きとNotion APIの簡単な紹介を行います。[公式から提供されているJavaScriptのSDK](https://www.npmjs.com/package/@notionhq/client)を使います。

なお、記事中で「ノート」という単語がこれからも出てきますが、これは一般名詞としての「ノート」であり、Notionの用語とかではないのでご注意。

## やること

Notionにデモ用のデータベースを用意し、そこに実際にMarkdownファイルのノートをAPIを利用して移行します。

データベースは以下のようにName(ノートのタイトル)とTagsをプロパティとして持ちます。この後のデモ用に`foo`というタイトルのノートを1つ入れておきました。

![](/images/markdown-to-notion-db/db-first.png)

プロジェクトのディレクトリを以下のように用意します。`notes`配下にある`hoge.md`と`fuda.md`をデータベースに追加していきます。

```
.
├── notes
│   ├── fuga.md
│   └── hoge.md
├── package.json
└── src
    ├── index.js
    └── markdown.js
```

hoge.mdの中身はこんな感じで、Front Matterの形式でタグが書かれており、これをNotionのデータベースのTagsに登録します。

```markdown:hoge.md
---
tags:
  - JavaScript
  - Notion
---

# HOGE

[何かリンク](https://zenn.dev/cizneeh)

## test

hoge

- ho
- ge

```javascript
function doSomething(){
  // do something
}

doSomething()
\```

```

## 前準備

Notion APIを使うためには、まず以下の準備をする必要があります。[公式のドキュメント](https://developers.notion.com/docs/create-a-notion-integration)が詳しいので、細かい手順は割愛します。

- Notion Integrationを作成し、トークン（APIキー）を取得
- 操作したいデータベース（上の「ノート」データベース）を作成したIntegrationに共有
- データベースのIDを取得

取得したトークンとデータベースのIDは環境変数に持たせておきます。

```shell
export NOTION_KEY=secret_...
export NOTION_DATABASE_ID=...
```

## データベースを取得してみる

ノートを追加していく前に、試しにAPIを使ってデータベースの情報を取得してみます。

まず公式のSDKをインストール。

```shell
npm i @notionhq/client
```

`notion.databases.retrieve()`でデータベースの情報を取得できます。

```javascript: src/index.ts
import { Client } from '@notionhq/client'

const token = process.env.NOTION_TOKEN
const databaseId = process.env.NOTION_DATABASE_ID

async function main() {
  const notion = new Client({ auth: token })

  const res = await notion.databases.retrieve({ database_id: databaseId })
  console.dir(res, { depth: null })
}

main()
```

すると以下のようにデータベースの情報が得られます。

```js
{
  object: 'database',
  id: 'xxxxxx',
  cover: null,
  icon: null,
  created_time: '2022-12-31T04:13:00.000Z',
  created_by: { object: 'user', id: 'xxxxxx' },
  last_edited_by: { object: 'user', id: 'xxxxxx' },
  last_edited_time: '2023-01-01T13:02:00.000Z',
  title: [
    {
      type: 'text',
      text: { content: 'ノート', link: null },
      annotations: {
        bold: false,
        italic: false,
        strikethrough: false,
        underline: false,
        code: false,
        color: 'default'
      },
      plain_text: 'ノート',
      href: null
    }
  ],
  description: [],
  is_inline: true,
  properties: {
    Tags: {
      id: 'wQ%3En',
      name: 'Tags',
      type: 'multi_select',
      multi_select: {
        options: [
          {
            id: 'f4aacbdf-a8d6-4ef9-a3ed-6c8f79422954',
            name: 'TypeScript',
            color: 'gray'
          },
          {
            id: '9a30eb2b-4919-41e3-93e6-d87d3781b099',
            name: 'Foo',
            color: 'orange'
          }
        ]
      }
    },
    Name: { id: 'title', name: 'Name', type: 'title', title: {} }
  },
  parent: { type: 'page_id', page_id: 'xxxxxx' },
  url: 'https://www.notion.so/xxxxxx',
  archived: false
}
```

色々書かれていますが、`properties`を見てみましょう。これはその名の通りデータベースの各プロパティ（「ノート」データベースの場合NameとTags）をキーとして持つオブジェクトですが、以下のような特徴があります。

まず、`type`は各プロパティのタイプ（`Number`や`Multi-Select`などNotion上でのプロパティのタイプ）を示すものですが、その中で`title`という特別な`type`が存在します。データベースは必ず一つ`type`が`title`となっているプロパティを持ち、このプロパティの値がそのデータベースに入っているそれぞれのページのタイトルに相当します。

上の例では、`Name`がそれに相当します。Notionでデータベースを作った時に勝手に一番左のカラムに設定されるヤツです。デフォルトで確かNameだったはず。

また、各プロパティのオブジェクト（上ではNameとTags）は、`type`の値がキーとなるプロパティを持ちます。文章にするととてもややこしいですが、例えば`Name`オブジェクトは`type: 'title'`というプロパティを持ち、さらに`title: {}`というプロパティを持っています。

この`title: {}`には、そのプロパティの何らかの情報が入るようになっているようです。例えば、`Tags`オブジェクトの`multi-select`プロパティには、選択肢の情報が入っていることがわかります。

Notion APIでは色々なところでこのパターンが出てくるらしいです。

## ノートの内容を読み込む

さて、それでは本題のノートの追加です。まずMarkdownファイルからファイルの内容を読み込む必要があります。`gray-matter`というnpmパッケージを使用し、各ノートの内容を読み込むモジュールを作成します。

https://www.npmjs.com/package/gray-matter

gray-matterはFront Matterで書いた部分の情報も構造データとして取得できるので便利です。

```js:src/markdown.js
import { readFileSync, readdirSync } from 'fs'
import matter from 'gray-matter'

export function getAllNotes(path) {
  const fileNames = readdirSync(path)

  const notes = fileNames.map(name => {
    const content = readFileSync(path.join(path, name))
    const matterResult = matter(content)

    return {
      // Notionのページのタイトルになる
      name: name.replace(/.md$/, ''),
      // NotionのページのTagsになる
      tags: matterResult.data.tags,
      // Notionのページの中身になるが、このままではただの文字列
      body: matterResult.content,
    }
  })

  return notes
}
```

しかし、このままでは問題があります。`name`と`tags`（つまりNotionのページのプロパティの部分）はそのままでいいのですが、Notionページの中身となる`body`の部分をただのマークダウン文字列からNotion APIの使用に則った構造データに変換する必要があります。

これも詳しくはドキュメントにありますが、ページの中身部分は以下のようにblock objectのリストとして表されます。Notion ページ中の箇条書きやパラグラフ、引用などは全てblockです。

https://developers.notion.com/docs/working-with-page-content

```js
// ドキュメントにある例
{
  "object": "block",
  "id": "380c78c0-e0f5-4565-bdbd-c4ccb079050d",
  "type": "paragraph",
  "created_time": "",
  "last_edited_time": "",
  "has_children": true,

  "paragraph": {
    "text": [/* details omitted */],
    "children": [
      {
        "object": "block",
        "id": "6d5b2463-a1c1-4e22-9b3b-49b3fe7ad384",
        "type": "to_do",
        "created_time": "",
        "last_edited_time": "",
        "has_children": false,
  
        "to_do": {
          "text": [/* details omitted */],
          "checked": false
        }
      }
    ]
  }
}
```

調べてみると、@tryfabric/martianという、マークダウン文字列をNotion APIのblock objectに変換してくれるライブラリがすでにあるようだったので、こちらを利用させてもらいました。

https://www.npmjs.com/package/@tryfabric/martian

こちらを使って以下のように書き換えます。

```js:src/markdown.js
import { readFileSync, readdirSync } from 'fs'
import path from 'path'
import matter from 'gray-matter'
import { markdownToBlocks } from '@tryfabric/martian'

export function getAllNotes(notePath) {
  const fileNames = readdirSync(notePath)

  const notes = fileNames.map(name => {
    const content = readFileSync(path.join(notePath, name))
    const matterResult = matter(content)

    return {
      name: name.replace(/.md$/, ''),
      tags: matterResult.data.tags,
      // block objectsに変換
      body: markdownToBlocks(matterResult.content),
    }
  })

  return notes
}
```

ちなみに上の方で示したhoge.mdの中身は以下のような構造データになります。

```js
[
  {
    object: 'block',
    type: 'heading_1',
    heading_1: {
      rich_text: [
        {
          type: 'text',
          annotations: {
            bold: false,
            strikethrough: false,
            underline: false,
            italic: false,
            code: false,
            color: 'default'
          },
          text: { content: 'HOGE', link: undefined }
        }
      ]
    }
  },
  {
    object: 'block',
    type: 'paragraph',
    paragraph: {
      rich_text: [
        {
          type: 'text',
          annotations: {
            bold: false,
            strikethrough: false,
            underline: false,
            italic: false,
            code: false,
            color: 'default'
          },
          text: {
            content: '何かリンク',
            link: { type: 'url', url: 'https://zenn.dev/cizneeh' }
          }
        }
      ]
    }
  },
  {
    object: 'block',
    type: 'heading_2',
    heading_2: {
      rich_text: [
        {
          type: 'text',
          annotations: {
            bold: false,
            strikethrough: false,
            underline: false,
            italic: false,
            code: false,
            color: 'default'
          },
          text: { content: 'test', link: undefined }
        }
      ]
    }
  },
  {
    object: 'block',
    type: 'paragraph',
    paragraph: {
      rich_text: [
        {
          type: 'text',
          annotations: {
            bold: false,
            strikethrough: false,
            underline: false,
            italic: false,
            code: false,
            color: 'default'
          },
          text: { content: 'hoge', link: undefined }
        }
      ]
    }
  },
  // 以下略
]
```

## ノートをNotionのデータベースに追加

さて、これでデータの準備は整ったので、いよいよAPIからデータベースにページを追加します。`notion.pages.create()`でデータベースにページを追加します。

```javascript:src/index.js
import { Client } from '@notionhq/client'
import { getAllNotes } from './markdown.js'

const token = process.env.NOTION_TOKEN
const databaseId = process.env.NOTION_DATABASE_ID

async function main() {
  const notion = new Client({ auth: token })

  const notes = getAllNotes('notes')
  const failedNotes = []

  for (const note of notes) {
    try {
      await notion.pages.create({
        parent: { database_id: databaseId },
        // 各ノート（ページ）のプロパティ
        properties: {
          // プロパティ名はcase sensitiveっぽいので注意
          Name: {
            type: 'title',
            title: [{ text: { content: note.name } }],
          },
          Tags: {
            type: 'multi_select',
            multi_select: note.tags.map(tag => ({ name: tag })),
          },
        },
        // ページの中身
        children: note.body,
      })
    } catch (e) {
      console.error(`${note.name}の追加に失敗: `, e)
      failedNotes.push(note.name)
    }
  }

  console.log('ページ作成に失敗したノート: ', failedNotes)
}

main()
```

エラー処理も入れてます。実際自分が実際に大量のノートを移行した際には、無効なリンクが含まれていたりファイルが長すぎた場合にちょくちょく登録が失敗したのでいくつかは手動でNotionにインポートしました。

これで無事Notionのデータベースに各ノートに応じたページが追加されました。タグもちゃんと登録されていますね。

![](/images/markdown-to-notion-db/db-pages-created.png)

ノートの中身もちゃんとマークダウンに応じたNotionのページになっています。ただし、APIからのページ追加だとコードブロックに色がつきませんでした。

![](/images/markdown-to-notion-db/page-content.png)

## 感想

Notion APIは初めて触りましたが、いろんなことができそうだなーと思いました。ただし、まだ公開からあまり時間が経っていないので機能はやや制限されている印象です。

仕事でもプライベートでもNotionは使っているので、今後も情報はキャッチアップしていきたいです。

今回のコードは以下のリポジトリにあるので、良ければ参考にしてください。
https://github.com/cizneeh/markdown-to-notion-demo
