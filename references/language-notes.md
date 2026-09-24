# 言語ごとの当てはめ方

原則の目的(理解と変更を楽にする)は共通。実現方法は言語のイディオムに合わせる。
言語の慣習と原則が衝突したら、言語の慣習とプロジェクトの規約を優先する。

## 目次
- TypeScript / JavaScript
- React / Next.js
- Go
- Python
- Java
- レビュー時の言語別 XXX 候補

---

## TypeScript / JavaScript

- `const` をデフォルトにし、`let` は必要な場合だけ。`var` は使わない
- 不変: `readonly` プロパティ、`ReadonlyArray<T>` / `readonly T[]`、`Readonly<T>`、`as const`
- 文字列への執着を避ける: 文字列リテラルの union 型(`type Status = "draft" | "published"`)
- 種別による分岐は、判別可能な union と `switch` の網羅性チェック(`never` による漏れ検出)が有効。strategy と使い分ける
- ID の取り違え防止: branded type
- `any` を避ける。境界(API レスポンスなど)では `unknown` で受けて、ユーザー定義型ガード(`function isUser(v: unknown): v is User`)で絞る。TypeScript の価値の大半は型定義なので、返り値の型も省略しない
- 型アサーション `as` の乱用を避ける。`as` は検証せず型を決めつけるので実行時に落ちる。まず型定義とデータ構造を見直し、避けられなければ型ガードで絞る(`as const` は readonly 化なので別。これは可)
- null / undefined は型で表す。プロパティに `undefined` を直接代入せず、optional(`age?: number`)で表現する。値がないときのデフォルトは `??` が読みやすい
- エラー: `Error` のサブクラスを throw する。`catch {}` で握り潰さない。Promise の reject を放置しない
- optional chaining(`?.`)の多用は null が設計に浸透しているサイン。境界で null を排除できないか検討する
- 命名: 変数・関数は camelCase、型・クラスは PascalCase、定数はプロジェクトの慣習(UPPER_SNAKE_CASE が多い)。配列は複数形(`users`)にし、`~List` は避ける(`List` は言語仕様上の意味を持ちうるため誤解を生む)

## React / Next.js

『良いコード/悪いコード』のクラス設計の話は、React では「コンポーネント設計」「カスタムフック設計」に置き換えて読む。AI生成のReactコードで特に頻出するのは、下記の「useEffectで導出」「不要なstate化」「過剰メモ化」の3つ。レビュー時はまずここを見る。

### コンポーネント設計
- **1コンポーネント1関心事**: レイアウト、データ取得、状態管理、表示ロジックが1つに全部乗っていたら分割サイン(神クラス相当)。目安として、JSX が長くて画面全体を一目で追えない(300行前後が分割検討ライン)、useState/useEffect が多数並ぶ、はどちらも分割シグナル
- **UI とロジックを分ける**: 「見た目と操作の受け渡し」と「データ取得・状態管理・機能ロジック」を別の層にする。ロジックはカスタムフックへ寄せ、コンポーネントは表示に専念させる(コンテナ/プレゼンテーション、あるいは hooks 分離)
- **フラグ props の増殖を避ける**: `<Button variant="primary" />` のように種類で分けるほうが、`isDanger`/`isSmall`/`isOutlined` の組み合わせ爆発より読みやすい。呼び出し側が内部の見た目分岐を知らなくて済む〔尋ねるな命じろ〕
- **props は分割代入で受け、型を付ける**: `({ label, onClick }: ButtonProps)`。`props.` の繰り返しを避け、必要な props が型から一目で分かる
- **children とコンポジション**を継承の代わりに使う(React に継承の概念は薄い)
- **prop drilling(深いバケツリレー)は低凝集・密結合の兆候**。特に「自分では props を使わず下へ渡すだけの中間コンポーネント」は見直し対象。上位と統合して階層を浅くするか、Context / コンポーネント合成(children を上位から渡す)で解消する
- **`export default` を避け、named export を使う**(Next.js が default を要求する page/layout 等は除く)。default だと呼び出し先の名前変更が import に現れず、意図から乖離した変更が入り込みやすい

### カスタムフック
- 「無関係な下位問題の抽出」がそのまま当てはまる。データ取得・購読・タイマーなどのロジックは `useXxx` に切り出し、コンポーネントは UI に専念させる
- **1フック1責務**。`useUser` に fetch・form・validation・pagination を詰め込まない。`useUserFetch` / `useUserForm` のように割る。詰め込むとテストも再利用もできない
- 名前は目的ベースで(`useUserData` より `useCurrentUserProfile`)
- テスト可能にするため、依存(fetcher など)は引数で受け取り差し替え可能にする

### 状態管理(state を増やさない)
- **計算で求まる値は state にしない**。`useState` + `useEffect` で同期するのではなく、レンダー中に導出するか `useMemo` を使う。二重管理は必ずズレる
  ```tsx
  // Before: items が変わるたびに effect で filtered を追従させる(ズレとバグの温床)
  const [filtered, setFiltered] = useState<Item[]>([]);
  useEffect(() => { setFiltered(items.filter((i) => i.active)); }, [items]);

  // After: 導出でよい
  const filtered = useMemo(() => items.filter((i) => i.active), [items]);
  ```
- **派生する真偽値を別 state にしない**: `isZero` を state にして effect で同期せず、`const isZero = count === 0;`
- **サーバー取得済み・props 由来の静的データを state に写さない**: `const items = props.items;` でよい
- サーバー由来データ(fetch結果)と UI 固有の一時状態(モーダル開閉など)を混ぜない。前者はキャッシュ機構(TanStack Query, SWR, Next.js の fetch キャッシュ)に、後者はローカル state に
- グローバル状態は「本当にグローバルに必要か」を疑う(YAGNI)。多くは近い親の state か Context で足りる。必要なら Jotai / Zustand などの軽量ライブラリを選ぶ

### useEffect / メモ化
- **useEffect を反射的に使わない**(React 公式「You Might Not Need an Effect」)。「レンダー中に計算できる」「イベントハンドラ内でやれる」ものを effect に押し込むと、不要な再レンダー・副作用のループ・依存ズレを生む。使うのは外部システムとの同期(購読、DOM 直接操作、非React連携)など、本当に必要な場合に絞る
- useEffect を使うなら、依存配列は正直に書く。依存を消すための `// eslint-disable` は握り潰しに近い
- effect 内に処理をベタ書きしない。名前付き関数に切り出して呼ぶ(20〜30行の直書きは範囲が見えなくなる)
- **`useMemo`/`useCallback` を反射的に付けない**。フック自体にコストがあり、軽い計算(単純な map、文字列結合)に付けると逆効果。使うのは、重い計算、memo 済み子への安定した参照渡し、参照の同一性が要る場面。React 19 の React Compiler が入る環境では、手動メモ化の多くは不要になる

### 型(React 文脈)
- **API レスポンスを `any` で受けない**。`unknown` で受けてユーザー定義型ガード(`v is User`)で絞る。`as` での決めつけは実行時に落ちる
- props やモデルの型は、プロジェクトの方針に従い `type` か `interface` に統一する

### Next.js 固有(App Router)
- **Server Component をデフォルトにし、`"use client"` は必要な葉の側に絞る**。クライアント境界が上位に寄ると、下のツリー全体がクライアントに引きずられる(密結合)。App Router / RSC / Server Actions を前提にすると、この分離が効く
- データ取得はできる処理をサーバーで行う(サーバーファースト)。取得と表示を近くに置きつつ、重複取得はキャッシュ(`fetch` のキャッシュ、`cache()`)で抑える
- Server Actions / Route Handlers のエラーはクライアントへ伝わる形で返す。`try/catch` で握り潰して何も返さないのは XXX 相当。入力は zod 等で検証する
- 環境変数はクライアント公開のものを `NEXT_PUBLIC_` で明示し、秘密情報をクライアントに混ぜない。使う場所ごとに `process.env.X` を直書きせず、起動時に検証した型付き設定として扱う〔完全コンストラクタの発想〕
- `next/image` や動的 import(`next/dynamic`)など、フレームワークが用意した最適化手段を、自前実装より先に検討する

### プロジェクト規約に従う領域(スキルが固定しないもの)
次はチーム/プロジェクトごとに正解が違うため、既存の規約・設定があればそれに従い、無ければ一貫性だけ担保する。スキルとして特定の選択を強制しない。
- コンポーネント分類の流儀(Atomic Design を採用しているか、features 単位か)とディレクトリ構成
- スタイリング手法(CSS Modules / Tailwind / CSS-in-JS)
- 純粋なスタイル規約: 省略記法、`??` 優先、テンプレートリテラル、配列は `T[]` 記法、真偽値 props の省略、`~List` より `~s`、`app/` 配下は kebab-case など。ESLint/Prettier が直せるものは指摘しない

## Go

- **エラーは戻り値で返すのが言語の慣習**。「例外をスローする」原則は「エラーを特殊な値で表さず、`error` で明示的に返す」と読み替える
- `err` を `_` で捨てない。`if err != nil { return err }` だけでなく、文脈を `fmt.Errorf("load config: %w", err)` で付ける
- 名前はスコープに比例させる、が Go の慣習とよく一致する。狭いスコープのループ変数やレシーバは1〜2文字でよい。パッケージ外に公開する名前ほど具体的にする
- getter に `Get` を付けない慣習(`Owner()`、setter は `SetOwner()`)
- パッケージ名に `util`, `common`, `helpers` を使わない(Go でもアンチパターン)。何を提供するかで名付ける
- 継承はない。構造体埋め込みの乱用は、継承と同じ密結合を生む
- interface は使う側で、小さく定義する
- 完全コンストラクタ: フィールドを非公開にし、検証付きの `NewXxx()` を用意する
- 不変性の言語サポートは弱い。値レシーバ、コピーを返す、スライスやマップを外に渡すときはコピーする、で守る
- 早期 return は Go の標準スタイル。`else` を避け、正常系を左端に保つ

## Python

- PEP 8 に従う(snake_case、定数は UPPER_SNAKE_CASE)
- 値オブジェクト: `@dataclass(frozen=True)`、`__post_init__` で検証。または `NamedTuple`
- 文字列への執着を避ける: `Enum`、`Literal`
- 型ヒントを付ける。戻り値が無い可能性は `X | None` で明示する
- `except Exception: pass` や裸の `except:` は握り潰し
- ミュータブルなデフォルト引数(`def f(items=[])`)は呼び出し間で状態を共有してしまう
- 内包表記は1段なら読みやすいが、ネストや複雑な条件が入るなら通常のループにする
- モジュールレベルの可変グローバルを避ける

## Java

- `final` をデフォルトにする(フィールド、ローカル変数、引数)
- 値オブジェクト: `record`(Java 16+)。コンパクトコンストラクタで検証する
- `Optional` は戻り値に使う。フィールドや引数には使わない
- コレクションを外に渡すときは `List.copyOf` や `Collections.unmodifiableList`
- 閉じた種別の分岐: `sealed interface` とパターンマッチングの `switch`(Java 21)も選択肢
- `catch (Exception e) {}` や、`e.printStackTrace()` だけの catch は握り潰し
- 引数が多いコンストラクタは、パラメータオブジェクトやビルダーを検討する
- クラス名の `Manager`, `Util`, `Helper` は責務が曖昧なサイン

## Rust

- 所有権・借用が「不変性」「尋ねるな命じろ」の多くを言語レベルで強制する。`&mut` を最小限にし、可能な限り値を消費するのではなく借用で済ませる
- 完全コンストラクタは自然に実現しやすい: 不正値を許さない型を作り、`pub` フィールドを避けてコンストラクタ関数(`new`, `try_new`)経由にする
- エラーは `Result<T, E>` で返す。`unwrap()`/`expect()` はプロトタイプやテスト以外では避ける。`?` 演算子でエラーを伝播する
- null 相当は `Option<T>`。`None` を握り潰さない
- 値オブジェクトは newtype パターン(`struct UserId(String);`)で表現し、取り違えを型で防ぐ
- 所有権の移動(move)とクローンの区別を、命名やコメントではなく型と借用チェッカに語らせる。無闇な `.clone()` は密結合・低凝集ではなく「設計の逃げ」のサインであることが多い
- トレイト境界を使えば、interface による多態(strategy)と同じことができる
- `unsafe` ブロックは、原則の適用よりも安全性の説明(なぜ安全か)を優先する

## C++

- RAII が「不変」「完全コンストラクタ」「リソース解放の握り潰し防止」を一括で担う。生ポインタでの手動 `new`/`delete` は避け、`unique_ptr`/`shared_ptr`、コンテナ、スマートラッパーに任せる
- コンストラクタで不変条件を保証し、デストラクタで後始末する。「生焼けオブジェクト」は、2段階初期化(`init()` を別途呼ぶ設計)で発生しやすい
- 値渡し・const 参照・不変メンバ(`const`)をデフォルトにする。書き換えが必要な理由がある場合だけ非 const にする
- 例外を使うコードベースなら、例外の握り潰し(空の `catch (...)`)を避ける。例外を使わない規約のコードベースなら、戻り値やステータスコードでのエラー伝播を握り潰さない(戻り値の無視、`[[nodiscard]]` の欠如)
- 継承よりコンポジション、多重継承は特に慎重に。ヘッダに実装を詰め込みすぎない(単一責任がファイル単位でも崩れる)
- マジックナンバーは `constexpr` で名前を付ける

## Ruby

- 動的型言語なので、名前による意図の伝達(可読性の軸)の重要度が相対的に高い。`is_`/`has_?` の代わりに `?` サフィックス(`valid?`, `empty?`)、破壊的メソッドには `!` サフィックス
- 不変性は `freeze` で部分的に強制できる。デフォルトで書き換え自由なので、意図的に不変にしたい値は明示する
- 値オブジェクトは `Struct`、`Data.define`(Ruby 3.2+)、または小さな PORO(Plain Old Ruby Object)で表す
- 例外の握り潰し: 裸の `rescue => e` で握り潰さない。`rescue Exception` は基本使わない(シグナルなども捕まえてしまう)
- `respond_to?`/`method_missing` の多用はメタプログラミングの乱用に当たりやすく、静的な読みやすさを犠牲にする。必然性がある場合に限る
- 「尋ねるな命じろ」は Ruby のダックタイピングと相性がよい。`obj.status == :active` のような外部からの判定より、`obj.active?` に寄せる

## Kotlin

- null 安全が言語機能にある。`?.`/`?:` に頼りきらず、境界(API レスポンスや外部入力)で早めに non-null な型へ変換する。`!!` は原則避ける
- 値オブジェクトは `data class` で簡潔に書ける。`init {}` ブロックで完全コンストラクタのガードを行う
- 不変はデフォルト志向: `val` を基本にし、`var` は理由があるときだけ。コレクションも `List`(読み取り専用)を返し、`MutableList` を外に渡さない
- 種別による分岐は `sealed class`/`sealed interface` + `when` の網羅性チェックで、switch の重複を防げる
- 拡張関数の乱用は、責務の置き場所を曖昧にすることがある。クラスの本質的な振る舞いは拡張関数ではなくクラス自身に持たせる

## C#

- `record` で値オブジェクトを簡潔に書ける(C# 9+)。コンストラクタでの検証は通常のクラスと同様に行う
- 不変性: `readonly` フィールド、`init` アクセサ、`ImmutableList<T>` などの Immutable コレクション
- null 許容参照型(nullable reference types)を有効にし、`?` の有無で null 許容を型で表明する
- LINQ の多用は式の可読性を上げることもあれば、1行に詰め込みすぎて下げることもある。説明変数(中間の `IEnumerable` に名前を付ける)で調整する
- 例外の握り潰し: 空の `catch { }` や `catch (Exception) { }` だけで再スローもログもない実装を避ける
- プロパティの getter に重い処理を書かない(呼び出し側は「軽い」と期待する)。重いなら明示的にメソッドにする

---

## レビュー時の言語別 XXX 候補

| 言語 | パターン |
|---|---|
| TS/JS | `catch {}`、放置された Promise(await も catch もない)、`any` 経由の未検証データ、`==` による意図しない型変換 |
| React/Next.js | `useEffect` での状態導出(レンダー中に計算できるもの)、計算で求まる値やサーバーデータの不要な state 化、`useEffect` の依存配列の省略(eslint-disable乗せ)、Server Action/Route Handler での握り潰し、API レスポンスの `any`/`as` 決めつけ、キーに index を使った可変リストの `.map()` |
| Go | `_ = err` や err の無視、goroutine 内の panic 放置、ロックなしでの map の並行書き込み、ループ変数のクロージャ捕捉(Go 1.22 未満) |
| Python | 裸の `except:`、ミュータブルなデフォルト引数、`eval`/`exec` への外部入力 |
| Java | 空の catch、`equals` をオーバーライドして `hashCode` をしない、`Optional.get()` の無条件呼び出し |
| Rust | 多用された `unwrap()`/`expect()`、`unsafe` の説明なしの使用、意味なく多用される `.clone()` |
| C++ | 生ポインタの手動 `delete` 漏れ、`catch (...)` での握り潰し、初期化されないメンバ変数、ダングリングポインタ/参照 |
| Ruby | 裸の `rescue => e` での握り潰し、`rescue Exception`、`method_missing` の乱用 |
| Kotlin | `!!` の多用、`lateinit var` の未初期化アクセス、`runCatching` で例外を握り潰したまま |
| C# | 空の `catch { }`、`async void`(イベントハンドラ以外)、`.Result`/`.Wait()` によるデッドロックの温床 |
