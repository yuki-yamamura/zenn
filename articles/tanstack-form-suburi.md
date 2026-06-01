---
title: "TanStack Form 素振り"
emoji: "🏸"
type: "tech"
topics:
  - "tanstackform"
published: true
publication_name: "frontendflat"
---

この記事では TanStack Form の特徴に触れつつ、基本的な使い方からフォーム実装の頻出パターンまで順を追って説明します。React / Next.js 環境で使用する場合の全体像をざっくりと把握したい方向けの記事となります。

## 環境とゴール

Next.js アプリケーションで [Server Actions](https://nextjs.org/docs/13/app/building-your-application/data-fetching/server-actions-and-mutations) を用いたマルチステップフォームの実装を目標にします。最も素朴な使い方からはじめて、徐々に実用に寄せた条件を追加していきます。記事の構成上、後のセクションはそれ以前の内容を前提としているためご了承ください。

```
# 各ライブラリのバージョン
"next": "16.2.4",
"react": "19.2.5",
"@tanstack/react-form": "^1.29.1",
"@tanstack/react-form-nextjs": "^1.29.1"
```

![](https://static.zenn.studio/user-upload/9bdaef038b91-20260602.png)
_作成するマルチステップフォーム_

## TanStack Form の哲学

公式ドキュメントで紹介されている哲学のうち、以下の 2 つは使っている中でも度々実感しました。事前にライブラリの特徴として抑えておくと良さそうです。

- Generics are grim: 徹底して値から型を推論させる API となっている
- Upgrading unified APIs: 頭の中に API の地図ができてくると使いやすく感じる。それまでは学習コストが高く感じた

https://tanstack.com/form/latest/docs/philosophy

## 基本的な使い方

はじめにフォームを構築する上で必須の API から見ていきます。

- フォームの初期化
- フィールドの状態にアクセス
- リアクティビティ

### フォームの初期化

フォームの宣言には [useForm](https://tanstack.com/form/latest/docs/framework/react/reference/functions/useForm) フックを使います。フォームの型は初期値である `defaultValues` から推論されるため、[型パラメータ](https://www.typescriptlang.org/docs/handbook/2/generics.html#generic-parameter-defaults)で改めて指定する必要はありません。この辺の使い勝手は他のフォームライブラリ ([React Hook Form](https://react-hook-form.com/api) など) と似通った馴染みのものだと思います。

```tsx
import { useForm } from "@tanstack/react-form";

function SpotForm() {
  // useForm フックでフォームを宣言
  const form = useForm({
    defaultValues: {
      description: "",
      name: "",
    },
    // 👆 defaultValues からフォームの型が推論される
    onSubmit: ({ value }) => {
      console.log({ value });
    },
  });

  // ...
}
```

TanStack Form 固有の部分として、`useForm` フックに渡すパラメータを共通化するための API である [formOptions](https://tanstack.com/form/latest/docs/reference/functions/formOptions) 関数が用意されています。`formOptions` 関数を使うことで、同じ構造を持つ 2 つ以上のフォーム (例えば、登録・更新など) や、クライアント / サーバー間のバリデーションといったケースでコードの重複を排除できます。

```tsx
import { formOptions } from "@tanstack/react-form";

// コンポーネントの外側でフォームのパラメータを管理できる
const spotFormOpts = formOptions({
  defaultValues: {
    description: "",
    name: "",
  },
  onSubmit: ({ value }) => {
    console.log({ value });
  },
});

// 登録用フォーム
function CreateSpotForm() {
  const form = useForm(spotFormOpts);

  // ...
}

// 更新用フォーム
function UpdateSpotForm() {
  const form = useForm(spotFormOpts);

  // ...
}
```

:::details formOptions 関数の実用的な使い方

実際に `formOptions` 関数を使ってみると、登録・更新用のフォームで初期値が違ったり、型推論のために初期値だけ必要といったケースが出てきます。
そのため、関数でラップして動的なパラメータを注入できるようにしたり、デフォルトパラメータによる初期値の隠蔽を行えるようにしておくと便利です。

```ts
export function createSpotFormOptions({
  defaultValues = {
    description: "",
    name: "",
  },
  onSubmit,
}: {
  defaultValues?: { description: string; name: string };
  onSubmit?: () => void;
} = {}) {
  return formOptions({
    defaultValues,
    onSubmit,
  });
}
```

:::

### フィールドの状態にアクセス

続いて `SpotForm` コンポーネントの JSX を見ていきます。抑えておきたい特徴は 3 つです。

1. Render Props による型安全
2. 制御コンポーネント
3. メタ情報へのアクセスとダーティチェック

```tsx
// SpotForm の JSX
<form
  onSubmit={(e) => {
    e.preventDefault();
    void form.handleSubmit(e);
  }}
  className={styles.base}
>
  {/* form.Field コンポーネントでフィールドをつくる */}
  <form.Field name="name">
    {(field) => (
      {/* 👆 ①  Render Props なので型推論が効く*/}
      <>
        <label className={styles.label}>
          <span>名前</span>
          <input
            name={field.name}
            // 👆 ②  制御コンポーネントを使用する (React が状態を管理)
            value={field.state.value}
            placeholder="東京港野鳥公園"
            onChange={(e) => {
              field.handleChange(e.target.value);
            }}
            className={styles.input}
          />
        </label>
        {field.state.meta.isDirty && field.state.meta.errors.length > 0 && (
          // 👆 ③ フィールドのメタ情報を field.state.meta プロパティで参照する
          <em role="alert">{field.state.meta.errors.at(0)?.message}</em>
        )}
      </>
    )}
  </form.Field>
  <form.Field name="description">
    {(field) => (
      <label className={styles.label}>
        <span>説明</span>
        <textarea
          value={field.state.value}
          placeholder="東京都大田区の野鳥を観察できる公園"
          onChange={(e) => {
            field.handleChange(e.target.value);
          }}
          className={styles.input}
        />
      </label>
    )}
  </form.Field>
  <button type="submit" className={styles.submitButton}>
    送信
  </button>
</form>
```

#### ① Render Props による型安全

`form.Field` コンポーネントはフォームの初期値から型推論され、フィールド API の型が注入された `children` を受け取ります。そのため、Render Props で `children` として渡す子コンポーネント内では型補完がバリバリ効いて嬉しいです。

#### ② 制御コンポーネント

TanStack Form では非制御コンポーネント向けの API は提供されておらず、一貫して制御コンポーネントを使用します。API の学習コストはありますが、パフォーマンスと DX (強力な型推論) を両立するための選択だと解釈しています。

https://legacy.reactjs.org/docs/forms.html#controlled-components

#### ③ メタ情報へのアクセスとダーティチェック

`field.state` オブジェクトにフィールドの状態に関するプロパティが生えており、その中の 1 つである `meta` を経由してフィールドのメタデータにアクセスできます。
TanStack Form では Persistent dirty state (一度フィールドの値を書き換えたら、初期に戻してもダーティと判定する方式) を採用しており、React Hook Form の Non-persistent dirty state と異なる挙動をするので注意が必要です。また、React Hook Form の `isDirty` と同じ挙動をする `isDefaultValue` も存在するため、利用者のニーズに応じて使い分けられる柔軟性も備えています。

![](https://static.zenn.studio/user-upload/aeccc53c2946-20260531.png)
_[公式ドキュメント](https://tanstack.com/form/latest/docs/framework/react/guides/basic-concepts#field-state)より「フィールドの状態」_

### リアクティビティ

フォームの状態変化に反応して UI を変化させるには、リアクティビティに関する API が必要です。ここでは TanStack Form で提供される 2 通りの方法を抑えておきましょう。

1. [useStore](https://tanstack.com/form/latest/docs/framework/react/guides/reactivity#usestore) フック
2. [form.Subscription](https://tanstack.com/form/latest/docs/framework/react/guides/reactivity#formsubscribe) コンポーネント

```tsx
// useStore フックの使い方
// 🙆‍♂️ 不要な再レンダリングを抑制するため、セレクタで使用する値だけ返す
const isSubmitting = useStore(form.store, (state) => state.isSubmitting);

// 🙅‍♂️ フォームの状態をそのまま返すと、関係のない状態変化でコンポーネント全体が再レンダリングされてしまう
const formState = useStore(form.store, (state) => state);
const { isSubmitting } = formState;
```

```tsx
// form.Subscribe コンポーネントの使い方
// セレクタで返す値が変わったら再レンダリングされる (ラップしたコンポーネント以外を巻き込まない)
<form.Subscribe selector={(state) => state.isSubmitting}>
  {(isSubmitting) => (
    <button
      type="submit"
      disabled={isSubmitting}
      className={styles.submitButton}
    >
      送信
    </button>
  )}
</form.Subscribe>
```

基本的には値の取得だけが目的であれば、`form.Subscribe` コンポーネントを使った方がシンプルだと思います。一方で JSX 内に値の計算といった処理を書きたくない、複数フィールドやフォーム全体で使用するといったケースでは `useStore` フックを使うと良いでしょう。

## バリデーション

このセクションでは、高いカスタマイズ性を持つバリデーションに関する API を紹介します。
バリデーションの単位とタイミングはユーザーが完全に制御できるようになっており、具体的には以下のようなコードで実装できます。

```tsx
// フィールド単位のバリデーション
<form.Field
  name="name"
  validators={{
    onChange: ({ value }) => {
      // 👆 バリデーションの実行タイミング
      if (value.length === 0) {
        return "必ず入力してください";
      }
    },
  }}
>
  {(field) => {
    /* ... */
  }}
</form.Field>
```

```diff ts
// フォーム全体のバリデーション
const spotFormOpts = formOptions({
  defaultValues: {
    description: "",
    name: "",
  },
  onSubmit: ({ value }) => {
    console.log({ value });
  },
+ validators: {
+   onSubmit: ({ value }) => {
+     if (value.name.length === 0) {
+       return {
+         fields: {
+           name: "必ず入力してください",
+         },
+       };
+     }
+   },
+ },
 });
```

実際にフォームバリデーションを実装するときは、[Zod](https://zod.dev/) や [Valibot](https://valibot.dev/) といったスキーマバリデーションライブラリと組み合わせることが多いと思います。上記のサンプルコードで使用していたバリデーション関数は [Standard Schema v1](https://standardschema.dev/json-schema) 型のオブジェクトに置き換えられるため、スキーマバリデーションライブラリで宣言したスキーマをそのまま渡すことができます。

```tsx
const spotFormSchema = z.object({
  description: z
    .string()
    .optional()
    .transform((value) => (value === "" ? undefined : value)),
  // 👆 Zod スキーマは API にデータを送信するときのパーサーとしても利用する。そのため、空文字は未登録として見なす変換処理を追加している
  name: z.string().min(1, "必ず入力してください"),
});

type SpotFormInput = z.input<typeof spotFormSchema>;

const defaultValues: SpotFormInput = {
  description: "",
  name: "",
};
// 👆 validators の型はフォームの初期値から推論されるため、フォームの入力値の型注釈を与える必要がある

const spotFormOpts = formOptions({
  defaultValues,
  onSubmit: ({ value }) => {
    console.log({ value });
  },
  validators: {
    onSubmit: spotFormSchema,
    // 👆 Zod スキーマをそのまま渡せる
  },
});
```

## 共通化

ここまで紹介してきたサンプルコードには、重複コードに起因する以下のような課題があります。

- ダーティーチェックやエラー判定など、**フォーム実装で頻出のパターンだが、実装をミスしやすいコードを繰り返し書く**ことになる
- 複数のフォーム間で**見た目や挙動の整合性を担保しにくい**

こういったコードは 1 箇所に集約してそれをテストすることで、アプリケーション全体の挙動を担保しやすくなります。このセクションでは、TanStack Form の提供する共通化に関する API を紹介します。

### 型パラメータ禁止！

TanStack Form を使用していると、「型パラメータを絶対に使わせない」という鉄の意思を感じます。強い型補完はとても便利でありがたいのですが、フォームの分割や共通化を考えると困ることがあります。例えば、React Hook Form の [useForm](https://www.react-hook-form.com/api/useform/) フックで以下のようなコードを書いたことがある方もいるのではないでしょうか？

```tsx
type ParentFormValues = {
  description: string;
  name: string;
};

function ParentComponent() {
  // 親コンポーネントでフォームを宣言
  const form = useForm({
    defaultValues: { description: "", name: "" },
  });

  return (
    <form onSubmit={form.handleSubmit((value) => console.log(value))}>
      <ChildComponent form={form} />
      {/* 👆 form オブジェクトを子コンポーネントと props で共有する

      {/* ... */}
    </form>
  );
}

type Props = {
  form: ReturnType<typeof useForm<ParentFormValues>>;
  // 👆 useForm フックの型パラメータで受け取れる form オブジェクトの型を絞り込む
};

function ChildComponent() {
  // ...
}
```

一方で TanStack Form の `useForm` フックでは、戻り値の型がいくつものパラメータから複雑に絡み合って推論されます。つまり、以下のコードは型パラメータが不足しているため型エラーになります。

```ts
// Type '<TFormData, TOnMount extends undefined | FormValidateOrFn<TFormData>, TOnChange extends undefined | FormValidateOrFn<TFormData>, TOnChangeAsync extends undefined | FormAsyncValidateOrFn<TFormData>, TOnBlur extends undefined | FormValidateOrFn<TFormData>, TOnBlurAsync extends undefined | FormAsyncValidateOrFn<TFormDa...' has no signatures for which the type argument list is applicable.
type UseFormReturn = ReturnType<
  typeof useForm<{ name: string; description?: string }>
>;
```

:::details useForm フックの型

```ts:useForm.d.ts
export declare function useForm<
  TFormData,
  TOnMount extends undefined | FormValidateOrFn<TFormData>,
  TOnChange extends undefined | FormValidateOrFn<TFormData>,
  TOnChangeAsync extends undefined | FormAsyncValidateOrFn<TFormData>,
  TOnBlur extends undefined | FormValidateOrFn<TFormData>,
  TOnBlurAsync extends undefined | FormAsyncValidateOrFn<TFormData>,
  TOnSubmit extends undefined | FormValidateOrFn<TFormData>,
  TOnSubmitAsync extends undefined | FormAsyncValidateOrFn<TFormData>,
  TOnDynamic extends undefined | FormValidateOrFn<TFormData>,
  TOnDynamicAsync extends undefined | FormAsyncValidateOrFn<TFormData>,
  TOnServer extends undefined | FormAsyncValidateOrFn<TFormData>,
  TSubmitMeta
>(
  opts?: FormOptions<
    TFormData,
    TOnMount,
    TOnChange,
    TOnChangeAsync,
    TOnBlur,
    TOnBlurAsync,
    TOnSubmit,
    TOnSubmitAsync,
    TOnDynamic,
    TOnDynamicAsync,
    TOnServer,
    TSubmitMeta
  >
): ReactFormExtendedApi<
  TFormData,
  TOnMount,
  TOnChange,
  TOnChangeAsync,
  TOnBlur,
  TOnBlurAsync,
  TOnSubmit,
  TOnSubmitAsync,
  TOnDynamic,
  TOnDynamicAsync,
  TOnServer,
  TSubmitMeta
>;
```

:::

このようにライブラリ全体を通じた設計として、**型パラメータではなく値から推論させる**ような API として設計されています。`useForm` フックの値を一度変数に入れて、そこから `typeof` で型を取り出す力技もできますが、素直に TanStack Form の提供する API に乗っかった方が良いと思います。

### フォームの部品を共通化

TanStack Form では [Context API](https://react.dev/learn/passing-data-deeply-with-context#context-an-alternative-to-passing-props) を用いた方法でフォームの部品 (各種フィールドに使うコンポーネントなど) を共通化します。大雑把なイメージとしては、`useForm` フックを拡張したカスタムフックをつくり、その戻り値である `form` オブジェクトがコンテキスト経由で共通コンポーネントを参照するような仕組みと理解してください。ここでは、以下で紹介する 2 つの関数とその戻り値であるフックが重要です。

- [createFormHookContexts](https://tanstack.com/form/latest/docs/framework/react/reference/functions/createFormHookContexts) 関数: アプリケーションで共有するフォームコンテキストと、値を取得するためのフックを返す
- [createFormHook](https://tanstack.com/form/latest/docs/framework/react/reference/functions/createFormHook) 関数: `useForm` を拡張したフックと、コンポーネントを共通化・分割するための [HOC](https://legacy.reactjs.org/docs/higher-order-components.html) を返します

```tsx
import { createFormHook, createFormHookContexts } from "@tanstack/react-form";

// アプリケーションで共有するフォームコンテキストを宣言
const { fieldContext, formContext, useFieldContext, useFormContext } =
  createFormHookContexts();

/*
 * - useAppForm: useForm を拡張したフック
 * - withFieldGroup: フィールドグループを共通化するための HOC
 * - withForm: フォームコンポーネントを分割するための HOC
 */
const { useAppForm, withFieldGroup, withForm } = createFormHook({
  fieldComponents: {
    DateField,
    TextareaField,
    TextField,
    // 👆 フォーム内で利用したいフィールドコンポーネントを登録
  },
  fieldContext,
  formComponents: {
    CancelButton,
    SubmitButton,
    // 👆 フォーム内で利用したい共通コンポーネントを登録
  },
  formContext,
});
```

共通化するコンポーネント側は `useFieldContext` フックでフィールドの状態にアクセスできます。以下の例はテキスト入力に用いるインプットフィールドです。

```tsx
import { useFieldContext } from "@/lib/tanstack-form/form-context";

type Props = PropsWithChildren<{
  label: string;
  placeholder: string;
  required?: boolean;
}> &
  Pick<ComponentProps<"input">, "inputMode">;

function TextField({ inputMode, label, placeholder, required = false }: Props) {
  const field = useFieldContext<string>();
  // 👆 コンテキストからフィールドオブジェクトを取得

  const { errors, isDirty, isValid } = field.state.meta;
  const hasSubmitted = field.form.state.submissionAttempts > 0;
  const isInvalid = (isDirty || hasSubmitted) && !isValid;
  // 👆 繰り返し書いていたメタデータに関するロジックを集約

  return (
    <Field className={styles.base}>
      <FieldLabel htmlFor={field.name}>{label}</FieldLabel>
      <Input
        id={field.name}
        type="text"
        inputMode={inputMode}
        required={required}
        name={field.name}
        value={field.state.value}
        aria-invalid={isInvalid}
        placeholder={placeholder}
        onChange={(e) => {
          field.handleChange(e.target.value);
        }}
        // 👆 Props の指定漏れも発生しにくい
      />
      {isInvalid && <FieldError errors={errors} />}
    </Field>
  );
}
```

:::details 送信ボタンの例

やっていることはテキストフィールドとほぼ同じですが、フォームの状態 (送信状態など) が必要な場合は `useFormContext` フックを使用します。

```tsx
import { useFormContext } from "@/lib/tanstack-form/form-context";

function SubmitButton() {
  const form = useFormContext();
  // 👆 コンテキストからフォームオブジェクトを取得

  return (
    <Button
      type="submit"
      disabled={form.state.isSubmitting}
      className={styles.base}
    >
      送信
    </Button>
  );
}
```

:::

最後にフォームコンポーネントで使用していた `useForm` フックを `useAppForm` に置き換えます。`useAppForm` フックが受け取るパラメーターは変わりませんが、戻り値である `form` オブジェクトが拡張されています。

```tsx
function SpotForm() {
  // useAppForm フックで拡張されたフォームを宣言
  const form = useAppForm(spotFormOpts);

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        void form.handleSubmit(e);
      }}
      className={styles.base}
    >
      <form.AppField name="name">
      {/* 👆 form.AppField に置き換え */}
        {(field) => (
          <field.TextField required label="名前" placeholder="東京港野鳥公園" />
          {/* 👆 コンテキストに登録したフィールドコンポーネントを参照できる */}
        )}
      </form.AppField>
      <form.AppField name="description">
        {(field) => (
          <field.TextareaField
            label="説明"
            placeholder="東京都大田区の野鳥を観察できる公園"
          />
        )}
      </form.AppField>
      <form.AppForm>
      {/* 👆 form.AppForm に置き換え  */}
        <form.SubmitButton />
        {/* 👆 フォームコンテキストに登録したコンポーネントを参照できる */}
      </form.AppForm>
    </form>
  );
}
```

### フィールドグループを共通化

似たような形状のフォームが複数ある場合、その一部をフィールドグループとして括り出して共通コンポーネントにできます。ここで利用するのは `createFormHook` フックの戻り値にあった `withFieldGroup` 関数です。`withFieldGroup` 関数は HOC になっており、戻り値で返すコンポーネントは `form` オブジェクトを受け取れます。これにより、親コンポーネントが持つフォームの状態へアクセスできるという仕組みになっています。

```tsx
// フィールドグループ (子コンポーネント)
const SpotFormFields = withFieldGroup({
  defaultValues,
  // 👆 ここで渡す初期値は型推論にのみ利用される
  render: ({ group }) => (
    <>
      <group.AppField name="name">
        {(field) => (
          <field.TextField required label="名前" placeholder="東京港野鳥公園" />
        )}
      </group.AppField>
      <group.AppField name="description">
        {(field) => (
          <field.TextareaField
            label="説明"
            placeholder="東京都大田区の野鳥を観察できる公園"
          />
        )}
      </group.AppField>
    </>
  ),
});

// フォーム (親コンポーネント)
function SpotForm() {
  const form = useAppForm(spotFormOpts);

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        void form.handleSubmit(e);
      }}
      className={styles.base}
    >
      <SpotFormFields
        form={form}
        fields={{ description: "description", name: "name" }}
      />
      {/* 👆 フィールドグループに form オブジェクトを渡し、マッピングするフィールドを指定する */}
      <form.AppForm>
        <form.SubmitButton />
      </form.AppForm>
    </form>
  );
}
```

https://tanstack.com/form/latest/docs/framework/react/guides/form-composition#reusing-groups-of-fields-in-multiple-forms

## Next.js 連携

このセクションでは Next.js の Server Actions と TanStack Form の連携を扱います。
長いので折りたたみますが、連携前のコードは以下の通りです。説明のため、緯度・経度フィールドを追加しました。

:::details Next.js 連携前のコード

```tsx:spot-form-schema.ts
import z from "zod";

export const spotFormSchema = z.object({
  description: z
    .string()
    .optional()
    .transform((value) => (value === "" ? undefined : value)),
  latitude: z
    // 👆 緯度を追加
    .string()
    .trim()
    .min(1, "必ず入力してください")
    .transform((value) => Number.parseFloat(value))
    .pipe(z.number().min(-90).max(90)),
  // 👆 経度を追加
  longitude: z
    .string()
    .trim()
    .min(1, "必ず入力してください")
    .transform((value) => Number.parseFloat(value))
    .pipe(z.number().min(-180).max(180)),
  name: z.string().min(1, "必ず入力してください"),
});

export type SpotFormInput = z.input<typeof spotFormSchema>;
```

```tsx:spot-form-options.ts
import { formOptions } from "@tanstack/react-form";

import { spotFormSchema } from "./spot-form-schema";

import type { SpotFormInput } from "./spot-form-schema";

export function createSpotFormOptions({
  defaultValues = {
    description: "",
    latitude: "",
    longitude: "",
    name: "",
  },
  onSubmit,
}: {
  defaultValues?: SpotFormInput;
  onSubmit?: () => void;
} = {}) {
  return formOptions({
    defaultValues,
    onSubmit,
    validators: {
      onSubmit: spotFormSchema,
    },
  });
}
```

```tsx:spot-form-fields.tsx
"use client";

import { useAppForm, withFieldGroup } from "@/lib/tanstack-form/app-form";

import { createSpotFormOptions } from "./spot-form-options";

const SpotFormFields = withFieldGroup({
  defaultValues: createSpotFormOptions().defaultValues,
  render: ({ group }) => (
    <>
      <group.AppField name="name">
        {(field) => (
          <field.TextField required label="名前" placeholder="東京港野鳥公園" />
        )}
      </group.AppField>
      <group.AppField name="latitude">
        {(field) => (
          <field.TextField required label="緯度" placeholder="35.584607" />
        )}
      </group.AppField>
      {/* 👆 緯度フィールドを追加 */}
      <group.AppField name="longitude">
        {(field) => (
          <field.TextField required label="経度" placeholder="139.756623" />
        )}
      </group.AppField>
      {/* 👆 経度フィールドを追加 */}
      <group.AppField name="description">
        {(field) => (
          <field.TextareaField
            label="説明"
            placeholder="東京都大田区の野鳥を観察できる公園"
          />
        )}
      </group.AppField>
    </>
  ),
});
```

:::

### ミニマム実装から

まずは最低限必要な構成要素として、以下の 3 つを順に実装します。

1. Server Action: リソース登録アクション
2. 追加・更新用に共通化したフォーム
3. 登録フォーム

Server Action は `ServerFormState` 型 である `ServerValidateError.formState` ) を用いてフォームと状態を共有します。この後出てくるので、抑えておいてください。
まずは Server Action のコードから見ていきます。

```ts:create-spot-action.ts
'use server';

import {
  createServerValidate,
  ServerValidateError,
} from "@tanstack/react-form-nextjs";

import type { ServerFormState } from "@tanstack/react-form-nextjs";

// サーバーサイドのバリデーション関数
const serverValidate = createServerValidate({
  ...createSpotFormOptions(),
  // 👆 ここでも型推論のために defaultValues を渡す
  onServerValidate: spotFormSchema,
});

export async function createSpotAction(
  _previousState: unknown,
  formData: FormData
): Promise<ServerFormState<SpotFormInput, undefined> | undefined> {
  try {
    const parsedFormData = await serverValidate(formData);
    // 👆 パースに成功したら SpotFormInput 型のデータを返し、失敗したら ServerValidateError を投げる
    const payload = spotFormSchema.parse(parsedFormData);
    // 👆 API が期待するデータ型にもう一度パース
    await createSpot({ json: payload });
  } catch (error) {
    if (error instanceof ServerValidateError) {
      return error.formState;
      // 👆 ServerValidateError を補足してフォームの状態を返す
    }

    throw error;
  }
}
```

続いてフロント側のコードを見ていきます。
Next.js のようなサーバーサイドで動作するメタフレームワークとの連携では、React の `useActionState` フックを利用します。そのため、登録・更新フォームで共有する `SpotForm` の props もそれに合わせた形にしておくと良いでしょう。

```tsx:spot-form.tsx
'use client';

type Props = Pick<ComponentProps<"form">, "action"> & {
  formState: ServerFormState<SpotFormInput, undefined> | undefined; // サーバーの状態
  isSubmitting: boolean; // 送信状態
  defaultValues?: SpotFormInput; // フォームの初期値
};

export function SpotForm({ action, defaultValues, formState, isSubmitting }: Props) {
  const formOptions = createSpotFormOptions({ defaultValues });
  const form = useAppForm({
    ...formOptions,
    transform: useTransform(
      (baseForm) => (formState ? mergeForm(baseForm, formState) : baseForm),
      [formState],
    ),
    // 👆 サーバーサイドの状態とフォームの状態をマージ
  });

  return (
    <form action={action} className={styles.base}>
      {/* 👆 action でフォームの値を送信  */}
      <SpotFormFields
        form={form}
        fields={{
          description: "description",
          latitude: "latitude",
          longitude: "longitude",
          name: "name",
        }}
      />
      <form.AppForm>
        <form.SubmitButton isSubmitting={isSubmitting} />
      </form.AppForm>
    </form>
  );
}
```

最後に登録用のフォームです。`useActionState` フックでフォームの状態とアクションを受け取ります。これで最もシンプルな形ではありますが、TanStack Form と Next.js が連動して動くところまで完了です。

```tsx:create-spot-form.tsx
"use client";

import { useActionState } from "react";

import { initialFormState } from "@tanstack/react-form-nextjs";

export function CreateSpotForm() {
  const [formState, action, isPending] = useActionState(
    createSpotAction,
    initialFormState,
  );

  return (
    <SpotForm formState={formState} action={action} isSubmitting={isPending} />
  );
}
```

### サーバーサイドのエラーハンドリング

ここではサーバー側で発生したエラーを関連するフィールドにマッピングする方法を紹介します。サンプルコードに出てくる場所登録 API は、リクエストボディの緯度と経度に一致するリソースが既に存在する場合、リソース重複エラーを返すという挙動を想定します。

```ts
type SpotDuplicatedError = {
  code: "SPOT_COORDINATE_DUPLICATED";
  message: string;
};
```

:::message
Result 型の実装に多少の違いがありますが、以前[こちらの記事](https://zenn.dev/frontendflat/articles/d5ea1c5c533c71#%E3%83%95%E3%83%AD%E3%83%B3%E3%83%88%E3%82%A8%E3%83%B3%E3%83%89%E3%81%A7api%E3%82%A8%E3%83%A9%E3%83%BC%E3%82%92%E3%83%8F%E3%83%B3%E3%83%89%E3%83%AA%E3%83%B3%E3%82%B0)で書いたエラーハンドリングの手法を使います。
:::

先程のサンプルコードで `ServerFormState` 型のオブジェクトをフォームの状態とマージしていることがわかりました。サーバーサイドで発生したカスタムエラーをフィールドエラーとして扱うには、`ServerFormState` 型のオブジェクトを組み立て返す必要があります。
しかし、現時点で試した限りでは、ライブラリ側の実装 (`ServerValidateError`) に any 型があり型推論が効きませんでした。そこで、ユーティリティ関数を実装することで内部の詳細を隠蔽しています。

```ts
import type { DeepKeys, ServerFormState } from "@tanstack/react-form-nextjs";

// ServerFormState を組み立てるユーティリティ関数
export function createServerFormState<T>(
  values: T,
  fields: Partial<Record<DeepKeys<T>, { message: string }[]>>
): ServerFormState<T, undefined> {
  return {
    errorMap: {
      onServer: { fields },
    } as never,
    errors: [],
    values,
  };
}
```

このユーティリティ関数を用いて、[判別可能なエラー型](https://www.typescriptlang.org/docs/handbook/unions-and-intersections.html#discriminating-unions)であるエラーコードとフィールドエラーをマッピングします。このような実装をすることで、API で想定されるエラーハンドリングを取りこぼすリスクを型レベルで検知できます。

```ts
try {
  const parsedFormData = await serverValidate(formData);
  const payload = spotFormSchema.parse(parsedFormData);
  const result = await createSpot({
    json: payload,
  });

  if (result.isErr) {
    switch (result.error.code) {
      case "SPOT_COORDINATE_DUPLICATED": {
        const message = "この座標の場所は既に存在します";

        throw new ServerValidateError({
          formState: createServerFormState(parsedFormData, {
            latitude: [{ message }],
            longitude: [{ message }],
          }),
        });
        // 👆 緯度・経度フィールドにエラーをマッピング
      }
      case "VALIDATION_ERROR":
      case "UNKNOWN_ERROR": {
        throw new Error(
          "予期しないエラーが発生しました。時間をおいて再度お試しください"
        );
      }
      default: {
        result.error satisfies never;
        // 👆 想定されるエラーは型レベルでケース網羅を保証
      }
    }
  }
} catch (error) {
  // ...予期しないエラーのハンドリング
}
```

## フォームの分割

このセクションではマルチステップフォームの実装を通じて、巨大なフォームを小さなコンポーネントに分割するための API を紹介します。
共通化のセクションでは `withFieldGroup` 関数を取り上げましたが、ここで使用するのは `withForm` 関数です。これらのインターフェースはよく似通っていますが、Render Props で受け取るオブジェクトが `fields` ではなく `form` になります。

```ts
import { withFieldGroup, withForm } from "@/lib/tanstack-form/app-form";

// 複数フォームで共有するフィールドグループを作成
withFieldGroup({
  props,
  render: ({ group }) => {
    // ...
  },
});

// 1 つのフォームを分割したコンポーネントを作成
withForm({
  props,
  render: ({ form }) => {
    // ...
  },
});
```

呼び出し側の目線で見ると、`withForm` で定義したコンポーネントにはフィールドを指定する必要がないことがわかります。この違いからも、`withForm` がフォーム分割のための API (子コンポーネントは親のフォーム = スキーマに依存する) であり、`withFieldGroup` が繰り返し登場するフィールドグループを共通化するための API (スキーマ非依存) と言えそうです。

```tsx
// withFieldGroup の場合、スキーマが異なるため fields のマッピングが必要
<SpotFormFields
  form={form}
  fields={{
    description: "description",
    latitude: "latitude",
    longitude: "longitude",
    name: "name",
  }}
/>

// withForm の場合、スキーマを共有するため form オブジェクトだけ渡せば良い
<SpotStep
  form={form}
  onNextButtonClick={() => {
    setCurrentStep("visit");
  }}
/>
```

それでは API の特徴を抑えたところで、関連するファイルをいっきに見ていきます。長くなってしまうので、新しく紹介する部分がないコードは折りたたみました。

:::details フォームのスキーマ・オプション

```ts
// スキーマ
const visitFormSchema = z.object({
  spot: spotFormSchema,
  visit: z.object({
    memo: z.string().trim().min(1, "必ず入力してください"),
    visitDate: z
      .date()
      .optional()
      .refine((value): value is Date => !!value, {
        error: "必ず入力してください",
      })
      .transform((value) => value.toISOString()),
  }),
});

type VisitFormInput = z.input<typeof visitFormSchema>;

// オプションのラッパー関数
function createVisitFormOptions({
  defaultValues = {
    spot: createSpotFormOptions().defaultValues,
    visit: {
      memo: "",
      visitDate: undefined,
    },
  },
}: { defaultValues?: VisitFormInput } = {}) {
  return formOptions({
    defaultValues,
    validators: {
      onSubmit: visitFormSchema,
    },
  });
}
```

:::

```tsx:spot-step.tsx
// Step1: 場所情報の入力
export const SpotStep = withForm({
  ...createVisitFormOptions(),
  props: {
    onNextButtonClick: () => {},
    // 👆 props は呼び出し側で渡した値に上書きされるため、実質的には型推論のために利用しているようなもの。
    //    ただし、ランタイムではスプレッド構文で上書きしているだけなので、型エラーを無視するような書き方をすると実行できてしまうので注意。
  },
  render: ({ form, onNextButtonClick }) => {
    const fields = {
      description: "spot.description",
      latitude: "spot.latitude",
      longitude: "spot.longitude",
      name: "spot.name",
    } as const;

    // 「次へ」ボタンのクリックでバリデーションを行うイベントハンドラ
    const handleNextButtonClick = async () => {
      const fieldNames = Object.values(fields);
      await Promise.all(
        fieldNames.map(async (fieldName) =>
          form.validateField(fieldName, "submit")
        )
      );

      const fieldNameSet = new Set<string>(fieldNames);
      const errorFieldNames = form.state.errorMap.onSubmit
        ? Object.keys(form.state.errorMap.onSubmit)
        : [];
      const hasError = errorFieldNames.some((errorFieldName) =>
        fieldNameSet.has(errorFieldName)
      );

      if (!hasError) {
        onNextButtonClick();
      }
    };

    return (
      <section className={styles.section}>
        <SpotFormFields form={form} fields={fields} />
        {/* 👆 Step1 ではフィールドグループに props を横流しするだけ */}
        <div className={styles.actions}>
          <Button type="button" onClick={() => void handleNextButtonClick()}>
            次へ
          </Button>
        </div>
      </section>
    );
  },
});
```

:::details Step2、Step3 のコンポーネント

```tsx:visit-step.tsx
// Step2: 訪問情報の入力
export const VisitStep = withForm({
  ...createVisitFormOptions(),
  props: {
    onBackButtonClick: () => {},
    onNextButtonClick: () => {},
  },
  render: ({ form, onBackButtonClick, onNextButtonClick }) => {
    const handleNextButtonClick = async () => {
      const fieldNames = Object.values({
        memo: "visit.memo",
        visitDate: "visit.visitDate",
      } as const);
      await Promise.all(
        fieldNames.map(async (fieldName) =>
          form.validateField(fieldName, "submit")
        )
      );

      const fieldNameSet = new Set<string>(fieldNames);
      const errorFieldNames = form.state.errorMap.onSubmit
        ? Object.keys(form.state.errorMap.onSubmit)
        : [];
      const hasError = errorFieldNames.some((errorFieldName) =>
        fieldNameSet.has(errorFieldName)
      );

      if (!hasError) {
        onNextButtonClick();
      }
    };

    return (
      <section className={styles.section}>
        <form.AppField name="visit.visitDate">
          {(field) => <field.DateField required label="訪問日" />}
        </form.AppField>
        {/* 👆 追加で必要になった訪問情報を扱うフィールド */}
        <form.AppField name="visit.memo">
          {(field) => (
            <field.TextareaField label="メモ" placeholder="訪問理由など" />
          )}
        </form.AppField>
        <div className={styles.actionsWithBack}>
          <Button type="button" variant="outline" onClick={onBackButtonClick}>
            戻る
          </Button>
          <Button type="button" onClick={() => void handleNextButtonClick()}>
            進む
          </Button>
        </div>
      </section>
    );
  },
});
```

```tsx:confirm-step.tsx
// Step3: 入力確認
export const ConfirmStep = withForm({
  ...createVisitFormOptions(),
  props: {
    isSubmitting: false,
    onBackButtonClick: () => {},
  },
  render: ({ form, isSubmitting, onBackButtonClick }) => (
    <section className={styles.section}>
      <form.Subscribe selector={(state) => state.values}>
        {/* 👆 親コンポーネントで Activity コンポーネントを使用するため、最新のフォーム状態をサブスクリプションする */}
        {(values) => (
          <dl className={styles.summaryCard}>
            <Row label="名前" value={values.spot.name} />
            <Row label="緯度" value={values.spot.latitude} />
            <Row label="経度" value={values.spot.longitude} />
            <Row label="説明" value={values.spot.description} />
            <Row
              label="訪問日"
              value={
                values.visit.visitDate &&
                values.visit.visitDate.toLocaleDateString("ja-JP")
              }
            />
            <Row label="メモ" value={values.visit.memo} />
          </dl>
        )}
      </form.Subscribe>

      <div className={styles.actionsWithBack}>
        <Button
          type="button"
          variant="outline"
          disabled={isSubmitting}
          onClick={onBackButtonClick}
        >
          戻る
        </Button>
        <Button type="submit" disabled={isSubmitting}>
          登録する
        </Button>
      </div>
    </section>
  ),
});

type RowProps = {
  label: string;
  value: string | undefined;
};

function Row({ label, value }: RowProps) {
  return (
    <div className={styles.summaryRow}>
      <dt className={styles.summaryLabel}>{label}</dt>
      <dd className={styles.summaryValue}>{value}</dd>
    </div>
  );
}
```

:::

親コンポーネントではこれらの各ステップをまとめます。特に目新しい部分はありません。

```tsx:visit-form.tsx
type Step = "confirm" | "spot" | "visit";

export function CreateVisitForm() {
  // ...

  return (
    <div className={styles.base}>
      <ol className={styles.steps}>
        {steps.map((step) => (
          <li
            key={step.id}
            data-current={currentStep === step.id}
            className={styles.step}
          >
            {step.label}
          </li>
        ))}
      </ol>
      <form action={action} className={styles.form}>
        {steps.map((step) => (
          <Activity
            key={step.id}
            mode={step.id === currentStep ? "visible" : "hidden"}
          >
            {step.component}
          </Activity>
        ))}
      </form>
    </div>
  );
}
```

:::details VisitForm の全体

```tsx:visit-form.tsx
type Step = "confirm" | "spot" | "visit";

export function CreateVisitForm() {
  const [currentStep, setCurrentStep] = useState<Step>("spot");

  const [formState, action, isSubmitting] = useActionState(
    createVisitAction,
    initialFormState
  );
  const form = useAppForm({
    ...createVisitFormOptions(),
    transform: useTransform(
      (baseForm) => (formState ? mergeForm(baseForm, formState) : baseForm),
      [formState]
    ),
  });

  const steps = [
    {
      component: (
        <SpotStep
          form={form}
          onNextButtonClick={() => {
            setCurrentStep("visit");
          }}
        />
      ),
      id: "spot",
      label: "場所を入力",
    },
    {
      component: (
        <VisitStep
          form={form}
          onBackButtonClick={() => {
            setCurrentStep("spot");
          }}
          onNextButtonClick={() => {
            setCurrentStep("confirm");
          }}
        />
      ),
      id: "visit",
      label: "訪問情報を入力",
    },
    {
      component: (
        <ConfirmStep
          form={form}
          isSubmitting={isSubmitting}
          onBackButtonClick={() => {
            setCurrentStep("visit");
          }}
        />
      ),
      id: "confirm",
      label: "入力確認",
    },
  ] as const satisfies { component: ReactElement; id: Step; label: string }[];

  return (
    <div className={styles.base}>
      <ol className={styles.steps}>
        {steps.map((step) => (
          <li
            key={step.id}
            data-current={currentStep === step.id}
            className={styles.step}
          >
            {step.label}
          </li>
        ))}
      </ol>
      <form action={action} className={styles.form}>
        {steps.map((step) => (
          <Activity
            key={step.id}
            mode={step.id === currentStep ? "visible" : "hidden"}
          >
            {step.component}
          </Activity>
        ))}
      </form>
    </div>
  );
}
```

:::

## おわりに

今回は TanStack Form の使い方を一通り見ていきました。フォームの実装はユーザーやサーバーとの界面になるため、個人的にはフロントエンドの実装の中で難しい方だと感じています。業務でフォームライブラリの選定をすることがあれば、今回学んだ内容を役立てていきたいです。
