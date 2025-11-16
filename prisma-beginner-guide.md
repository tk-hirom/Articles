# Prismaとは？Next.js初心者のためのデータベース入門

## はじめに

Next.jsでアプリケーションを作っていると、「データをどこに保存するか」という課題に直面します。ユーザー情報、ブログ記事、商品データなど、アプリケーションに必要なデータをデータベースに保存し、効率的に取り出す必要があります。

**Prisma**は、そんなデータベース操作を簡単にしてくれるツールです。生のSQLを書かなくても、JavaScriptやTypeScriptのコードでデータベースを扱えるようになります。

## Prismaの3つの特徴

### 1. 型安全なデータベース操作

TypeScriptを使っている場合、Prismaは自動的に型定義を生成してくれます。これにより、VSCodeなどのエディタで自動補完が効き、タイプミスによるバグを防げます。

```typescript
// Prismaを使った例
const user = await prisma.user.findUnique({
  where: { id: 1 }
});
// userの型が自動的に推論される！
console.log(user.email); // 自動補完が効く
```

### 2. データベースの種類を気にしなくていい

PostgreSQL、MySQL、SQLite、MongoDB など、様々なデータベースに対応しています。データベースを変更しても、コードをほとんど書き換える必要がありません。

### 3. マイグレーション機能

データベースのテーブル構造を変更する際、Prismaが自動的に変更履歴を管理してくれます。チーム開発でも、全員が同じデータベース構造を共有できます。

## Prismaのインストール

Next.jsプロジェクトにPrismaをインストールしましょう。

```bash
# Prismaのインストール
npm install prisma --save-dev
npm install @prisma/client

# Prismaの初期化
npx prisma init
```

`npx prisma init` を実行すると、以下のファイルが生成されます：

- `prisma/schema.prisma` - データベースの設計図
- `.env` - データベース接続情報

## Prismaの独自ファイル解説

Prismaを使う上で理解しておくべき重要なファイルがいくつかあります。それぞれ詳しく見ていきましょう。

### 1. `schema.prisma` - データベースの設計図

`prisma/schema.prisma` は、Prismaで最も重要なファイルです。このファイルで、データベースの構造を定義します。

#### ファイルの構成

```prisma
// データベース接続の設定
datasource db {
  provider = "postgresql"  // 使用するデータベースの種類
  url      = env("DATABASE_URL")  // 接続URL（.envから読み込む）
}

// Prisma Clientの生成設定
generator client {
  provider = "prisma-client-js"  // JavaScriptクライアントを生成
}

// データモデルの定義
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())
  posts     Post[]
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  authorId  Int
  author    User     @relation(fields: [authorId], references: [id])
}
```

#### 各セクションの説明

##### datasource ブロック

- `provider`: 使用するデータベースの種類（postgresql, mysql, sqlite, mongodb など）
- `url`: データベースへの接続文字列（通常は環境変数から読み込む）

##### generator ブロック

- Prisma Clientの生成方法を指定
- 通常は `prisma-client-js` を使用

##### model ブロック

- データベースのテーブルに対応
- 各フィールドが列（カラム）になる

#### よく使う型とデコレータ

**基本的なデータ型：**

- `String` - 文字列
- `Int` - 整数
- `Float` - 小数
- `Boolean` - 真偽値
- `DateTime` - 日時
- `Json` - JSON形式のデータ

**主要なデコレータ：**

- `@id` - 主キー（Primary Key）
- `@unique` - 一意制約（重複を許さない）
- `@default()` - デフォルト値
- `@relation()` - リレーション（他のテーブルとの関連）
- `?` - オプショナル（nullを許可）

### 2. `.env` - 環境変数ファイル

データベース接続情報などの機密情報を保存します。**このファイルはGitにコミットしないでください！**

```env
# PostgreSQLの例
DATABASE_URL="postgresql://user:password@localhost:5432/mydb?schema=public"

# SQLiteの例（開発環境で便利）
DATABASE_URL="file:./dev.db"

# PlanetScaleの例（本番環境）
DATABASE_URL="mysql://user:password@host.connect.psdb.cloud/database?sslaccept=strict"
```

### 3. `migrations/` - マイグレーションファイル

`prisma/migrations/` ディレクトリには、データベース構造の変更履歴が保存されます。

```text
prisma/
  migrations/
    20231115120000_init/
      migration.sql
    20231116090000_add_user_profile/
      migration.sql
    migration_lock.toml
```

各マイグレーションフォルダには：

- タイムスタンプ付きの名前
- 実際のSQL文が書かれた `migration.sql`

**重要：** マイグレーションファイルは手動で編集しないでください。Prismaが自動生成します。

### 4. `node_modules/.prisma/client/` - 生成されたクライアント

`npx prisma generate` を実行すると、`schema.prisma` を基に型定義付きのクライアントコードが自動生成されます。このディレクトリは触る必要はありませんが、ここに生成されたコードを使ってデータベース操作を行います。

## Prismaの基本的な使い方

### 1. スキーマの定義

まず、`schema.prisma` でデータモデルを定義します。

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  createdAt DateTime @default(now())
}
```

### 2. マイグレーションの実行

```bash
# マイグレーションファイルを作成し、データベースに適用
npx prisma migrate dev --name init
```

このコマンドは：

1. schema.prismaの変更を検出
2. SQLマイグレーションファイルを生成
3. データベースに変更を適用
4. Prisma Clientを再生成

### 3. Prisma Clientの使用

Next.jsのAPIルートやServer Componentsで使います。

```typescript
// lib/prisma.ts - Prismaクライアントのシングルトン
import { PrismaClient } from '@prisma/client';

const globalForPrisma = global as unknown as { prisma: PrismaClient };

export const prisma =
  globalForPrisma.prisma ||
  new PrismaClient({
    log: ['query'],
  });

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma;
```

```typescript
// app/api/users/route.ts - APIルートの例
import { prisma } from '@/lib/prisma';
import { NextResponse } from 'next/server';

export async function GET() {
  const users = await prisma.user.findMany();
  return NextResponse.json(users);
}

export async function POST(request: Request) {
  const body = await request.json();
  const user = await prisma.user.create({
    data: {
      email: body.email,
      name: body.name,
    },
  });
  return NextResponse.json(user);
}
```

## よく使うPrisma操作

### データの作成（Create）

```typescript
// 1件作成
const user = await prisma.user.create({
  data: {
    email: 'user@example.com',
    name: 'John Doe',
  },
});

// 複数件作成
const users = await prisma.user.createMany({
  data: [
    { email: 'user1@example.com', name: 'User 1' },
    { email: 'user2@example.com', name: 'User 2' },
  ],
});
```

### データの読み取り（Read）

```typescript
// 全件取得
const allUsers = await prisma.user.findMany();

// 条件付き取得
const users = await prisma.user.findMany({
  where: {
    email: {
      contains: '@example.com',
    },
  },
  orderBy: {
    createdAt: 'desc',
  },
  take: 10, // 最初の10件
});

// 1件取得（ユニークな条件）
const user = await prisma.user.findUnique({
  where: {
    email: 'user@example.com',
  },
});

// 1件取得（任意の条件）
const user = await prisma.user.findFirst({
  where: {
    name: 'John',
  },
});
```

### データの更新（Update）

```typescript
// 1件更新
const user = await prisma.user.update({
  where: {
    id: 1,
  },
  data: {
    name: 'Updated Name',
  },
});

// 複数件更新
const result = await prisma.user.updateMany({
  where: {
    email: {
      contains: '@old-domain.com',
    },
  },
  data: {
    email: {
      // 注意: updateManyでは単純な値の設定のみ可能
    },
  },
});
```

### データの削除（Delete）

```typescript
// 1件削除
const user = await prisma.user.delete({
  where: {
    id: 1,
  },
});

// 複数件削除
const result = await prisma.user.deleteMany({
  where: {
    createdAt: {
      lt: new Date('2023-01-01'),
    },
  },
});
```

### リレーションを含む操作

```typescript
// ネストしたデータの作成
const user = await prisma.user.create({
  data: {
    email: 'user@example.com',
    name: 'John',
    posts: {
      create: [
        { title: 'First Post', content: 'Hello World' },
        { title: 'Second Post', content: 'Prisma is awesome' },
      ],
    },
  },
});

// リレーションを含む取得
const userWithPosts = await prisma.user.findUnique({
  where: {
    id: 1,
  },
  include: {
    posts: true, // 関連するPostsも取得
  },
});

// 特定のフィールドのみ取得
const user = await prisma.user.findUnique({
  where: {
    id: 1,
  },
  select: {
    id: true,
    email: true,
    posts: {
      select: {
        title: true,
      },
    },
  },
});
```

## 実践的なTips

### 1. 開発環境ではSQLiteを使う

最初はSQLiteを使うと簡単に始められます。

```prisma
datasource db {
  provider = "sqlite"
  url      = "file:./dev.db"
}
```

### 2. Prisma Studioでデータを確認

```bash
npx prisma studio
```

ブラウザでGUIが開き、データベースの中身を確認・編集できます。

### 3. スキーマを変更したら

```bash
# 開発環境
npx prisma migrate dev --name 変更内容の説明

# 本番環境
npx prisma migrate deploy
```

### 4. データベースをリセットしたいとき

```bash
npx prisma migrate reset
```

**注意：** すべてのデータが削除されます！

### 5. シードデータの投入

`prisma/seed.ts` を作成：

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

async function main() {
  const user = await prisma.user.create({
    data: {
      email: 'admin@example.com',
      name: 'Admin User',
    },
  });
  console.log({ user });
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

`package.json` に追加：

```json
{
  "prisma": {
    "seed": "ts-node --compiler-options {\"module\":\"CommonJS\"} prisma/seed.ts"
  }
}
```

実行：

```bash
npx prisma db seed
```

## よくある問題と解決方法

### エラー: "Can't reach database server"

- データベースが起動しているか確認
- `.env` の `DATABASE_URL` が正しいか確認
- ネットワーク接続を確認

### エラー: "Migration is in a failed state"

```bash
npx prisma migrate resolve --rolled-back マイグレーション名
```

### 型が更新されない

```bash
npx prisma generate
```

## Next.jsでの実装例

完全な例として、簡単なブログアプリを作ってみましょう。

### 1. スキーマ定義

```prisma
model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String
  published Boolean  @default(false)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

### 2. APIルート

```typescript
// app/api/posts/route.ts
import { prisma } from '@/lib/prisma';
import { NextResponse } from 'next/server';

export async function GET() {
  const posts = await prisma.post.findMany({
    where: { published: true },
    orderBy: { createdAt: 'desc' },
  });
  return NextResponse.json(posts);
}

export async function POST(request: Request) {
  const json = await request.json();
  const post = await prisma.post.create({
    data: {
      title: json.title,
      content: json.content,
      published: json.published ?? false,
    },
  });
  return NextResponse.json(post);
}
```

### 3. Server Component

```typescript
// app/posts/page.tsx
import { prisma } from '@/lib/prisma';

export default async function PostsPage() {
  const posts = await prisma.post.findMany({
    where: { published: true },
    orderBy: { createdAt: 'desc' },
  });

  return (
    <div>
      <h1>ブログ記事一覧</h1>
      {posts.map((post) => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>{post.content}</p>
          <time>{post.createdAt.toLocaleDateString('ja-JP')}</time>
        </article>
      ))}
    </div>
  );
}
```

## まとめ

Prismaを使うことで：

✅ 型安全にデータベース操作ができる  
✅ SQLを直接書かなくても良い  
✅ データベースの種類を気にしなくて良い  
✅ マイグレーションが簡単  
✅ チーム開発がしやすい

Next.jsとPrismaの組み合わせは、モダンなWebアプリケーション開発の強力な選択肢です。まずは小さなプロジェクトから始めて、徐々に慣れていきましょう！

## 参考リンク

- [Prisma公式ドキュメント](https://www.prisma.io/docs)
- [Next.js with Prisma](https://www.prisma.io/nextjs)
- [Prisma Examples](https://github.com/prisma/prisma-examples)
