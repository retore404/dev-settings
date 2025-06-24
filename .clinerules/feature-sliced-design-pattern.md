# このファイルの内容

- フロントエンド開発をするにあたり従うべき設計アプローチ・ディレクトリ構成である"Feature Sliced Design(FSD)"に関するルールを定めたものである．

# FSDの基本概念

## Layers（レイヤー）

アプリ全体を役割で6つのレイヤーに分割する．上位レイヤから順に以下のレイヤーが存在する．

| Layer      | 役割概要                                                                    |
| ---------- | -------------------------------------------------------------------------- |
| `app`      | エントリポイント，スタイル，プロバイダーなど，アプリを実行するためのすべての要素． |
| `pages`    | 特定のページでのみ使用されるロジックとUIコンポーネント．                        |
| `widgets`  | 再利用可能な大規模UI部品                                                     |
| `features` | ユーザーにビジネス価値をもたらすアクション，ユースケース．                      |
| `entities` | プロジェクトが取り扱うビジネスに関するエンティティ．"user"や"product"など．     |
| `shared`   | 特定のビジネスに紐づかない，再利用可能な共通UI, utils, 型定義、設定．           |


## Slices（スライス）

- 各レイヤー内をビジネスドメイン単位で分割する（例：features/user、entities/account）．
  - スライスごとに責任とコードを閉じ込め、疎結合・再利用性を高める．
  - ただし，機能管理の都合上スライスとサブ・スライスの2階層のスライスを配置する．
    これは，FSDの基本にはない本プロジェクト固有のルールである．
  - この固有ルールの策定背景は以下の通り．
    - サブ・スライスを定義しない場合のfeaturesレイヤー配下
      - `features/registerUser`, `features/updateUser`, `features/topUp`, `features/payment`のような構成となる．
      - 例えば`features/registerUser`, `features/updateUser`は別の機能ではあるものの，"user"に関する機能群の1機能とも言える．
      - このようなディレクトリ構成となると，機能が増えた時にどれが"user"に関する機能なのか把握しづらい．
      - そこで，以下のようにサブ・スライスを定義する固有ルールを定義する．
    - サブ・スライスを定義する場合のfeaturesレイヤー配下
      - `features/user/register/`, `features/user/update`, `features/wallet/topUp`, `features/wallet/payment`のような構成となる．
      - このような構成とすることで，例えばuserに関する機能は`features/user`配下に集約されていることが一目で把握できる．
- `app`レイヤーおよび`shared`レイヤは基本的にスライスを持たず，後述のセグメントを直接持つ．

## Segments（セグメント）

スライス内で責務別に整理されたサブ構造．推奨される命名は以下の通り．

| Segment名 | 内容例                                                             |
| -------- | ------------------------------------------------------------------- |
| `ui`     | UIに関わる全て（UIコンポーネント，日付のフォーまた，スタイル定義など）    |
| `model`  | データモデル（スキーマ，インターフェース，ビジネスロジックなど）          |
| `api`    | バックエンドとのやり取り（リクエスト実行用の関数，データ型，マッパーなど） |
| `lib`    | 同じスライスの他のモジュールが使用するライブラリ                        |
| `config` | 設定ファイル、定数など                                               |
| `hooks`  | React hooks                                                        |

# 依存関係のルール

上位レイヤーからのみ下位レイヤーに依存可．逆依存は禁止．

| Layer      | 依存可能なレイヤー                                     |
| ---------- | ---------------------------------------------------- |
| `app`      | `pages`, `widgets`, `features`, `entities`, `shared` |
| `pages`    | `widgets`, `features`, `entities`, `shared`          |
| `widgets`  | `features`, `entities`, `shared`                     |
| `features` | `entities`, `shared`                                 |
| `entities` | `shared`                                             |
| `shared`   | （他レイヤーに依存禁止）                               |


# Public APIルール（公開エントリーポイント）

- 各スライスは index.ts によって公開するものを明示する．
- 外部からの参照はすべて index.ts 経由で行う．
- 内部構造は外部に漏らさず、スライスの独立性を保つ．

```
// NG: 内部ファイルを直接参照（依存方向の漏れが起きやすい）
import { useUserApi } from 'entities/User/model/useUserApi';

// OK: Public API からの参照
import { useUserData } from 'entities/User';
```

# 相対パス / 絶対パスの使い分け

- 同一スライス内: ../ などの相対パスを使用．
- 他スライスを参照: entities/User などの絶対パス（alias）で公開APIのみを参照．

# 避けるべきアンチパターン

- shared が features や entities に依存している．
- features 同士が直接 import している（代わりに mediator pattern や event bus を使う）．
- Public API を経由せず、スライス内部の深いファイルに依存している．

# アプリケーションフレームワーク特有のフォルダ構成との関連

## React Router v7

- root.tsx / routes.tsxなど，コモンセンスとして`/`に存在が期待されるファイルについては，FSDの思想ではappレイヤーに配置するのが適切だが，`/`に配置することとする．
- 特定のページからのみ参照されるコンポーネント類は`pages`レイヤーに配置するが，コモンセンスとして`/routes`に配置されるルートは，`pages`レイヤーではなく，`/routes`に配置することとする．

# 典型的なディレクトリ構成の例

上記のルールに従うと，以下のようなディレクトリ構成となることが想定される．

```
src/
├── app/
│   └── index.ts
├── pages/
│   └── user/
├── widgets/
│   └── header/
├── features/
│   ├── user/
│   │   ├── register/
│   │   │     ├── ui/
│   │   │     ├── model/
│   │   │     └── index.ts
│   │   ├── delete/
│   │   │     ├── ui/
│   │   │     ├── model/
│   │   │     └── index.ts
│   │   └── index.ts
│   └── wallet/
│       ├── payment/
│       │     ├── ui/
│       │     ├── model/
│       │     └── index.ts
│       ├── topUp/
│       │     ├── ui/
│       │     ├── model/
│       │     └── index.ts
│       └── index.ts
├── entities/
│   ├── user/
│   │   ├── model/
│   │   ├── ui/
│   │   └── index.ts
│   └── wallet/
│       ├── model/
│       ├── ui/
│       └── index.ts
└── shared/
    ├── ui/
    ├── lib/
    └── index.ts
```