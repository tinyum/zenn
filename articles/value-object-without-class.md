---
title: "Value ObjectをClassで書くのはもうやめませんか"
emoji: "🪦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "ddd", "設計", "zod"]
published: true
---

## はじめに

TypeScript で Value Object を作るとき、私は昔こう書いていました。

```typescript
class Email {
  private readonly value: string;

  constructor(value: string) {
    if (!/^[^@]+@[^@]+$/.test(value)) {
      throw new Error("Invalid email");
    }
    this.value = value;
  }

  getValue(): string {
    return this.value;
  }

  equals(other: Email): boolean {
    return this.value === other.value;
  }
}
```

Java の設計本で覚えた作法をそのまま TypeScript に持ち込んでいたわけですが、今はもう書いていません。TypeScript にはもっと軽い表現があると気づいたからです。  
この記事では、私が今どう書いているか、モジュールと Branded Type で Value Object を書く方法を紹介します。

## Class 方式の何が問題か

まず、Class のインスタンスは `===` が参照比較です。Value Object の本質は「同じ値なら同じもの」なのに、素直に比較すると壊れる。だから `equals` を実装して、使う側もそれを呼ぶことを覚えておく必要があります。

次に、JSON との往復。Class インスタンスは `JSON.parse` では復元できないので、API や DB の値をドメインに持ち込むたびに復元コードを書くことになります。ネストしていれば再帰的に組み立て直しです。

そしてボイラープレート。冒頭のコードでドメインの知識は正規表現の1行だけです。残りの `private readonly`、`constructor`、`getValue`、`equals` は毎回同じ形で書かされる定型コードで、Value Object の数だけ増えます。

## モジュールで書く

同じものをモジュールで書くとこうなります。検証は自前で書かず、最初から Zod に任せます。

```typescript
// email.ts
import { z } from "zod";

// 使い捨てメールなどのブラックリスト
const BLOCKED_DOMAINS = new Set(["tempmail.example", "trashmail.example"]);

const schema = z
  .email()
  .refine((v) => !BLOCKED_DOMAINS.has(v.split("@")[1]), "Blocked domain")
  .brand<"Email">();

export type Email = z.infer<typeof schema>;

export function of(value: string): Email {
  return schema.parse(value);
}

export function domain(email: Email): string {
  return email.split("@")[1];
}
```

使う側は名前空間ごとインポートします。

```typescript
import * as Email from "./email";

const email = Email.of("user@example.com");
Email.domain(email); // "example.com"
email === Email.of("user@example.com"); // true（ただのstringなので値比較）
```

`Email.of` や `Email.domain` という呼び出しは、Class のメソッドと変わらない凝集です。「関数がバラバラになる」という心配は名前空間インポートで解決します。

等価性は `===` がそのまま値比較。`equals` の実装も呼び忘れもありません。JSON との往復も、実体がただの string なので変換コードが不要です。

関数を `map` にそのまま渡せるのも地味に効きます。

```typescript
// モジュール方式: 関数をそのまま渡せる
const domains = rawInputs.map(Email.of).map(Email.domain);

// Class方式: メソッドは渡せないのでアロー関数で包むしかない
const domains2 = rawInputs.map((v) => new Email(v)).map((e) => e.getDomain());
```

メソッドは `this` に縛られているので、`map(e.getDomain)` と直接渡すと壊れます。モジュールの関数にはこの罠がありません。

## Branded Type が Class の検証を引き継ぐ

「`type Email = string` のエイリアスでよくない？」と思うかもしれません。ダメです。  
TypeScript は構造的型付けなので、エイリアスだと検証を通っていない生の string がどこでも `Email` として通ります。Class の constructor が持っていた「検証済みの値しか存在しない」という保証が消えるのです。

さっきのコードに `.brand<"Email">()` を付けたのはこのためです。`Email` は実行時にはただの string ですが、型の上では `parse` を通さない限り作れません。

```typescript
function send(to: Email) { /* ... */ }

send("raw string");                // コンパイルエラー
send(Email.of("user@example.com")); // OK
```

Class の一番の存在意義だった「検証の入口を1つに絞る」を、型だけで再現できます。境界で `parse` した瞬間からドメインの型として扱えるので、変換レイヤーもほぼ消えます。

brand は重ねられます。「形式が正しい」の上に「社内ドメインである」という業務ルールを、別の型として積めます。

```typescript
// email.ts に追記
const internalSchema = schema
  .refine((v) => v.endsWith("@example.co.jp"), "Not an internal email")
  .brand<"InternalEmail">();

export type InternalEmail = z.infer<typeof internalSchema>;

export function asInternal(email: Email): InternalEmail {
  return internalSchema.parse(email);
}
```

```typescript
function grantAdminRole(to: InternalEmail) { /* ... */ }

const email = Email.of("user@example.co.jp");
grantAdminRole(email);                   // コンパイルエラー（社内ドメイン検証がまだ）
grantAdminRole(Email.asInternal(email)); // OK
```

`InternalEmail` は `Email` の一種なので、`Email.domain` などはそのまま使えます。  
「管理者権限は社内メールにしか付与できない」という業務ルールがコンパイラで守られる。Class でやるとサブクラスや別ラッパーが要るところが、brand の重ね掛けなら数行です。

## 値からは分からないことも型にできる

認証システムで必ず出てくる「本人確認済みかどうか」は、文字列を見ても分かりません。値の外にある事実です。  
これも brand で表現できます。ただし `refine` ではなく、事実が確定する場所を関数に絞ります。

```typescript
// email.ts に追記
export type VerifiedEmail = Email & z.$brand<"VerifiedEmail">;

// トークン照合の成功時と、DB からの復元時だけ呼ぶ
export function markVerified(email: Email): VerifiedEmail {
  return email as VerifiedEmail;
}

// DB の行から復元。repository のマッパーから呼ぶ
// email_verified_at が入っていれば確認済みとして返す
export function verifiedFromRow(row: { email: string; email_verified_at: Date | null }): VerifiedEmail | null {
  return row.email_verified_at ? markVerified(of(row.email)) : null;
}
```

```typescript
function sendPasswordReset(to: VerifiedEmail) { /* 本人確認済みにしか送らない */ }

sendPasswordReset(email);                      // コンパイルエラー（本人確認前）
sendPasswordReset(Email.markVerified(email));  // OK（トークン照合後）
```

`markVerified` の中身はただのキャストですが、呼べる場所を2つに閉じることで「`VerifiedEmail` が存在する ＝ 本人確認が済んでいる」が成立します。  
未確認のメールにパスワードリセットを送るバグが、レビューではなくコンパイルで落ちます。

## まとめ

Value Object は不変でライフサイクルもない、ただの値です。そこに Class の道具は要りません。  
Java の作法ではなく、TypeScript の型システムに乗った Value Object を書きましょう。

## お知らせ

合同会社 tinyum では、バックエンドの負荷改善・開発生産性の向上を中心にお手伝いしています。

仕事のご相談・ご依頼は、プロフィールからお気軽にどうぞ。
