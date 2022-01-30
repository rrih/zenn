---
title: "Heroku でデプロイした Node.js サーバーでデプロイ時 prisma の Error: P3009 によるサーバーダウンの解消"
emoji: "🐱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["heroku", "prisma"]
published: true # 公開設定
---

## 事象

Heroku で Node.js + prisma のアプリケーションをデプロイした時に Heroku サーバーがダウンしていた。ちなみに DB は Heroku の postgresql を使用している。
その時に`heroku logs --tail`コマンドで取得したログが以下の通り。

```bash
2022-01-27T10:12:15.811019+00:00 app[web.1]: Error: P3009
2022-01-27T10:12:15.811020+00:00 app[web.1]:
2022-01-27T10:12:15.811023+00:00 app[web.1]: migrate found failed migrations in the target database, new migrations will not be applied. Read more about how to resolve migration issues in a production database: https://pris.ly/d/migrate-resolve
2022-01-27T10:12:15.811024+00:00 app[web.1]: The `20220126142850_` migration started at 2022-01-27 08:13:18.846098 UTC failed with the following logs:
```

## 解決方法

これは prisma の migration に失敗したことが原因でデプロイに失敗している。よって migration に失敗している log を吐いている migration 用のレコードを削除して再度 migration することでデプロイに成功する(はず)

具体的には以下のような感じ

### Heroku の dataclip 
以下のクエリで _prisma_migrations から消すべきレコードを確認する。

#### _prisma_migrations テーブルについて

以下のようなテーブル構造になっている。

```bash
xxxxxxxxxxxxxx=> \d _prisma_migrations;
                        Table "public._prisma_migrations"
       Column        |           Type           | Collation | Nullable | Default
---------------------+--------------------------+-----------+----------+---------
 id                  | character varying(36)    |           | not null |
 checksum            | character varying(64)    |           | not null |
 finished_at         | timestamp with time zone |           |          |
 migration_name      | character varying(255)   |           | not null |
 logs                | text                     |           |          |
 rolled_back_at      | timestamp with time zone |           |          |
 started_at          | timestamp with time zone |           | not null | now()
 applied_steps_count | integer                  |           | not null | 0
Indexes:
    "_prisma_migrations_pkey" PRIMARY KEY, btree (id)
```

マイグレーションに失敗したレコードは nullable な `logs` カラムになんらかのエラーメッセージが存在するので `not null` で指定する。

### 該当レコードの削除

```bash
select id from  _prisma_migrations where logs is not null;
```

上記クエリで取得した id (uuid) で heroku posgre に cli で繋いで物理削除するクエリを実行。

``` 
heroku-app $ psql -h ec2-XX-XXX-XXX-XXX.compute-1.amazonaws.com -U xxxxxxxxxxxxxx -d xxxxxxxxxxxxxx
xxxxxxxxxxxxxx=> delete from _prisma_migrations where id='xxxxxxxx-123a-456b-7cd8-910e11fg1213'; -- 上記クエリで取得した id指定
```

その後再度 Heroku の dashboard から Manual deploy を実行すると、サーバーが正常に作動している。(はず)

## 参考文献

https://www.prisma.io/docs/guides/database/developing-with-prisma-migrate/customizing-migrations
https://stackoverflow.com/questions/15526776/its-possible-to-delete-a-row-in-heroku-postgresql
