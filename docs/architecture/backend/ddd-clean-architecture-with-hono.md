# **HonoにおけるClean Architecture/DDD実装ガイドライン**

## このドキュメントについて

このドキュメントは，Honoをフレームワークとして利用しているバックエンドアプリケーションにおいて，Clean Architectureおよびドメイン駆動設計（DDD）に従い設計・ディレクトリ構成するためのガイドラインである．
本アプリケーションのバックエンド開発では，**必ずこのガイドラインに従い設計・実装を行うこと．**

## アーキテクチャの全体像

アプリケーションはClean Architectureの原則に基づき、以下の4つのレイヤから構成される：

- **Interface Layer**：外部とのインターフェース
  - Routes：エンドポイント定義とバリデーション
  - Controllers：HTTPリクエスト/レスポンス処理
- **UseCase Layer**：アプリケーション固有のビジネスルール
  - UseCases：ユースケースの実現とエンティティの協調
- **Domain Layer**：コアビジネスルール
  - Entities：識別子を持つビジネスオブジェクト
  - Value Objects：値を表現するオブジェクト
  - Repository Interfaces：データ永続化の抽象化
- **Infrastructure Layer**：外部システムとの具体的な接続実装
  - Repository Implementations：リポジトリインターフェースの実装
  - Database：DB接続とスキーマ定義


### レイヤー間の依存関係

各レイヤはClean Architectureの思想に基づき，以下のような依存関係を持つ．
**内側の層は外側の層を知ってはいけない（依存関係逆転の原則）**

```
Interface → UseCase → Domain ← Infrastructure
```

この依存関係を実現するため，middleware機能を活用しindex.ts（エントリポイント）でRepositoryImplをusecaseにDIする．

```index.ts
// 依存注入に必要なクラスをインポート
import { SpotRepositoryImpl } from './infrastructure/repositories/spots/SpotRepositoryImpl';
import { CreateSpotUseCase } from './usecase/spots/CreateSpotUseCase';
import { SpotController } from './interface/controllers/SpotController';

// 依存注入ミドルウェア
app.use('/api/spots/*', async (c, next) => {
  // リポジトリとユースケースのインスタンス化
  const spotRepository = new SpotRepositoryImpl(c);
  const createSpotUseCase = new CreateSpotUseCase(spotRepository);

  // コントローラーのインスタンス化
  const spotController = new SpotController(
    createSpotUseCase,
  );

  // コンテキストにコントローラーを設定
  c.set('spotController', spotController);

  await next();
});
```

### ディレクトリ構成

例として，「スポット管理」「荷物管理」というBoundedContextが存在するアプリのディレクトリ構成図を示す．

```
src/
 ├── index.ts                              # エントリーポイント・DI設定
 ├── features/                             # BoundedContext別のディレクトリを管理
 │    ├── spots/                             # スポット管理 BoundedContext
 │    │   ├── domain/                          # Domain Layer
 │    │   │   ├── entity/                        # エンティティ・集約・値オブジェクト
 │    │   │   ├── repository/                    # リポジトリインターフェース
 │    │   │   └── service/                       # ドメインサービス
 │    │   ├── usecase/                         # UseCase Layer
 │    │   ├── interface/                       # Interface Layer
 │    │   │   ├── route/                         # ルート定義
 │    │   │   └── controller/                    # コントローラ
 │    │   └── infrastructure/                  # Infrastructure Layer
 │    │        └── repository/                   # RepositoryImpl
 │    └── luggage/                           # 荷物管理 BoundedContext
 │        ├── …                                # spotsと同様の構成
 │        └── …                                # spotsと同様の構成
 └── shared/                                 # 共通機能
      ├── database/                            # データベース機能
      ├── auth/                                # 認証関連機能
      └── middleware/                          # Honoミドルウェア定義
```

## デザインパターン

### Domain（ドメイン）

1.  **Entity（エンティティ）:**
    *   明確な識別子を持つ可変な概念を表現しする（例: `Spot`, `LuggageGroup`）．
    *   エンティティに関連する業務ロジックと状態をカプセル化する．
    *   アプリケーションが生成するUUIDv7形式のIDで識別される．
2.  **Value Object（値オブジェクト）:**
    *   概念的な識別子を持たないドメインの記述的側面を表現します（例: `Money`, `Percentage`, `SpotId`, `LuggageId`）。
    *   不変であり、値として扱われます。
    *   その値に固有の関連データとドメインロジックをカプセル化します（例: `Money.add()`, `Percentage.applyTo(Money)`）。
3.  **Aggregate（集約）:**
    *   データ変更の単一単位として扱われるEntityとValue Objectの集まりです。Aggregate Rootは主要なエンティティの参照ポイントです。
    *   単一トランザクション内での境界内データの一貫性を保証します。
4.  **Repository（リポジトリ）:**
    *   ドメイン層ではデータアクセス層のインターフェースのみを定義する．

### Usecase（ユースケース）

ユースケース層は業務ロジックとトランザクション管理を担当します。

1.  **Usecase（ユースケース）:**
    *   各Bounded Contextの `usecase` パッケージに配置（例: `CreateSpotUsecase`）
    *   Domainを呼び出すことで業務ロジックの実装とワークフローの制御

### Controller（コントローラ）

Controllerは外部からの入り口となるREST APIエンドポイントを定義する．

1.  **REST Controllers:**
    *   各Bounded Contextの `controller` パッケージに配置
    *   HTTPリクエスト/レスポンスの処理
    *   Request DTOの受け取りとResponse DTOの返却
    *   Usecaseへの処理委譲
    *   薄いレイヤとして、業務ロジックはUsecaseに委ねる

## エラーハンドリング

- 各レイヤはそれぞれのレイヤで発生するエラーをキャッチし，キャッチしたエラーの内容を伝える例外をスローする．
- スローされた例外はHonoの`app.onError`でキャッチし，適切なHTTPステータスコードでレスポンスする．

### インフラストラクチャ層がスローする例外

- **DatabaseException**
  - データベースへのクエリ実行時にエラーが発生した場合にスローする例外．
  - DatabaseExceptionには**エラーを発生させたクエリの情報を含める**
  - APIのレスポンスとしては500 Internal Server Errorとして扱われることが期待される

### ドメイン層がスローする例外

- **DomainException（抽象基底クラス）**
  - **DomainValidationException:** 値オブジェクトの制約違反（例：緯度が-90〜90度の範囲外）
  - **BusinessRuleException:** ビジネスルール違反（例：削除済みスポットの更新試行）
  - **AggregateException:** 集約内の一貫性制約違反（例：必須フィールドの欠損）
  - **WorkflowException:** ワークフロー制約違反（例：前提条件未満足での処理実行）
  - 全てAPIのレスポンスとしては422 Unprocessable Entityとして扱われることが期待される

### ユースケース層がスローする例外

- **UseCaseException（抽象基底クラス）**
  - **UseCaseValidationException:** 値オブジェクトの制約違反（例：緯度が-90〜90度の範囲外）
    - 400 Bad Requestとして扱われることが期待される
  - **ResourceNotFoundException:** リソースが見つからない（例：存在しないスポットIDでの更新）
    - 404 Not Foundとして扱われることが期待される
  - **ConflictException:** 競合状態（例：同時更新による楽観的ロック違反）
    - 409 Conflictとして扱われることが期待される
  - **AuthorizationException:** 認可エラー（例：他ユーザーのスポット編集試行）
    - 404 Not Foundとして扱われることが期待される（ステータスコードによってリソースの存在を検知されないよう，403 Forbiddenとはしない）

### コントローラ層（またはHonoにおいてはミドルウェアやZod）がスローする例外

- **ZodError:** Zodにより精査エラーとなった場合
  - 400 Bad Requestとして扱われることが期待される
- **InterfaceValidationException:** Controller層での精査でエラーとなった場合
  - 400 Bad Requestとして扱われることが期待される
- **AuthException:** Authorizationヘッダが未設定，またはJWTデコードに失敗した場合
  - 401 Unauthorizedとして扱われることが期待される

## テスト戦略

- **階層化テスト戦略:**
  - ドメイン単体テスト: Domain Entity、Value Object、Domain Serviceに焦点を当てます。純粋なJava、依存関係なし、非常に高速です。コアビジネスルールと計算を検証します。
  - Usecaseテスト: Usecaseに焦点を当てます。Repositoryインターフェースと外部サービスにモックを使用して、ビジネスワークフローのオーケストレーションをテストします。ユースケース実行フローとエラーハンドリングを検証します。
  - 結合テスト: APIと永続化レイヤに焦点を当てます。概念と外部技術（DB、API）間のマッピングをテストします。これらのテストは低速で、実行中のデータベース/APIエンドポイントが必要です。
- **Repositoryモック:** 高速で分離されたUsecaseテストに不可欠です。UsecaseテストでRepositoryインターフェースをモックします。