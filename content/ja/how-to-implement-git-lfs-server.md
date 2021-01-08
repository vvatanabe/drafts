# [詳解] Git LFSサーバーの作り方

Gitリポジトリでは管理しにくい大きなファイルを効率的に取り扱う仕組みの1つにGit LFSが上げられます。

本記事では、Git LFSの通信フローやAPI仕様等の仕組みを通してGit LFSサーバーの作り方を解説します。

**Outline**

```
cat ./content/ja/how-to-implement-git-lfs-server.md | grep "#"
```

## Git LFSのファイルの取り扱い

Git LFSはどのようにファイルを取り扱うのでしょうか。大きな特徴として以下の2つが上げられます。

### 大きなファイルを必要な分だけダウンロードする

Git LFSで管理しているファイルは、`git checkout`のタイミングで必要な分だけGit LFSサーバーからダウンロードされます。

### 大きなファイルの実態をリポジトリの外で管理する

リモートリポジトリにはGit LFSで管理しているファイルの実態はありません。代わりにファイルのメタ情報のみ書かれたポインタファイルを保持します。ポインタファイルはGit LFSサーバーに実体を問い合わせるための情報を格納した数百バイトのテキストファイルです。

ポインタファイルの例:

```
version https://git-lfs.github.com/spec/v1
oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
size 12345
```

## 用語の定義

本記事では以下の用語で使って解説します。

### LFSオブジェクト

Git LFSで管理するファイルの実態をLFSオブジェクトと呼びます。

### LFSメタデータ

Git LFSで管理するファイルのメタ情報をLFSメタデータと呼びます。

### ポインタファイル

リポジトリに保持されるLFSメタデータを格納するファイルをポインタファイルと呼びます。

### LFSクライアント

WIP

### LFSサーバー

WIP

## Git LFSの動作の仕組み

Git LFSはGitコマンドそのものに組み込まれているわけではありません。拡張モジュールとして別途インストールする必要があります。そのインストールしたGit LFSを使って、ローカルリポジトリが保存されているマシンから、リモートリポジトリを保持しているサーバーへGit LFSの規約に從っていくつかの通信を挟み、対象のファイルを読み書きします。

Git LFSコマンドは[github.com/git-lfs/git-lfs](https://github.com/git-lfs/git-lfs)で開発されています。Goで実装されており、コマンド自体はOS毎にシングルバイナリで提供されています。

Git LFSコマンドをインストールしたら、`git lfs`のような形式で Git のサブコマンドとして実行できるようになります。

次に、Git LFSコマンドが、どのようにGitのワークフローに処理を介入させているのか見ていきます。

### clean、smudge フィルターによる処理の介入

Git LFSは以下2つのGitのフィルター機能を使うことで、Gitのワークフローに適切な処理を挟みます。

- clean
- smudge

Git LFSのインストール後に、以下のコマンドでgitconfigの内容を見るとフィルターが設定されていることが確認できます。

```
$ git config --list --show-origin
file:/etc/gitconfig     filter.lfs.clean=git-lfs clean -- %f
file:/etc/gitconfig     filter.lfs.smudge=git-lfs smudge -- %f
file:/etc/gitconfig     filter.lfs.process=git-lfs filter-process
file:/etc/gitconfig     filter.lfs.required=true
```

#### cleanフィルター

cleanフィルターはステージングの直前に挟む処理を設定できます。

以下は、git-scm.comで使われているcleanフィルターの解説画像です。

![clean](../../static/images/clean.png)

cleanフィルターによりステージングエリアに持っていくファイルに対して任意の変換処理を実施しています。

Git LFSの場合、ここで`git-lfs clean`コマンドが実行され、対象のファイルのLFSメタデータを作成して、実態のLFSオブジェクトをローカルリポジトリの`.git/lfs`以下に保持しています。

#### smudgeフィルター

smudgeフィルターはチェックアウトの直前に挟む処理を設定できます。

以下は、git-scm.comで使われているsmudgeフィルターの解説画像です。

![smudge](../../static/images/smudge.png)

smudgeフィルターによりステージングエリアからワーキングディレクトリに展開するファイルに対して任意の変換処理を実施しています。

Git LFSの場合、ここで`git-lfs smudge`コマンドが実行され、対象のファイルのLFSメタデータを元に実態のLFSオブジェクトをワーキングディレクトリへ展開します。`.git/lfs`以下に対象のLFSオブジェクトが存在しなければ、Git LFSサーバーからプロトコルに従っていくつか通信を挟みダウンロードされます。

### Gitフックによる処理の介入

Git LFSは以下4つのGitフックを使うことで、Gitのワークフローに適切な処理を挟みます。

- post-checkout
- post-commit
- post-merge
- pre-push

上から順に処理内容を解説します。

#### post-checkout、post-commit、post-merge

post-checkoutフックは`git checkout`の後に実行されるGitフックです。スクリプト内では`git lfs post-checkout`が実行されています。

post-commitフックは`git commit`の後に実行されるGitフックです。スクリプト内では`git lfs post-commit`が実行されています。

post-mergeフックは`git merge`の後に実行されるGitフックです。スクリプト内では`git lfs post-merge`が実行されています。

これらは、Git LFSが提供するファイルのロック機能に関わっています。具体的には、`git lfs track`でロック可能なものとしてマークされているファイルが、ローカルユーザーによってロックされていない場合、ワーキングディレクトリのファイルのアクセス権を読み込み専用に変更します。

#### pre-pushフック

pre-pushフックは`git push`の直前に実行されるGitフックです。スクリプト内では`git lfs pre-push`が実行されています。

プッシュされるコミットにGit LFSで管理されているファイルがあれば、それらを後述のGit LFS APIを使ってプッシュします。

### Git LFSサーバーとの通信フロー

Git LFS APIと呼ばれる、Git LFSで管理しているファイルをリポジトリの外で効率的に管理するためのAPIを提供します。
規約の詳細は後述しますが、大きく分けて以下の3つのAPIをHTTPで提供します。

- Batch API
- Basic Transfer API
- File Locking API

※ SSHプロトコルでcloneやpushした場合でもGit LFSの通信はHTTPです。

次に、Git LFSのクライアントとサーバーはどのように通信して大きなファイルをやりとりするのか、大枠をシーケンス図をもとに解説します。

#### アップロードのフロー

以下の図は、 Git LFS で管理しているファイルをアップロードする通信のシーケンスです。

![ファイルのアップロード](https://cacoo.com/diagrams/dRR7txSDzNoo2TR5-F4177.png)

ローカルマシンで`git push`を実行すると直前のGitフックとしてpre-pushフックが実行されます。pre-pushフックで実行される処理は以下のとおりです。

1. Git LFSサーバーのBatch APIを呼び出し、LFSメタデータを保存します。レスポンスにはLFSオブジェクトの保存先となるBasic Transfer APIのURL等の情報が含まれます。
2. 1.のレスポンスを元に、Basic Transfer APIを呼び出しLFSオブジェクトをアップロードします。
3. もし、1.のレスポンスにVerifyオブジェクトと呼ばれる検証用APIのURL等の情報が含まれていれば、そのURLへPOSTリクエストを送信します。これはオプションの機能なので、LFSサーバーはVerifyオブジェクトを返さなくても動作に影響はありません。

成功すればローカルリポジトリ内のGitオブジェクトがリモートリポジトリにプッシュされます。

#### ダウンロードのフロー

以下の図は、 Git LFS で管理しているファイルをダウンロードする通信のシーケンスです。

![ファイルのダウンロード](https://cacoo.com/diagrams/dRR7txSDzNoo2TR5-AB13D.png)

ローカルマシンで`git checkout`を実行すると直前のフィルターとしてsmudgeフィルターが実行されます。smudgeフィルターで実行される処理は以下のとおりです。

1. Git LFSサーバーのBatch APIを呼び出し、LFSメタデータを読み取ります。レスポンスにはLFSオブジェクトの取得先となるBasic Transfer APIのURL等の情報が含まれます。
2. 1.のレスポンスを元に、Basic Transfer APIを呼び出しLFSオブジェクトをダウンロードします。

#### ファイルロックの通信フロー

WIP

以下の図は、 Git LFSで管理しているファイルをロックする通信のシーケンスです。

![ファイルのロック]()

## Git LFS API の仕様

Git LFS を実現する上で、サーバーで提供する必要があるAPI（以下、Git LFS API）について、公式のドキュメントを基に解説します。

### Batch API ― LFSメタデータを転送する

Batch APIは、LFSオブジェクトと呼ばれるGit LFSで管理しているファイルの実態をLFSサーバーへ転送するためのエンドポイントを取得するために使用されます。

Batch APIのURLは、以下の例のようにリモートリポジトリのベースのURLに、`/info/lfs/objects/batch`を接尾辞として追加することによって構築されます。

```
Git remote: https://example.com/foo/bar.git
Batch API:  https://example.com/foo/bar.git/info/lfs/objects/batch
```

Git LFSコマンドがGit LFS APIのURLを構築する仕組みの詳細は、[Server Discovery](https://github.com/git-lfs/git-lfs/blob/master/docs/api/server-discovery.md)に記載されています。

#### パス, メソッド, ヘッダーのルール

以下は、Batch APIのURLを形成する、パス, メソッド, ヘッダーのルールです。

Path:

`*/lfs/info/objects/batch`

Method:

`POST`

Header:

```
Accept: application/vnd.git-lfs+json
Content-Type: application/vnd.git-lfs+json
```

#### リクエスト

以下は、Batch APIのリクエストパラメータの例です。各プロパティの詳細を解説します。

```
// POST https://example.com/foo/bar.git/info/lfs/objects/batch
// Accept: application/vnd.git-lfs+json
// Content-Type: application/vnd.git-lfs+json
// Authorization: Basic ...

{
  "operation": "download",
  "transfers": [ "basic" ],
  "ref": { "name": "refs/heads/master" },
  "objects": [
    {
      "oid": "12345678",
      "size": 123,
    }
  ]
}
```

`operation`

Git LFSで管理しているファイルの転送種別を示します。値は文字列の`download`か`upload`である必要があります。

`transfers`

Git LFSクライアントが設定した転送方式を示します。現在（2020/12）のGit LFSの仕様ではBasic Transfer方式のみをサポートしています。

値は文字列の配列でBasic Transfer方式を示す`basic`のみ受け付けます。省略された場合、Git LFSサーバはデフォルトの挙動としてBasic Transfer方式を想定する必要があります。

`operation`が`upload`の時は[tusプロトコル](https://tus.io/protocols/resumable-upload.html)を使用できます。tusプロトコルはあくまで任意なのでサーバー側で実装しなくてもGit LFSは問題なく動作します。

`ref`

転送するLFSオブジェクトが属しているシンボリック参照を示すオブジェクトです。Git LFS v2.4から追加されました。

`ref.name`

シンボリック参照と呼ばれる`refs/heads/master`のような形式のブランチのパスの文字列が格納されます。これにより、Git LFSサーバーは転送するLFSオブジェクトが属するブランチ単位で認証できるようになります。サーバーはこの値を無視しても問題なく動作します。

`objects`

転送するオブジェクトの配列です。

`object.oid`

LFSオブジェクトの識別子となる、LFSオブジェクトの中身から算出されたSHA256の文字列です。

`object.size`

LFSオブジェクトのバイトサイズを示す整数です。

#### 成功時のレスポンス

以下は、Batch APIの成功時のレスポンスの例です。リクエストに問題がある場合（不正な承認、不正なjsonなど）がない限り、Batch APIは常に200のステータスを返す必要があります。

```
// HTTP/1.1 200 Ok
// Content-Type: application/vnd.git-lfs+json
{
  "transfer": "basic",
  "objects": [
    {
      "oid": "1111111",
      "size": 123,
      "authenticated": true,
      "actions": {
        "upload": {
          "href": "https://example.com/foo/bar.git/lfs/objects/111111",
          "header": {
            "Key": "value"
          },
          "expires_at": "2016-11-10T15:29:07Z",
        }
        "verify": {
          "href": "https://example.com/foo/bar.git/lfs/verify/111111",
          "header": {
            "Key": "value"
          },
          "expires_at": "2016-11-10T15:29:07Z",
        }
      }
    }
  ]
}
```

`transfer`

Git LFS サーバーが優先する転送方式を示す文字列です。基本的にはBasic Transfer方式となります。

リクエストから与えられた転送方式である`transfers`の1つでなければなりません。

このプロパティを省略すると、LFSクライアントはBasic Transfer方式を使用します。

`objects`

転送するLFSオブジェクトの配列です。

`object.oid`

LFSオブジェクトの識別子となるLFSオブジェクトの中身から算出されたSHA256の文字列です。

`object.size`

LFSオブジェクトのバイトサイズを示す整数です。

`object.authenticated`

対象のLFSオブジェクトに対するリクエストが認証されたかどうかを示す真偽値です。省略またはfalseの場合、Git LFSコマンドはリポジトリの認証情報を探して、LFSオブジェクトの転送時のBasic認証で使用します。

`object.actions`

Git LFSクライアントが次に行うアクションを示すオブジェクトです。

返却されるアクションはリクエストで指定された`operation`によって異なり、`download`、`upload`、`verify`のいずれかのキーを持ちます。

また、なんらかの理由でLFSオブジェクトをダウンロードできない場合は、`error`をキーにしてLFSオブジェクト単位のエラー情報を含める必要があります。

リクエストの`operation`が`download`の場合、レスポンスの`actions`に`download`を含める必要があります。

リクエストの`operation`が`upload`の場合、レスポンスの`actions`に`uplaod`と`verify`を含めることができます。

レスポンスのオブジェクトに`verify`のアクションがある場合、LFSクライアントはアップロードが成功した後にこのURLへリクエストします。リクエスト先のサーバーは必要に応じて対象のLFSオブジェクトの検証処理用のエンドポイントとして使用できます。

LFSクライアントがLFSサーバーが既に持っているオブジェクトをアップロードするようにリクエストした場合、LFSサーバーは対象のLFSオブジェクトの`actions`プロパティを完全に省略しれレスポンスを返却する必要があります。LFSクライアントは、LFSサーバーが既に持っていると仮定して動作します。

`action.href`

LFSオブジェクトをダウンロードやアップロードするためのURLの文字列です。

`action.header`

LFSオブジェクトをダウンロードやアップロードするためのリクエストに適用するHTTPヘッダです。

`action.expires_in`

LFSオブジェクトをダウンロードやアップロードする有効期限を秒数で表します。`expires_in`と`expires_at`の両方が指定されている場合は`expires_in`が優先されます。最大値は2147483647、最小値は-2147483647です。

`action.expires_at`

LFSオブジェクトをダウンロードやアップロードする有効期限をRFC3339形式（大文字のタイムスタンプ）で表します。

#### エラーレスポンス

それぞれのLFSオブジェクトへのアクセスに問題がある場合、LFSサーバーは200のステータスコードを返し、LFSオブジェクトごとのエラー内容を提供する必要があります。
`error.code`には可能な限り40x等のHTTPのステータスコードを使用すべきです。例えば、対象のLFSオブジェクトはサーバー上で見つからない場合は、`error.code`は404を指定してください。

以下はその例です。

```
// HTTP/1.1 200 Ok
// Content-Type: application/vnd.git-lfs+json
{
  "transfer": "basic",
  "objects": [
    {
      "oid": "1111111",
      "size": 123,
      "error": {
        "code": 404,
        "message": "Object does not exist"
      }
    }
  ]
}
```

```
// HTTP/1.1 404 Not Found
// Content-Type: application/vnd.git-lfs+json

{
  "message": "Not found",
  "documentation_url": "https://example.com/docs/errors",
  "request_id": "123"
}
```