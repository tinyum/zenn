---
title: "ORMのMapperはZodで攻略する"
emoji: "🧩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "drizzle", "zod", "ddd", "設計"]
published: true
---

## はじめに

最近の ORM 批判は「AI が SQL を書けるなら、マッピング層はいらない」です。オブジェクトとリレーショナルの変換に価値がない、という話。  
ORM と生 SQL の比較は、あえてしません。私は ORM を捨てる側ではありません。捨てずにドメインを守ろうとすると、Mapper が重い。

```typescript
function toUser(row: UserRow): User {
  return {
    id: UserId.of(row.id),
    email: Email.of(row.email),
    name: UserName.of(row.name),
    // フィールドが増えるたびにここも増える
  };
}
```

Entity の数だけこれを書き、フィールドを足すたびに直す。  
でも重いのは ORM のせいではなく、Mapper を手続きとして書いているからです。この記事では、Drizzle と Zod で Mapper を書かずに済ませる方法を紹介します。

## Value Object は schema を持っている

[前回の記事](https://zenn.dev/tinyum/articles/value-object-without-class)で、Value Object をこう書きました。

```typescript
// email.ts
import { z } from "zod";

const BLOCKED_DOMAINS = new Set(["tempmail.example", "trashmail.example"]);

export const schema = z
  .email()
  .refine((v) => !BLOCKED_DOMAINS.has(v.split("@")[1]), "Blocked domain")
  .brand<"Email">();

export type Email = z.infer<typeof schema>;

export function of(value: string): Email {
  return schema.parse(value);
}
```

`Email.of(row.email)` の中身は `schema.parse(row.email)` です。Mapper が各フィールドにやっている変換は、`z.object` に schema を並べれば Zod がまとめてやってくれます。

## Entity も schema で書く

```typescript
// user.ts
import { z } from "zod";
import * as UserId from "./user-id";
import * as Email from "./email";
import * as UserName from "./user-name";

export const schema = z.object({
  id: UserId.schema,
  email: Email.schema,
  name: UserName.schema,
});

export type User = z.infer<typeof schema>;

export function fromRow(row: z.input<typeof schema>): User {
  return schema.parse(row);
}
```

冒頭の `toUser` は消えて、`fromRow` の中身は `parse` だけになります。フィールドを足すときは schema に1行足せば、型も Mapper も同時に追従します。

引数を `z.input<typeof schema>` にしているのがポイントです。Drizzle の行をそのまま渡したとき、カラムが足りなければコンパイルエラーになります。

## Drizzle の行を流す

```typescript
// db/schema.ts
export const users = pgTable("users", {
  id: serial("id").primaryKey(),
  email: text("email").notNull(),
  name: text("name").notNull(),
  emailVerifiedAt: timestamp("email_verified_at"),
});
```

```typescript
// user.repository.ts
export async function findById(id: UserId): Promise<User | null> {
  const row = await db.query.users.findFirst({ where: { id } });
  return row ? User.fromRow(row) : null;
}
```

Repository はクエリを書いて `fromRow` に渡すだけ。snake_case との対応は Drizzle のテーブル定義が持っているので、Mapper で気にすることはありません。余分なカラムは `z.object` が無視します。

なお、テーブル定義から Zod schema を生成する [drizzle-zod](https://orm.drizzle.team/docs/zod) は使いません。あれは「テーブル → schema」で、ドメインが DB に依存する向きになります。やりたいのは逆で、ドメインが schema を持ち、DB の行はそれを満たす入力にすぎません。

## 値からは分からないことも schema に書く

前回の `VerifiedEmail` のように、複数カラムから1つの値が決まるケースは `transform` で書きます。

```typescript
// user.ts
export const schema = z
  .object({
    id: UserId.schema,
    email: Email.schema,
    name: UserName.schema,
    emailVerifiedAt: z.date().nullable(),
  })
  .transform(({ emailVerifiedAt, ...rest }) => ({
    ...rest,
    verifiedEmail: emailVerifiedAt ? Email.markVerified(rest.email) : null,
  }));
```

`z.input` が行の形、`z.infer` が Entity の形。1つの schema が両方の型を持つので、行の型と Entity の型を別々に定義して手で対応させる必要がありません。

## テーブル 1:1 でない Entity

User が複数の住所を持つなら、Entity の形にクエリを寄せて、その形の schema で受けます。

```typescript
// user.ts
export const schema = z.object({
  id: UserId.schema,
  email: Email.schema,
  name: UserName.schema,
  addresses: z.array(Address.schema),
});
```

```typescript
// user.repository.ts
const row = await db.query.users.findFirst({
  where: { id },
  with: { addresses: true },
});
return row ? User.fromRow(row) : null;
```

Drizzle の relational query はネストしたオブジェクトを返すので、`z.array(Address.schema)` でそのまま受かります。テーブルの形に引きずられるのはクエリ結果までで、Entity は引きずられません。

## DB の値を再検証するのは無駄では

私は必要だと思っています。DB の制約はドメインルールより弱いからです。`BLOCKED_DOMAINS` を DB は知りません。ルールを後から足したとき、古いデータが違反していると気づけるのは境界で `parse` しているからです。

実際、ルールを厳しくして過去データが読めなくなることはあります。ただそれは「バックフィルが必要なデータ」が境界で見つかったということです。旧データ用の schema を `z.union` で受けて `transform` で新しい形に寄せれば、バージョンごとの分岐も schema の中で完結します。手続きの Mapper に `if` を足すより見通しがいいです。

## まとめ

Mapper が重いのは、VO ごとの変換を Entity ごとに手で列挙しているからです。  
VO が schema を持っていれば Entity は schema の合成で書けて、Mapper は `parse` の1行になります。

## お知らせ

合同会社 tinyum では、バックエンドの負荷改善・開発生産性の向上を中心にお手伝いしています。

仕事のご相談・ご依頼は、プロフィールからお気軽にどうぞ。
