---
title: "CockroachDBでデータ型の変更(ALTER COLUMN TYPE)ができないときに読む記事"
emoji: "2️⃣"
type: "tech"
topics:
  - "sql"
  - "database"
  - "cockroachdb"
  - "newsql"
published: true
published_at: "2024-12-15 19:18"
publication_name: "comsize_press"
---

## ⚠WARNING⚠

:::message
この記事に記載されている内容は
2024/12/15 CockroachDB@v24.3.1 時点での情報です
関連IssueとCockroachDBの制約について記載されたドキュメントを
記事内に貼っているのでそちらも併せてご確認ください
:::

## TL;DR

- `ALTER COLUMN TYPE`はPreviewのためSESSION変数を書き換える必要がある
    - `SET enable_experimental_alter_column_type_general = true;`
- `ALTER COLUMN TYPE`は他の`ALTER TABLE`サブフォームと一緒に実行できない
    - e.g. `ALTER TABLE t ALTER COLUMN x TYPE STRING, ALTER COLUMN x SET NOT NULL;`
- `ALTER COLUMN TYPE`は**明示的な**トランザクション内で実行することもできない

## 経緯

`timestamp`型で定義したカラムを`timestampz`に変更するためのSQLを書いていました

```sql
ALTER TABLE foo
    ALTER COLUMN created_at SET DATA TYPE TIMESTAMP WITH TIME ZONE USING created_at AT TIME ZONE 'UTC',
    ALTER COLUMN updated_at SET DATA TYPE TIMESTAMP WITH TIME ZONE USING updated_at AT TIME ZONE 'UTC';
```

CockroachDB CloudのSQL Shellで試し打ちをしてみたところ
以下の怒られが発生してしまいました
![](https://storage.googleapis.com/zenn-user-upload/a15e3d0c611f-20241215.png)

```log
[XCEXF] ERROR: ALTER COLUMN TYPE from timestampt to timestamptz is only supported experimentally
See: https://go.crdb.dev/issue-v/49329/v24.2
--
you can enable alter column type general support by running `SET enable_experimental_alter_column_type_general = true`
```

https://github.com/cockroachdb/cockroach/issues/49329?version=v24.2

## 原因1: SESSION変数の設定

https://www.cockroachlabs.com/docs/v24.3/alter-table#alter-column-data-types

> Support for altering column data types is in preview, with certain limitations. To enable column type altering, set the enable_experimental_alter_column_type_general session variable to true.

データ型の変更はまだPreview機能でいくつかの制限付き
データ型を変更したい場合は`enable_experimental_alter_column_type_general`をtrueにしてね

とのこと

実際に設定されている値を見てみると無効になっていることがわかります
```sql
SHOW SESSION enable_experimental_alter_column_type_general;
```
| | enable_experimental_alter_column_type_general |
| --- | --- |
| 1 | off |

早速下記のクエリで機能を有効化していきます

```sql
SET enable_experimental_alter_column_type_general = true;
```

Serverless版を使っている人向けに書くと
SQL Shellで実行するとエラーになるので
`cockroach sql`やGUIクライアントなどで実行する必要があります

私はGoLandの[Database Tools and SQLプラグイン](https://www.jetbrains.com/help/go/relational-databases.html)を使いました
![SQL ShellでSETを実行してエラーが出ている](https://storage.googleapis.com/zenn-user-upload/853149b876f9-20241215.png)

## 原因2: もう一つ以上のサブフォームと組み合わせられない

先ほどSESSION変数を書き換えたので気を取り直して再実行していきます

```sql
ALTER TABLE foo
    ALTER COLUMN created_at SET DATA TYPE TIMESTAMP WITH TIME ZONE USING created_at AT TIME ZONE 'UTC',
    ALTER COLUMN updated_at SET DATA TYPE TIMESTAMP WITH TIME ZONE USING updated_at AT TIME ZONE 'UTC';
```

すると今度は別の怒られが
```log
[0A000] ERROR: unimplemented: ALTER COLUMN TYPE cannot be used in combination with other ALTER TABLE commands
You have attempted to use a feature that is not yet implemented.
See: https://go.crdb.dev/issue-v/49351/v24.2
```

https://go.crdb.dev/issue-v/49351/v24.2

> Similarly, ALTER COLUMN TYPE cannot be used in combination with other ALTER TABLE statements since this runs the statements inside a transaction.
>
> Example:
ALTER TABLE t ALTER COLUMN x TYPE STRING, ALTER COLUMN x SET NOT NULL;

データ型を変更する場合はそれ単独でALTER TABLEを実行する必要があるとのこと

`created_at`、`updated_at`のALTER COLUMN TYPEを個別で変更するように修正して実行
```sql
ALTER TABLE foo
    ALTER COLUMN created_at SET DATA TYPE TIMESTAMP WITH TIME ZONE USING created_at AT TIME ZONE 'UTC';

ALTER TABLE foo
    ALTER COLUMN updated_at SET DATA TYPE TIMESTAMP WITH TIME ZONE USING updated_at AT TIME ZONE 'UTC';
```

無事通りました
```log
ALTER COLUMN TYPE changes are finalized asynchronously; further schema changes on this table may be restricted until the job completes; some writes to the altered column may be rejected until the schema change is finalized
completed in 575 ms
```

私は[goose](https://github.com/pressly/goose)でマイグレーションをしているので
このSQLをgooseのマイグレーションファイルに落とし込んでいきます

## 原因3: トランザクション内で実行できない

`SET enable_experimental_alter_column_type_general = true;`を
ベタ書きしている是非はさておき

gooseのマイグレーションファイルにこれまでの歩みを反映させました

```sql
-- +goose Up
SET enable_experimental_alter_column_type_general = true;

ALTER TABLE foo
    ALTER COLUMN created_at SET DATA TYPE TIMESTAMP WITH TIME ZONE USING created_at AT TIME ZONE 'UTC';

ALTER TABLE foo
    ALTER COLUMN updated_at SET DATA TYPE TIMESTAMP WITH TIME ZONE USING updated_at AT TIME ZONE 'UTC';

-- +goose Down
SET enable_experimental_alter_column_type_general = true;

ALTER TABLE foo
    ALTER COLUMN created_at SET DATA TYPE TIMESTAMP WITHOUT TIME ZONE;

ALTER TABLE foo
    ALTER COLUMN updated_at SET DATA TYPE TIMESTAMP WITHOUT TIME ZONE;
```

`goose up`で実行してみるとまたしても別の怒られが発生
トランザクション内では実行できないそうです

```log
ERROR: unimplemented: ALTER COLUMN TYPE is not supported inside a transaction (SQLSTATE 0A000)
```

[原因2](#原因2%3A-もう一つ以上のサブフォームと組み合わせられない)に戻ってみると
Issueタイトルにも「sql: alter column type in transaction not supported」とあります

> ALTER COLUMN TYPE cannot be used in combination with other ALTER TABLE statements since this runs the statements inside a transaction.

原因2の根本的な原因は2つ以上のサブフォームを含む`ALTER TABLE`で内部的にトランザクションを使っていることに起因しているっぽいです

`-- +goose NO TRANSACTION`ディレクティブを追記して再実行
```sql
-- +goose NO TRANSACTION
-- +goose Up
SET enable_experimental_alter_column_type_general = true;

ALTER TABLE foo
    ALTER COLUMN created_at SET DATA TYPE TIMESTAMP WITH TIME ZONE USING created_at AT TIME ZONE 'UTC';

ALTER TABLE foo
    ALTER COLUMN updated_at SET DATA TYPE TIMESTAMP WITH TIME ZONE USING updated_at AT TIME ZONE 'UTC';

-- +goose Down
SET enable_experimental_alter_column_type_general = true;

ALTER TABLE foo
    ALTER COLUMN created_at SET DATA TYPE TIMESTAMP WITHOUT TIME ZONE;

ALTER TABLE foo
    ALTER COLUMN updated_at SET DATA TYPE TIMESTAMP WITHOUT TIME ZONE;
```

今度こそ完遂しました
```log
OK   19700101000001_alter_timestamp_with_tz.sql (2.61s)
goose: successfully migrated database to version: 19700101000001
```

## おわりに

実は`ALTER COLUMN TYPE`に対する制限は下記のページに一覧化されています
また、"該当カラムにCHECK制約が掛かっている場合" etc
私が今回遭遇したエラー以外にも制限もあるため詳しくはこちらをご参照ください
https://www.cockroachlabs.com/docs/stable/known-limitations#alter-column-limitations