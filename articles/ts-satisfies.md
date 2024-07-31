---
title: "satisfiesはどう活用するべきか"
emoji: "👮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typescript]
published: false
---

# はじめに

satisfiesについて、現場でstyleguideに記載があり、コーディングする中で使っています。
しかしながら、通常の型アノテーションとどう違うのか、どんなシーンで活用すべきなのかあまり把握していなかったので、
自己理解のためにも記事にしてみました。

# satifies について

まず、satisfiesとか何なのか、解説します。
satisfies は TypeScript 4.9 で追加された型アサーションの一種です。

**型アノテーション（型注釈）は型を明示的に指定して上書きするのに対し、satisfies は型付けに加えて、型アサーション（型推論）結果を保ちます。**

これはどういうことか、以下の例で示します。

```typescript
type RGB = [red: number, green: number, blue: number];

interface Color {
  red: RGB | string;
  green: RGB | string;
  blue: RGB | string;
}

// 型注釈
const customColor1: Color = {
  red: [255, 0, 0],
  green: "#00ff00",
  blue: "#0000ff",
};
// NG
const r1 = customColor1.green.toUpperCase();
// error: プロパティ 'toUpperCase' は型 'string | RGB' に存在しません。
// プロパティ 'toUpperCase' は型 'RGB' に存在しません

// satifies
const customColor2 = {
  red: [255, 0, 0],
  green: "#00ff00",
  blue: "#0000ff",
} satisfies Color;
// OK
const r2 = customColor2.green.toUpperCase();
```

通常の型注釈では green が`string | RGB`と上書きされて、型推論結果が失われます。
そのため、上記では string にのみ使える`toUpperCase`が使えません。

一方、satisfies では型推論が保たれます。
greenプロパティの実際の値はstring型なので、その型推論が残り、`toUpperCase`が使えます。

そして、satifsfies では型注釈と同様に、オブジェクトのキーの過不足がチェックされるため、型安全性は変わらず担保されます。

# 活用方法

## その 1: テストでの活用

テストで使うモックデータの実際の値は決まっています。
それにもかかわらず、コード上では実際に扱うデータを想定して型付けされているため、少し冗長なコードを書く必要があります。

例えば、以下の関数をテストしたいとします。

```typescript
const sampleFunc = (userName: string) => {
  return userName;
};
```

以下 User 型を用いたモックデータ`user1`を利用する場合、
sampleFunc は undefined を期待していないため、型アノテーションでモックデータを定義すると
関数の引数に 非 null アサーション（`!`）をつける必要があります。

```typescript
type User = {
  id: number;
  name?: string;
};

// モックデータ
const user1: User = {
  id: 1,
  name: "John",
};

const sampleFunc = (userName: string) => {
  return userName;
};

// NG
expect(sampleFunc(user1.name)).toStrictEqual(user1.name);
// error: 型 'string | undefined' の引数を型 'string' のパラメーターに割り当てることはできません。
// 型 'undefined' を型 'string' に割り当てることはできません

// そのため、非nullアサーションをつける必要がある
expect(sampleFunc(user1.name!)).toStrictEqual(user1.name);
```

そこで、satisfies を使うと、非 null アサーションをつける必要がなくなります。

```typescript
type User = {
  id: number;
  name?: string;
};

// モックデータ
const user2 = {
  id: 1,
  name: "John",
} satisfies User;

// 非nullアサーションを使う必要がない
expect(sampleFunc(user2.name)).toStrictEqual(user2.name);
```

なぜこれができるのかというと、satisfies では

- 型注釈と同様の型付けの効果により型安全性を担保する
- かつ、**実際に定義したオブジェクトの型推論結果を保つことができる**ことによって、柔軟性を損なわない

という性質があるからです。

## その 2 as const と組み合わせて定数を export する

例えば`constant.ts`みたいなファイルを作り、そこで定数を定義して export するケースがあると思います。
その際、定数定義に`as const`と`satisfies`を組み合わせると型安全性が向上します。

- `satisfies` が、型のキー過不足をチェックする
- `as const` が、値の widening を防止する

### 補足: as const について

`as const`は複数の作用を持ちますが、as const が付けられた式に登場するリテラルを「変更できないもの」として扱う機能 と理解すれば良いです。

また、widening とは、簡単に言うと型が大きくなる挙動のことです。これは型安全性の観点では、できるだけ抑制したい挙動です。
**そこで、`as const`がこの widening を防止できます。これにより変数が変更されないことが保証できます。**

```typescript
// 普通の変数定義
const names1 = ["makoto", "John", "Taro"];
// 推論結果: string[]型 → リテラル型ではない

// as constによる変数定義
const names2 = ["makoto", "John", "Taro"] as const;
// 推論結果: readonly ["makoto", "John", "Taro"]型 → wideningしないリテラル型になっている
```

### satisfies と as const の組み合わせ

配列の定数定義で組み合わせた例です。
colors から型を抽出した MyColor typeの推論結果は string[]ではなく、リテラル型になっています。
（**もし、satisfies ではなく型注釈を用いた場合は、当然 string[]になってしまいます**）

```typescript
export const colors = ["red", "blue", "green"] as const satisfies string[];

// import先
type MyColor = (typeof colors)[number];
// 推論結果: "red" | "blue" | "green"
```

詳しくは、こちらの記事がとてもわかりやすいのでご参考下さい。

https://zenn.dev/tonkotsuboy_com/articles/typescript-as-const-satisfies

# まとめ

Typescript では、変数が特定の値だけを持つことを保証するために、型注釈が利用されます。
一方それは、型推論がより具体的な型を導き出す可能性があるのにも関わらず、一般的な型で上書きしてしまうことになります。
型推論による柔軟性を保持しつつ、型安全性を担保したいシーンで、satisfies の利用を検討してみると良いと思いました。

# 参考

- https://devblogs.microsoft.com/typescript/announcing-typescript-4-9-beta/#the-satisfies-operator
- https://zenn.dev/tonkotsuboy_com/articles/typescript-as-const-satisfies
- https://zenn.dev/leaner_dev/articles/536b4c8d0a2bbd
