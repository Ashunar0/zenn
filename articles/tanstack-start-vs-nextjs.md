---
title: "Next.jsはもう要らない？TanStack Startに入門してみた"
emoji: "🏎️"
type: "tech"
topics: ["tanstackstart", "nextjs", "react", "typescript"]
published: true
published_at: 2026-02-20 08:00
---

TanStack Startが2025年秋にv1 RCを出した。

TanStackといえばQuery、Router、Table…と、Reactエコシステムでは定番のライブラリ群だ。そこが本気でフルスタックフレームワークを作ったらしい。しかも「Next.jsより型安全」「キャッシュが素直」みたいな評判がちらほら聞こえてくる。

気になったので実際にTODOアプリまで作ってみた（v1.161.1で検証）。Next.jsユーザーの自分が触ってみて感じた違いを紹介する。

## TanStack Startとは

[TanStack Start](https://tanstack.com/start)は、TanStack Routerベースのフルスタックフレームワーク。Vite + Nitroで動く。

Next.jsとの一番の違いは**思想**だ。

- **Next.js** = オートマ車。乗れば走る。でもエンジンの中身は見えない
- **TanStack Start** = マニュアル車。全部自分で操作する。だからこそ仕組みがわかる

Next.jsは**サーバーファースト**（Server Componentがデフォルト）。TanStack Startは**クライアントファースト**（SPAにSSRをopt-inする思想）。素のReactを書いてる感覚のまま、フルスタックになる。RemixやReact Routerに近い。

## プロジェクト構造の比較

まず全体像を掴むために、主要ファイルを並べてみる。

| TanStack Start     | 役割                    | Next.js        |
| ------------------ | ----------------------- | -------------- |
| `__root.tsx`       | レイアウト + HTML shell | `layout.tsx`   |
| `routes/index.tsx` | ページ                  | `app/page.tsx` |
| `routeTree.gen.ts` | ルートの型を自動生成    | **無い**       |
| `router.tsx`       | ルーター設定を明示      | **無い**       |

注目すべきは**Next.jsに無い2つのファイル**だ。

- `routeTree.gen.ts` — ファイルを追加するたびに自動更新される。有効なルートがTypeScriptの型として生成される
- `router.tsx` — スクロール復元、プリフェッチ戦略などを自分で書く。Next.jsだとこの辺はフレームワークが裏で管理している

この「暗黙 vs 明示」の違いが、これから紹介する3つの驚きに全部繋がっている。

## 驚き①: 型安全ルーティング — タイポが型エラーになる

一番興味深かったのがこれだ。

TanStack Startでは、ファイルを追加すると`routeTree.gen.ts`が自動更新されて、有効なパスがTypeScriptの型として生成される。

```typescript
// routeTree.gen.ts（自動生成される）
export interface FileRoutesByFullPath {
  "/": typeof IndexRoute;
  "/about": typeof AboutRoute;
  "/counter": typeof CounterRoute;
  "/posts": typeof PostsRoute;
  "/todos": typeof TodosRoute;
}
```

つまりこうなる。

```tsx
// ✅ 型チェックOK
<Link to="/todos">TODOアプリ</Link>

// ❌ 型エラー！ "/abuot" は有効なパスじゃない
<Link to="/abuot">About</Link>
```

Next.jsにも`typedRoutes`という設定があり、有効にすれば同様の型チェックができる。ただしデフォルトでは`<Link href="/abuot">`はただの文字列で、設定を知らなければ素通りする。TanStack Startはこれが**ゼロコンフィグで最初から有効**になっている。

さらに、ページを削除してリンクが残っていても、TypeScriptが全箇所を赤線で教えてくれる。ページが増えれば増えるほど、この安全網の価値は上がる。地味に嬉しい。

## 驚き②: loaderの型伝播 — 入口で型をつければ最後まで繋がる

TanStack Startのデータフェッチは、Remix由来の**loaderパターン**を使う。

```tsx
type Post = { id: number; title: string; body: string };

const getPosts = createServerFn({ method: "GET" }).handler(async () => {
  const res = await fetch(
    "https://jsonplaceholder.typicode.com/posts?_limit=10",
  );
  return res.json() as Promise<Post[]>; // ← ここで1回だけ型をつける
});

export const Route = createFileRoute("/posts")({
  loader: () => getPosts(),
  component: PostsPage,
});

function PostsPage() {
  const posts = Route.useLoaderData(); // ← Post[] が自動で伝播する！
  return posts.map((post) => <div key={post.id}>{post.title}</div>);
}
```

ポイントは、**`as Post[]`を入口で1回書けば、`useLoaderData()`の返り値まで型が自動で伝播する**こと。コンポーネント側で`as`は不要だ。

Next.jsだとこうなる。見覚えのある人も多いんじゃないだろうか。

```tsx
// Next.js — Server/Client Componentの境界で型定義が必要になる
// Server Component (page.tsx)
async function PostsPage() {
  const res = await fetch("https://jsonplaceholder.typicode.com/posts");
  const posts = (await res.json()) as Post[]; // ← 1回目
  return <PostList posts={posts} />;
}

// Client Component — 検索やソートなどインタラクティブな処理があるとこうなる
"use client";
function PostList({ posts }: { posts: Post[] }) {
  // ← 2回目（propsの型定義）
  const [query, setQuery] = useState("");
  const filtered = posts.filter((p) => p.title.includes(query));
  return filtered.map((post) => <div key={post.id}>{post.title}</div>);
}
```

Server ComponentとClient Componentの境界がある以上、propsの型定義は避けられない（これはNext.js固有の問題というよりReactのコンポーネント合成の話だ）。TanStack Startにはこの境界がないので、loaderからコンポーネントまで直結する。

もう一つ。loaderは**ページ遷移前に**実行される。データが揃ってから画面が切り替わるので、ローディングスピナーが出にくい。`defaultPreload: 'intent'`を設定すればリンクをホバーした瞬間にデータを先読みするので、クリックしたら一瞬で表示される。

## 驚き③: createServerFn — 「これはサーバーで動く」が明示的

Next.jsのServer Actionsは`'use server'`ディレクティブで定義する。TanStack Startでは`createServerFn()`という関数で明示的に作る。

TODOアプリを作ったときのコードがこれだ。

```tsx
const addTodo = createServerFn({ method: "POST" })
  .inputValidator((data: { text: string }) => {
    if (!data.text.trim()) throw new Error("テキストが空です");
    return data;
  })
  .handler(async ({ data }) => {
    todos.push({ id: nextId++, text: data.text, completed: false });
  });
```

`createServerFn()`で「サーバーで動く関数」を作り、`.inputValidator()`でバリデーションを定義し、`.handler()`で処理を書く。全部メソッドチェーンで1箇所に集まる。

これが重要な理由がある。TanStack Startのloaderは、**SSR時はサーバーで、クライアントナビゲーション時はブラウザで**実行される。つまり素の関数をloaderに書くと、クライアントナビゲーション時にブラウザで動いてしまう。`createServerFn`で包めば、クライアントから呼ばれたときに自動でHTTPリクエストに変換してくれる。

Next.jsのServer Componentは常にサーバーで実行されるので、この問題を意識しなくていい。ここもマニュアル車の特徴だ。

### ミューテーション後の再取得

データを変更したあと、画面を更新するにはどうするか。

```tsx
const router = useRouter();

const handleAdd = async () => {
  await addTodo({ data: { text: newTodo } });
  router.invalidate(); // ← 「loaderのキャッシュが古くなったから再取得して」
};
```

`router.invalidate()`で、**クライアント側から**loaderの再実行を指示する。Next.jsの`revalidatePath()`はServer Action内からサーバー側のキャッシュを無効化し、結果的にクライアント側にも反映される。アプローチが違うだけで結果は同じだが、TanStack Startの方が「クライアントからloaderを叩き直す」という挙動が直感的でわかりやすいと感じた。

## 正直な感想

良いことばかり書いてきたが、正直なところも書いておく。

**コード量は確実に増える。** 明示的にやる = 書く量が多い。Next.jsなら裏でやってくれることを全部自分で書くので、同じTODOアプリでもコードが長くなる。

**初心者にはきつい。** ルーター設定、loaderの実行タイミング、createServerFnの必要性…。これらはNext.jsで一通り経験した人なら「あーこれを自分で書くのか」と理解できるが、フレームワーク初体験の人には辛い。

**でも、「使ってるけど中身わかってない」が全部可視化される。** これが一番の価値だと思う。Next.jsのキャッシュ、ルーティング、サーバー実行が裏で何をやってるか、TanStack Startを触ると手触りで理解できる。

**個人開発で好き放題やりたい人にはかなり良い。** フレームワークの「お作法」に縛られず、全部自分でコントロールできる。ルーター設定もキャッシュ戦略も自分で決められるので、「なんか勝手にこうなる」がない。

## まとめ: Next.jsはいらない？

タイトルで煽っておいてなんだが、**Next.jsはいらないとは思わない**。

自分の結論はこうだ。

- **不特定多数に見せるプロダクト**（SEO・OGP必要）→ Next.js
- **ログイン後のみのアプリ** → Vite + Reactで十分
- **仕組みを理解してフルコントロールしたい** → TanStack Start

仕事ではNext.jsが安泰だと思う。エコシステム、ドキュメント、採用事例、どれを取っても圧倒的だ。

ただ、**マニュアル車に乗る経験は、オートマ車の運転も上手くする**。Next.jsの裏側で何が起きてるか知りたい人は、一度TanStack Startを触ってみてほしい。
