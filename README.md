# nodejs-import-playground

CommonJS、ES Modules、Native ESMなどがよくわかっていないので、理解したい。

今困っていることは、antlr4からインポートしようとすると、型定義ファイルがないと言われてコンパイルエラーになること。

package.jsonの"types"ではなく、"exports > . > node"のフィールドから読み取っているように見える。生成するコードがどの形式になるかによって、どこからインポートするのかが変わるのだと思う。どの形式の場合にどのフィールドが使われるのかも理解したい。

## メモ

前提

- ブラウザとNode.jsで動かすためのライブラリを作っている
- Native ESMを使う予定

まず、tsconfig.jsonの関係ありそうなオプションを確認する。

### `target`

コンパイル先のJavaScriptのバージョンを表す。`es3`、`es5`、`es2021`、`esnext`などを指定する。JavaScriptのバージョンによって、使用できる関数が違ったりする。

Node 16は`es2021`をサポートしているようなので、`es2021`を設定することにした。

### `module`

どのモジュール形式を使うかを表す。`commonjs`、`umd`、`es6/es2015`などがある。このオプションだけで決まるわけではなく、`package.json`の"type"も影響しているようだった。

moduleをNode16にすると、typeがmoduleかどうかによって、ES Modulesで出力されるかどうかが決まった。

ここで、package.jsonのtypeフィールドについておさらいする。package.jsonのtypeに"module"を指定すると、その中のコードがES Modulesとして扱われる。

↑Node.jsで実行するときの話？

[Node.js における ES Modules を理解する - 30歳からのプログラミング](https://numb86-tech.hatenablog.com/entry/2020/08/07/091142)

とりあえずNode.jsで実行するときの話。"type": "module"を指定したり、拡張子をmjsに変更すると、ES Modulesとして実行される。

npmのpackage.jsonのドキュメントには、typeフィールドについての説明はなかった。https://docs.npmjs.com/cli/v9/configuring-npm/package-json

### `moduleResolution`

### CommonJSとES Modulesの相互運用について

Quramyさんの記事に分かりやすい表がある。ライブラリの使用側が Native ESMだった場合は、ライブラリがCJSでもESMでもimportでインポートできる。

使用側がCJSだった場合は、CJSはrequire、ESMはdynamic importでインポートする必要がある。

TypeScriptでmodule: "node16"を設定すると、package.jsonがtype: moduleの場合はimportをそのまま出力し、そうでない場合はimportをrequireに変換する。

ライブラリで、CommonJSとESMでエントリーポイントが分かれている場合があるのがなぜなのかが気になる。同じコードを使うのであれば、片定義は同じでいいのかなと思った。

ライブラリのビルド結果にCJSとESMの両方を含めるとどのようなメリットがあるのか。トランスパイラがCJS<->ESMの変換をするような気もするけれど、いったんその話は置いておく。

まず、使用側（TypeScriptでコードを書いているとする）がNative ESMだった場合は、CJSとESMの両方をインポートできるので問題ない。問題があるのは、使用側がCJSだった場合だ。TypeScriptはimportをdynamic importに書き換えないような気がするので、Native ESMのライブラリをCJSからstatic importしようとするとコンパイルエラーになるはずだ。つまりライブラリがESMのみを提供していると、使用側がCJSの場合にdynamic importする必要がある。CJSを提供すると、static importできる。

### 拡張子の省略について

### ライブラリを使用する時にはどうなるのか

package.jsonに`main`、`types`、`exports`などのフィールドがある。どのような場合にどのフィールドが使われるのかが知りたい。

次のファイルで試してみた。

```ts
import { CharStream } from "antlr4";

const chars = new CharStream("input");
console.log({ chars });
```
コンパイルエラーが出ているわけではないが、型定義を認識していない。VSCodeでホバーすると次が表示された。

```text
モジュール 'antlr4' の宣言ファイルが見つかりませんでした。'/Users/tekihei2317/ghq/github.com/tekihei2317/nodejs-import-playground/node_modules/antlr4/dist/antlr4.node.mjs' は暗黙的に 'any' 型になります。
  There are types at '/Users/tekihei2317/ghq/github.com/tekihei2317/nodejs-import-playground/node_modules/antlr4/src/antlr4/index.d.ts', but this result could not be resolved when respecting package.json "exports". The 'antlr4' library may need to update its package.json or typings.ts(7016)
```

"index.d.ts"はあるけれど、"exports"フィールドからは解決でき図、antlr4のライブラリ側に変更が必要かもしれないと書かれている。

"dist/antrl4.node.mjs"は、"main"と"exports"の両方で書かれている。ここでは"exports"側を参照している気がする。そこの"types"フィールドのファイル（"src/index.node.d.ts"）は、ライブラリ側に存在しなかった。つまり、ここを変更すれば解決すると思う。

```json
{
  "name": "antlr4",
  "version": "4.13.0",
  "type": "module",
  "browser": "dist/antlr4.web.js",
  "main": "dist/antlr4.node.mjs",
  "types": "src/antlr4/index.d.ts",
  "exports": {
    ".": {
      "node": {
        "types": "src/index.node.d.ts",
        "import": "./dist/antlr4.node.mjs",
        "require": "./dist/antlr4.node.cjs",
        "default": "./dist/antlr4.node.mjs"
      },
      "browser": {
        "types": "src/index.web.d.ts",
        "import": "./dist/antlr4.web.mjs",
        "require": "./dist/antlr4.web.cjs",
        "default": "./dist/antlr4.web.mjs"
      }
    }
  }
}
```

無理やり書き換えてみると、次のコンパイルエラーが発生した（一部省略）。

```tsc
index.ts:1:10 - error TS2305: Module '"antlr4"' has no exported member 'CharStream'.

1 import { CharStream } from "antlr4";
           ~~~~~~~~~~

node_modules/antlr4/src/antlr4/index.d.ts:1:15 - error TS2835: Relative import paths need explicit file extensions in EcmaScript imports when '--moduleResolution' is 'node16' or 'nodenext'. Did you mean './InputStream.js'?

1 export * from "./InputStream";
```

拡張子が省略できないと書かれている。上のメンバーが存在しないというのもこれが原因な気がする。あれ、最初はなんでコンパイルが成功したんだろう。

つまり、このモジュールはNative ESMで使うことを想定されていないということだろうか。Native ESMから使われる場合は、拡張子を省略せずに書く必要があるということか。つまり、ライブラリの拡張子が省略されている場合は、Native ESMはそもそも使えないということになる？

そこで、使用側をtype: "module"を消してCJSにしてみた。そうすると、importでエラーになる。CJSからESMを読み込むときは、Dynamic Importcを使う必要があるため。

Dynamic Importに書き換えてみる。上手くいかない。使用側がCJSである必要がありそうだ。そのため、tsconfigのmoduleをCommonJSに変更してみた。今度はちゃんと動いた。

```ts
import { CharStream } from "antlr4";

const chars = new CharStream("input");
console.log(chars);
```

### 問題の原因について

antlr4が、拡張子を省略しているのに"type": "module"でESMになっているのが問題？使用側がNative ESMだと、tscのコンパイルが通らない。

"module": "CommonJS"にするとちゃんと動くようになった。CommonJSからESMをインポートするときは、dynamic importが必要男じゃなかっただろうか（普通のimportがrequireに変換され、動作している）。つまり、antlr4はESMの設定になっているけれど実体はCommonJSになっているということ？

おそらく、antlr4のnpmのライブラリは、JavaScriptファイルに後から型定義を付け足した感じになっているのではないかなと思った。Webpackを使っているみたいだった。tscは入っているけれど、d.tsは手書きされた感じがある。

ESMとCommonJS、そしてtscによるコンパイル前と後の世界があり、複雑で頭がかなり混乱している。これに加えてトランスパイラ・モジュールバンドラなどもある。ややこしすぎる。

## メモ

[TypeScript 4.7 と Native Node.js ESM | by Yosuke Kurami | Medium](https://quramy.medium.com/typescript-4-7-%E3%81%A8-native-node-js-esm-189753a19ba8)
