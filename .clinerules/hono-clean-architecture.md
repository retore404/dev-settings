# Hono + クリーンアーキテクチャ開発ルール

## このファイルの目的

新しいHonoアプリケーションを開発する際に従うべきクリーンアーキテクチャの設計原則とディレクトリ構成ルールを定義します。

**⚠️ 重要: Honoアプリ開発時は必ずこのルールに従って実装すること**

## 必須技術スタック

### Core Framework
- **Hono**: Webフレームワーク
- **TypeScript**: 型安全性のため必須
- **@hono/zod-openapi**: API仕様定義とバリデーション
- **@hono/swagger-ui**: API仕様のドキュメント化

### Testing & Quality
- **Vitest**: テストフレームワーク
- **@vitest/coverage-v8**: コードカバレッジ
- **Biome**: Linter & Formatter

### Database & ORM
- **Drizzle ORM**: データベースORM
- **Zod**: スキーマバリデーション

## 必須ディレクトリ構成

```
src/
├── index.ts                         # エントリーポイント・DI設定
├── domain/                          # ドメイン層（最重要）
│   ├── entities/                   # ビジネスエンティティ
│   │   ├── {Entity}.ts             # エンティティクラス
│   │   ├── {Entity}.test.ts        # エンティティテスト
│   │   └── index.ts                # Public API
│   ├── repositories/               # リポジトリインターフェース
│   │   ├── {Entity}Repository.ts   # リポジトリIF
│   │   └── index.ts                # Public API
│   ├── services/                   # ドメインサービス
│   │   ├── {Service}.ts            # ドメインサービス
│   │   └── index.ts                # Public API
│   ├── errors/                     # ドメインエラー定義
│   │   ├── DomainError.ts          # エラークラス階層
│   │   └── index.ts                # Public API
│   ├── types/                      # ドメイン型定義
│   │   ├── {domain}.ts             # ドメイン固有型
│   │   └── index.ts                # Public API
│   └── index.ts                    # Domain Layer Public API
├── usecase/                        # アプリケーション層
│   └── {domain}/                   # ドメイン別グルーピング
│       ├── {Action}UseCase.ts      # ユースケース実装
│       ├── {Action}UseCase.test.ts # ユースケーステスト
│       └── index.ts                # Public API
├── interface/                      # インターフェース層
│   ├── controllers/                # HTTPコントローラー
│   │   ├── {Entity}Controller.ts   # コントローラー実装
│   │   ├── {Entity}Controller.test.ts # コントローラーテスト
│   │   └── index.ts                # Public API
│   ├── routes/                     # OpenAPIルート定義
│   │   ├── api.ts                  # ルート集約
│   │   ├── {domain}/               # ドメイン別ルート
│   │   │   ├── {action}.ts         # 各エンドポイント定義
│   │   │   └── index.ts            # ドメインルート集約
│   │   └── index.ts                # Public API
│   └── types/                      # API型定義
│       ├── context.ts              # Honoコンテキスト型
│       ├── error.ts                # APIエラー型
│       ├── {domain}.ts             # API入出力型
│       └── index.ts                # Public API
└── infrastructure/                 # インフラストラクチャ層
    ├── database/                   # データベース
    │   ├── connection.ts           # DB接続設定
    │   ├── schema.ts               # DBスキーマ定義
    │   └── index.ts                # Public API
    ├── repositories/               # リポジトリ実装
    │   └── {domain}/
    │       └── {Entity}RepositoryImpl.ts
    ├── auth/                       # 認証機構
    │   ├── {provider}.ts           # 認証プロバイダー実装
    │   └── index.ts                # Public API
    └── types/                      # インフラ型定義
        ├── database.ts             # DB関連型
        └── index.ts                # Public API
```

## 依存関係ルール（厳守）

### 基本原則
**内側の層は外側の層を知ってはいけない（依存関係逆転の原則）**

```
interface → usecase → domain ← infrastructure
              ↑           ↑
              └─────────────┘
```

### 依存可能性マトリックス

| 層 | domain | usecase | interface | infrastructure |
|---|---|---|---|---|
| **domain** | ✅ | ❌ | ❌ | ❌ |
| **usecase** | ✅ | ✅ | ❌ | ❌ |
| **interface** | ✅ | ✅ | ✅ | ❌ |
| **infrastructure** | ✅ | ❌ | ❌ | ✅ |

### 絶対に守るべきルール

1. **ドメイン層は何にも依存しない**
   - 外部ライブラリへの依存も最小限
   - ビジネスルールのみに集中

2. **ユースケース層はドメイン層のみに依存**
   - インフラの実装詳細を知らない
   - リポジトリはインターフェースで受け取る

3. **インターフェース層は下位層のみに依存**
   - インフラ層を直接参照しない
   - DIコンテナで依存性を注入

4. **インフラ層はドメイン層のインターフェースを実装**
   - 上位層のビジネスロジックを知らない

## 必須実装パターン

### 1. エントリーポイント（index.ts）

```typescript
import { Hono } from 'hono';
import { prettyJSON } from 'hono/pretty-json';
import { cors } from 'hono/cors';
import { ZodError } from 'zod';
import { HTTPException } from 'hono/http-exception';
import type { AppVariables } from './interface/types/context';
import { api } from './interface/routes/api';

// 依存注入のインポート
import { EntityRepositoryImpl } from './infrastructure/repositories/domain/EntityRepositoryImpl';
import { ActionUseCase } from './usecase/domain/ActionUseCase';
import { EntityController } from './interface/controllers/EntityController';

export const app = new Hono<{ Variables: AppVariables }>();

// 必須ミドルウェア
app.use(prettyJSON());
app.use('/*', cors({ origin: [...] }));

// エラーハンドリング
app.onError((err, c) => {
  if (err instanceof ZodError) {
    return c.json({ message: err.issues[0].message }, 400);
  }
  if (err instanceof HTTPException) {
    return c.json({ message: err.message }, err.status);
  }
  return c.json({ message: 'Internal Server Error' }, 500);
});

// 依存注入ミドルウェア
app.use('/api/entities/*', async (c, next) => {
  const repository = new EntityRepositoryImpl(c);
  const useCase = new ActionUseCase(repository);
  const controller = new EntityController(useCase);

  c.set('entityController', controller);
  await next();
});

// API認証ミドルウェア
app.use('/api/*', async (c, next) => {
  // 認証が不要なエンドポイントを除外
  if (c.req.path !== '/api/doc' && c.req.path !== '/api/specification') {
    return verifyToken(c, next);
  }
  await next();
});

app.route('/api', api);
export default app;
```

### 2. ドメインエンティティ

```typescript
import { DomainValidationError } from '../errors';

export class Entity {
  private readonly _id: string;
  private readonly _name: string;

  constructor(id: string, name: string) {
    // バリデーション
    if (!id?.trim()) {
      throw new DomainValidationError('IDは必須です', 'id');
    }
    if (!name?.trim()) {
      throw new DomainValidationError('名前は必須です', 'name');
    }

    this._id = id;
    this._name = name.trim();
  }

  get id(): string { return this._id; }
  get name(): string { return this._name; }

  // ビジネスロジック
  isValidForOperation(): boolean {
    return this._name.length >= 3;
  }
}
```

### 3. ドメインエラー階層

```typescript
// domain/errors/DomainError.ts
export abstract class DomainError extends Error {
  abstract readonly code: string;

  constructor(message: string) {
    super(message);
    this.name = this.constructor.name;
    if (Error.captureStackTrace) {
      Error.captureStackTrace(this, this.constructor);
    }
  }
}

export class DomainValidationError extends DomainError {
  readonly code = 'DOMAIN_VALIDATION_ERROR';

  constructor(message: string, public readonly field: string) {
    super(message);
  }
}

export class BusinessRuleViolationError extends DomainError {
  readonly code = 'BUSINESS_RULE_VIOLATION';

  constructor(message: string, public readonly rule: string) {
    super(message);
  }
}

export class EntityNotFoundError extends DomainError {
  readonly code = 'ENTITY_NOT_FOUND';

  constructor(entityName: string, identifier: string) {
    super(`${entityName} with identifier '${identifier}' not found`);
  }
}
```

### 4. リポジトリインターフェース

```typescript
// domain/repositories/EntityRepository.ts
import type { Entity } from '../entities/Entity';

export interface EntityRepository {
  save(entity: Entity): Promise<void>;
  findById(id: string): Promise<Entity | null>;
  findByConditions(conditions: SearchConditions): Promise<Entity[]>;
  update(entity: Entity): Promise<boolean>;
  delete(id: string): Promise<boolean>;
}
```

### 5. ユースケース実装

```typescript
// usecase/domain/ActionUseCase.ts
import type { EntityRepository } from '../../domain/repositories/EntityRepository';
import { Entity } from '../../domain/entities/Entity';
import { v7 as uuidv7 } from 'uuid';

export interface ActionInput {
  name: string;
  // その他の入力フィールド
}

export interface ActionOutput {
  id: string;
}

export class ActionUseCase {
  constructor(private repository: EntityRepository) {}

  async execute(input: ActionInput): Promise<ActionOutput> {
    // 1. ビジネスロジック実行
    const id = uuidv7();
    const entity = new Entity(id, input.name);

    // 2. 永続化
    await this.repository.save(entity);

    // 3. 結果返却
    return { id };
  }
}
```

### 6. コントローラー実装

```typescript
// interface/controllers/EntityController.ts
import type { Context } from 'hono';
import type { AppVariables } from '../types/context';
import type { ActionUseCase } from '../../usecase/domain/ActionUseCase';
import { DomainError, DomainValidationError } from '../../domain/errors';

export class EntityController {
  constructor(private actionUseCase: ActionUseCase) {}

  private handleDomainError(c: Context, error: DomainError) {
    console.error(`Domain Error [${error.code}]:`, error.message);

    if (error instanceof DomainValidationError) {
      throw new Error(`Validation Error: ${error.message}`);
    }
    // その他のエラーハンドリング...
    throw new Error(`Domain Error: ${error.message}`);
  }

  async executeAction(c: Context<{ Variables: AppVariables }>) {
    try {
      const requestBody = await c.req.json();

      const result = await this.actionUseCase.execute({
        name: requestBody.name,
      });

      return c.json({ id: result.id }, 201);
    } catch (error) {
      if (error instanceof DomainError) {
        this.handleDomainError(c, error);
      }

      console.error('Unexpected error:', error);
      return c.json({ message: 'Internal Server Error' }, 500);
    }
  }
}
```

### 7. OpenAPIルート定義

```typescript
// interface/routes/domain/action.ts
import { createRoute } from '@hono/zod-openapi';
import { ActionRequest, ActionResponse } from '../../types/domain';
import { ErrorResponse } from '../../types/error';

export const actionRoute = createRoute({
  path: '/',
  method: 'post',
  description: 'アクションを実行します',
  tags: ['Entity'],
  request: {
    headers: AuthHeaderSchema,
    body: {
      content: {
        'application/json': {
          schema: ActionRequest,
        },
      },
    },
  },
  responses: {
    201: {
      content: {
        'application/json': {
          schema: ActionResponse,
        },
      },
      description: 'アクション実行成功',
    },
    400: {
      content: {
        'application/json': {
          schema: ErrorResponse,
        },
      },
      description: 'バリデーションエラー',
    },
  },
});
```

## 必須テスト戦略

### 1. エンティティテスト
```typescript
// domain/entities/Entity.test.ts
import { describe, test, expect } from 'vitest';
import { Entity } from './Entity';
import { DomainValidationError } from '../errors';

describe('Entity', () => {
  test('正常なエンティティが作成できる', () => {
    const entity = new Entity('id', 'name');
    expect(entity.id).toBe('id');
    expect(entity.name).toBe('name');
  });

  test('IDが空の場合DomainValidationErrorが発生する', () => {
    expect(() => new Entity('', 'name'))
      .toThrow(DomainValidationError);
  });
});
```

### 2. ユースケーステスト
```typescript
// usecase/domain/ActionUseCase.test.ts
import { describe, test, expect, vi } from 'vitest';
import { ActionUseCase } from './ActionUseCase';

describe('ActionUseCase', () => {
  test('正常にアクションが実行される', async () => {
    const mockRepository = {
      save: vi.fn(),
    };

    const useCase = new ActionUseCase(mockRepository);
    const result = await useCase.execute({ name: 'test' });

    expect(mockRepository.save).toHaveBeenCalled();
    expect(result.id).toBeDefined();
  });
});
```

## 禁止事項

### 絶対にやってはいけないこと

1. **層を跨いだ直接依存**
   ```typescript
   // ❌ 絶対NG: UseCaseがInfrastructureに直接依存
   import { EntityRepositoryImpl } from '../../infrastructure/repositories/EntityRepositoryImpl';
   ```

2. **ドメイン層での外部依存**
   ```typescript
   // ❌ 絶対NG: ドメインエンティティでHTTPライブラリを使用
   import axios from 'axios';
   ```

3. **ビジネスロジックの漏出**
   ```typescript
   // ❌ 絶対NG: コントローラーでビジネスルール判定
   if (requestBody.amount > 1000000) {
     return c.json({ error: 'Amount too large' }, 400);
   }
   ```

4. **型安全性の放棄**
   ```typescript
   // ❌ 絶対NG: any型の使用
   function processData(data: any): any {
     return data;
   }
   ```

## 品質チェックリスト

新しいHonoアプリを作成する際は以下を必ず確認する:

### 必須チェック項目
- [ ] ディレクトリ構成がルールに準拠している
- [ ] 依存関係の方向が正しい（依存関係逆転の原則）
- [ ] 各層にindex.tsのPublic APIが定義されている
- [ ] ドメインエンティティにビジネスルールが実装されている
- [ ] ドメインエラークラスが適切に定義されている
- [ ] 各層のテストが実装されている
- [ ] TypeScriptの型チェックが通る
- [ ] Lintエラーがない
- [ ] OpenAPI仕様が正しく定義されている

### 品質チェックコマンド
```bash
npm run typecheck  # 型チェック
npm run lint       # Lint
npm test           # テスト実行
npm run test:coverage  # カバレッジ確認
```

## 推奨パッケージ構成

### package.json テンプレート
```json
{
  "scripts": {
    "dev": "wrangler dev --ip 0.0.0.0",
    "deploy": "wrangler deploy --minify",
    "format": "npx @biomejs/biome format --write .",
    "lint": "npx @biomejs/biome lint .",
    "test": "vitest",
    "test:coverage": "vitest run --coverage",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@hono/swagger-ui": "^0.5.0",
    "@hono/zod-openapi": "^0.18.3",
    "hono": "^4.6.13",
    "uuid": "^11.0.3",
    "zod": "^3.24.1"
  },
  "devDependencies": {
    "@biomejs/biome": "1.9.4",
    "@types/node": "^24.0.3",
    "@vitest/coverage-v8": "^3.2.2",
    "typescript": "^5.8.3",
    "vitest": "^3.1.3"
  }
}
```

## 最終的な成果物の確認

新しいアプリが完成したら以下を確認する:

1. **アーキテクチャ品質**: 依存関係が正しく、各層の責務が明確
2. **テスト品質**: カバレッジ80%以上、主要ビジネスロジックがテスト済み
3. **型安全性**: TypeScriptエラーなし、any型不使用
4. **ドキュメント**: OpenAPI仕様が完全、Swagger UIで確認可能
5. **保守性**: コードが読みやすく、変更に強い設計

このルールに従うことで、保守性が高く、テストしやすく、変更に強いHonoアプリケーションを構築できます。