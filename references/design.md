# 設計の軸: 詳細と例

『良いコード/悪いコードで学ぶ設計入門』の考え方を、Write/Read の判断に使える形でまとめたもの。
例は TypeScript で書いているが、原則は言語に依存しない。言語ごとの当てはめ方は `language-notes.md` を参照。

## 目次
1. 悪い構造の兆候
2. クラス設計: 完全コンストラクタと値オブジェクト
3. 不変性
4. 低凝集を防ぐ
5. 条件分岐を整理する
6. コレクション
7. 密結合を防ぐ
8. 設計を蝕むもの
9. 名前設計
10. メソッド設計
11. リファクタリングの進め方
12. 判断の目安

---

## 1. 悪い構造の兆候

- **低凝集**: 関連するデータとロジックが、あちこちに散らばっている。例: 金額の計算ロジックが画面やサービスごとに重複している
- **生焼けオブジェクト**: 生成した直後は使えず、setter や初期化メソッドを呼ばないと正しく動かない。未初期化の状態が存在しうる
- **悪い構造は悪い構造を呼ぶ**: 散らばったロジックは、次の変更でさらに散らばる

読むときは「この値を変更したいとき、何箇所を直す必要があるか」を考えると兆候が見える。

---

## 2. クラス設計: 完全コンストラクタと値オブジェクト

### 完全コンストラクタ
インスタンス変数をすべて初期化できる引数を取り、不正な値はガード節で弾く。
生成できた時点で正しい状態が保証される。

```ts
// Before: 生焼け。どこからでも不正な値を入れられる
class Money {
  amount = 0;
  currency = "";
}

// After
type Currency = "JPY" | "USD";

class Money {
  private constructor(readonly amount: number, readonly currency: Currency) {}

  static of(amount: number, currency: Currency): Money {
    if (!Number.isInteger(amount) || amount < 0) {
      throw new RangeError(`金額は0以上の整数: ${amount}`);
    }
    return new Money(amount, currency);
  }

  add(other: Money): Money {
    if (other.currency !== this.currency) {
      throw new Error(`通貨が異なる: ${this.currency} と ${other.currency}`);
    }
    return Money.of(this.amount + other.amount, this.currency);
  }
}
```

### 値オブジェクト
意味のある値をクラスや専用の型で表す。
- 値に関するロジック(検証、計算、比較)をその型に集められる(高凝集)
- 取り違えを型で防げる

```ts
// Before: 両方 string なので、引数の順序を間違えてもコンパイルが通る
function assignOrder(orderId: string, userId: string) { ... }
assignOrder(user.id, order.id); // バグ

// After(branded type の例)
type UserId = string & { readonly __brand: "UserId" };
type OrderId = string & { readonly __brand: "OrderId" };
function assignOrder(orderId: OrderId, userId: UserId) { ... }
```

### 計算ロジックはデータを持つ側に
データを持つクラスと、それを計算するクラスが分かれていると、計算が他の場所にも書かれ始める。

### 使い所
値オブジェクトは、ドメインの中心にあり、ルール(範囲、形式、単位)を持つ値に使う。
すべての数値や文字列を包むのはやりすぎ。

---

## 3. 不変性

- 再代入しない。用途が違う値には、別の変数を用意する
- 引数を書き換えない
- 状態を変えたいときは、新しいインスタンスを返す

```ts
// Before: 同じ変数に意味の違う値が入っていく
let price = item.basePrice;
price = price * (1 - discountRate);
price = price + shippingFee;
price = Math.floor(price * (1 + TAX_RATE));

// After
const discountedPrice = item.basePrice * (1 - discountRate);
const priceWithShipping = discountedPrice + shippingFee;
const totalWithTax = Math.floor(priceWithShipping * (1 + TAX_RATE));
```

可変が妥当な場面もある: 性能が重要なホットループ、完全にローカルに閉じた組み立て処理など。
その場合も、可変な範囲を小さく閉じ込める。

---

## 4. 低凝集を防ぐ

- **static メソッドの用途を絞る**: ログ出力やフォーマット変換など、インスタンスの状態に関係ないものに限る。ドメインロジックを static に置くと、データとロジックが離れる
- **private コンストラクタ + ファクトリメソッド**: 生成の用途を名前で限定する(`GiftPoint.forStandardMember()`, `GiftPoint.forPremiumMember()`)
- **汎用名のクラスを作らない**: `Common`, `Util`, `Helper`, `Manager` には無関係なロジックが吸い寄せられる。具体的な名前で分ける
- **結果を返す引数(出力引数)を使わない**: 引数を書き換えて結果を返すと、どこで何が変わったか追えない
- **多すぎる引数を避ける**: 目安として4つ以上なら、値オブジェクトやパラメータオブジェクトにまとめることを検討する

### 尋ねるな、命じろ
他のオブジェクトの状態を取り出して、呼び出し側で判断・変更しない。
判断はそのオブジェクト自身に任せ、呼び出し側は命じるだけにする。

```ts
// Before: 残高のルールが呼び出し側に漏れている
if (account.balance >= amount) {
  account.balance -= amount;
} else {
  throw new InsufficientBalanceError();
}

// After
account.withdraw(amount); // 残高チェックは Account 自身が行う
```

---

## 5. 条件分岐を整理する

### 早期 return
ネストを浅くする第一手。`readable-code.md` の「制御フロー」を参照。

### 同じ分岐が複数箇所にある: interface による多態(strategy)
```ts
// Before: 会員種別の switch が、割引・送料・ポイントの各所に重複
function discountRate(plan: string) {
  switch (plan) { case "free": return 0; case "premium": return 0.1; ... }
}
function shippingFee(plan: string) {
  switch (plan) { case "free": return 500; case "premium": return 0; ... }
}

// After: 種別ごとの振る舞いを1箇所に集める
interface MemberPlan {
  discountRate(): number;
  shippingFee(): Money;
}
class FreePlan implements MemberPlan { ... }
class PremiumPlan implements MemberPlan { ... }

const PLANS: Record<PlanType, MemberPlan> = {
  free: new FreePlan(),
  premium: new PremiumPlan(),
};
```
新しい種別を追加するとき、switch の修正漏れが起きない。

### 複雑なルールの組み合わせ: ポリシーパターン
```ts
interface Rule {
  ok(customer: Customer): boolean;
}

class Policy {
  private readonly rules: Rule[] = [];
  add(rule: Rule): void { this.rules.push(rule); }
  complyWithAll(customer: Customer): boolean {
    return this.rules.every((rule) => rule.ok(customer));
  }
}

// 優良顧客の判定 = 購入金額ルール + 返品率ルール + ...
```

### やりすぎない
分岐が1箇所だけで、ケースが2〜3個なら、if/switch のままで十分。
閉じた種別の集合なら、判別可能な union 型と網羅性チェック(TypeScript)や sealed interface(Java)も選択肢になる。

---

## 6. コレクション

- 自前のループより標準 API(`some`, `every`, `find`, `filter`, `map`)を使う。意図が名前で伝わる
- ループ内の深いネストは、早期 `continue` か関数抽出で解消する
- **ファーストクラスコレクション**: コレクションと、それに関するルールをクラスに閉じ込める

```ts
class Party {
  static readonly MAX_MEMBERS = 4;
  private readonly members: readonly Member[];

  private constructor(members: readonly Member[]) {
    this.members = members;
  }

  static empty(): Party {
    return new Party([]);
  }

  add(member: Member): Party {
    if (this.members.length >= Party.MAX_MEMBERS) {
      throw new Error("パーティは最大4人");
    }
    return new Party([...this.members, member]);
  }

  list(): readonly Member[] {
    return this.members; // 外には読み取り専用で渡す
  }
}
```

---

## 7. 密結合を防ぐ

### 継承よりコンポジション
継承すると、サブクラスはスーパークラスの実装の都合に縛られる。
スーパークラスの変更が、全サブクラスに予期せず波及する。
共通の処理は、別クラスのインスタンスを内部に持って使う(コンポジション)。

### 単一責任 = 単一目的
似た処理でも、目的が違うなら共通化しない。

```ts
// Before: 通常割引とセール割引が同じ関数を共用
// → セール割引の仕様変更で、通常割引まで壊れる
function discountedPrice(price: number) { ... }

// After: 目的ごとに分ける
class RegularDiscountedPrice { ... }
class SaleDiscountedPrice { ... }
```
DRY は「同じ知識を2箇所に書かない」原則で、「見た目が同じコードを必ず共通化する」原則ではない。

### その他
- **なんでも public**: 外部から触れる範囲が広いほど、依存が増えて変更しにくくなる。公開は最小限にする
- **神クラス**: あらゆる責務が詰め込まれたクラス。目的ごとに分割する
- **トランザクションスクリプト**: 手続きを延々と1メソッドに書き連ねた構造。データとロジックをドメインのクラスへ移す
- **高凝集を狙って密結合に陥る**: 関係ありそうなものを1箇所に寄せすぎると、無関係な変更が波及する。目標は「疎結合かつ高凝集」

---

## 8. 設計を蝕むもの

| 問題 | 対処 |
|---|---|
| デッドコード(到達不能・未使用) | 削除する。履歴はバージョン管理に残る |
| YAGNI 違反(使われない先回り実装) | 今必要なものだけ作る |
| マジックナンバー | 名前付き定数、または値オブジェクトにする |
| 文字列型への執着(区切り文字で複数の値を1つの文字列に詰めるなど) | 専用の型や構造体にする |
| グローバル変数 | スコープを縮める。共有が必要なら責務を持つクラスに閉じ込める |
| null | 返さない・渡さない。Optional、空コレクション、専用の型で表す |
| 例外の握り潰し | ログだけで済ませず、上位に伝える・回復処理をする・明示的に諦める理由を書く |
| メタプログラミング(リフレクションなど)の乱用 | 型安全と静的解析が効かなくなる。必然性がある場合だけ使う |
| 技術レイヤーだけでのパッケージ分割 | ビジネス概念ごとに分けると、関連コードが近くに集まる |
| コピペ | 同じ知識なら共通化する。目的が違うなら共通化しない |
| 銀の弾丸への期待 | どの設計手法にも適用範囲がある。状況で選ぶ |

---

## 9. 名前設計

名前は、あるべき構造を見破る道具になる。

- **具体的で、意味の範囲が狭い名前を選ぶ**: 「商品」より「予約品」「注文品」「在庫品」。範囲が広い名前には無関係なロジックが集まる
- **存在ベースではなく目的ベース**: 「それが何か」ではなく「何のために使うか」で名付ける
- **関心事を分析する**: そのコードが扱っている関心事を列挙し、名前がそれを表しているか確認する
- **声に出して説明する**: 人に説明すると、名前と中身のズレに気づく(ラバーダッキング)
- **仕様書や利用規約の用語を使う**: ビジネス上の正式な言葉は、そのまま精度の高い名前になる
- **別の名前に置き換えられないか考える**: 汎用的な名前しか付けられないなら、責務が曖昧なサイン

---

## 10. メソッド設計

- **自身のインスタンス変数を使う**: 使わないメソッドは、所属するクラスが間違っている可能性がある
- **不変をベースにする**: 引数や自身の状態をむやみに書き換えない
- **尋ねるな、命じろ**: 上の「低凝集を防ぐ」を参照
- **コマンド・クエリ分離**: 状態を変えるメソッドは値を返さない、値を返すメソッドは状態を変えない。呼び出し側が副作用を予測できる
- **引数**
  - 引数は変更しない
  - フラグ引数を使わない。`render(true)` は呼び出し側で意味が読めない。メソッドを分けるか、strategy を渡す
  - null を渡さない
  - 出力引数を使わない
  - 数を少なくする
- **戻り値**
  - 型で意図を表す(`number` より `Money`、`boolean` の組より判別可能な union)
  - エラーを特殊な戻り値(-1、空文字など)で表さない。例外、または言語の慣習に従ったエラー型で返す

---

## 11. リファクタリングの進め方

### 流れ
1. ネストを解消する
2. 意味のあるまとまりでロジックを分ける
3. 条件を読みやすくする(説明変数、要約変数、メソッド化)
4. ベタ書きのロジックを、目的を表すメソッドや型に置き換える

### 守ること
- ユニットテストで振る舞いを固定してから始める。テストがなければ、現状の振る舞いを記録するテストを先に書く
- 機能追加とリファクタリングを同じ変更に混ぜない。レビューも切り戻しも難しくなる
- 小さなステップで進め、ステップごとにテストを通す
- 不要になった仕様は、削除も選択肢に入れる

---

## 12. 判断の目安

- **マジカルナンバー4**: 人が一度に保持できる要素は4±1程度。引数の数、ネストの深さ、1関数内の分岐数の目安になる
- **循環的複雑度**: 分岐が増えるほど理解とテストが難しくなる。一般的には関数あたり10前後を超えたら分割を検討する
- **コアドメインに設計コストを集中する**: 変更頻度が高く、ビジネスの価値の中心にあるコードほど丁寧に設計する。周辺の使い捨てコードは簡素でよい
- **粗悪なコードは速くない**: 「早く終わらせたい」で設計を省くと、その後の変更がすべて遅くなる
