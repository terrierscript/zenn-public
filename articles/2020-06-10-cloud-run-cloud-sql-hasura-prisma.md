---
title: CloudRun + CloudSQLでHasuraとかPrismaとか使うときの設定のメモ
emoji: 🎾
type: idea
topics:
  - cloudrun
  - cloudsql
  - hasura
  - prisma
  - graphql
published: true
terrier_export_from: https://scrapbox.io/terrierscript/CloudRun_+_CloudSQL%E3%81%A7Hasura%E3%81%A8%E3%81%8BPrisma%E3%81%A8%E3%81%8B%E4%BD%BF%E3%81%86%E3%81%A8%E3%81%8D%E3%81%AE%E8%A8%AD%E5%AE%9A%E3%81%AE%E3%82%A2%E3%83%AC
---

> 記事作成日: 202/06/10

# 前提

まずCloudRunとCloudSQLの接続はunix_socketで行われる。
https://cloud.google.com/sql/docs/mysql/connect-run

prisma / hasuraはそれを指定すれば良いだけなのだが、それぞれパースがされないことなどがあり、困るケースがある。

また、今回はPostgreSQLの場合の例になる

# prisma

[Prisma](https://www.prisma.io/)の場合はちょっと注意が必要

```
postgres://user:password@localhost:5432/dbname?host=/cloudsql/PROJECT_ID:REGION:INSTANCE
```


ポイントは`localhost`の部分を省略せず記述すること。省略するとパースが壊れてうまく読み込まれない。

see
* https://github.com/prisma/prisma-client-js/issues/437
* https://www.prisma.io/docs/reference/database-connectors/postgresql

# hasura

[Hasura](https://hasura.io/)の場合はPrismaほどハマらない。
HASURA_GRAPHQL_DATABASE_URLの変数に下記のように指定すれば良い

```
postgres://user:password@/dbname?host=/cloudsql/PROJECT_ID:REGION:INSTANCE 
```

see:
* https://stackoverflow.com/questions/58361874/use-hasura-with-google-cloud-run-and-google-cloud-sql
* https://medium.com/tactable-blog/building-a-serverless-graphql-app-with-next-js-hasura-and-cloudrun-fb8ca7c5e757

