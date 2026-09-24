# 可読性の軸: 詳細と例

『リーダブルコード』の考え方を、Write/Read の判断に使える形でまとめたもの。
例は TypeScript で書いているが、原則は言語に依存しない。

## 目次
1. 名前に情報を詰め込む
2. 誤解されない名前
3. コメント
4. 制御フロー
5. 巨大な式を分割する
6. 変数
7. コードの再編成(下位問題の抽出、一度に1つのこと、言葉で説明してから書く)

---

## 1. 名前に情報を詰め込む

### 動詞を具体的にする
`getUser()` だけでは、キャッシュから取るのか、DB を引くのか、HTTP を叩くのか分からない。
呼び出し側はコストや失敗の仕方を推測できない。

| 曖昧 | 具体的な候補 |
|---|---|
| get | fetch(ネットワーク), load(ストレージ), find(検索、無ければ空), compute(計算) |
| send | deliver, dispatch, publish, notify |
| make | create, build, generate, compose |
| start | launch, open, begin, initialize |
| check | validate, verify, ensure(失敗時に例外), is〜(真偽を返す) |

### 汎用名を避ける
```ts
// Before: tmp が一時保管ではなく、本文を組み立てる変数になっている
let tmp = "";
for (const item of order.items) tmp += `${item.name} x${item.qty}\n`;
sendMail(customer.email, tmp);

// After
let receiptBody = "";
for (const item of order.items) receiptBody += `${item.name} x${item.qty}\n`;
sendMail(customer.email, receiptBody);
```
`tmp` が許されるのは、値の入れ替えのように寿命が数行で「一時保管」そのものが意味である場合だけ。

### 単位と状態を名前に入れる
```ts
// Before: 30 は秒のつもりだが setTimeout はミリ秒を取る → 30ms で再試行してしまう
const retryDelay = 30;
setTimeout(retry, retryDelay);

// After
const RETRY_DELAY_MS = 30_000;
setTimeout(retry, RETRY_DELAY_MS);
```

| 種類 | 例 |
|---|---|
| 時間 | `Ms`, `Sec`, `Min`, `Hours` |
| サイズ | `Bytes`, `Kb`, `Mb` |
| 距離・角度 | `Meters`, `Km`, `Degrees`, `Radians` |
| 状態 | `plaintextPassword`, `hashedPassword`, `unsafeHtml`, `escapedHtml`, `rawInput`, `validatedInput` |

危険を伴う値(未エスケープ、未検証、平文)ほど、状態を名前に入れる価値が高い。

### 名前の長さはスコープに比例させる
```ts
// 数行で完結するなら短くてよい
users.map((u) => u.email);

// モジュール全体から参照される値は具体的に
export const MAX_UPLOAD_SIZE_BYTES = 10 * 1024 * 1024;
```
略語は、チームの誰もが知っているもの(`id`, `url`, `db`)に限る。自分しか分からない省略や連番(`data2`, `flg`)は使わない。

### ネストしたループのインデックス
`i`, `j`, `k` が混ざると取り違えても気づけない。
まずインデックス自体を消せないか考え(for-of、標準API)、残るなら対象が分かる名前にする。
```ts
// Before
for (let i = 0; i < teams.length; i++)
  for (let j = 0; j < teams[i].members.length; j++)
    if (teams[i].members[j].id === users[i].id) { /* i と j の取り違え */ }

// After
for (const team of teams)
  for (const member of team.members)
    if (activeUserIds.has(member.id)) { /* ... */ }
```

---

## 2. 誤解されない名前

「この名前から、他の意味に読み取れないか」を自問する。

### 上限・下限は max / min
```ts
const CART_ITEM_LIMIT = 10;     // 10 を含む? 含まない?
const MAX_CART_ITEMS = 10;      // 10 まで可、と読める
```

### 範囲の端点
| 名前 | 意味 |
|---|---|
| `first` / `last` | 両端を含む [first, last] |
| `begin` / `end` | 半開区間 [begin, end)。多くの言語・ライブラリの慣習 |
| `start` / `stop` | 曖昧。避ける |

### 真偽値
- `is`, `has`, `can`, `should` で始める
- 否定形を避ける: `isNotFound` → `isFound`、`disableCache` → `useCache`
- `read_password` のように、動詞にも真偽値にも読める名前を避ける: `needsPassword`, `isAuthenticated`

### ユーザーの期待に合わせる
- `getX()` や `size()` は「軽い処理」と期待される。重い計算なら `computeX()`, `countX()` にする
- `filter` は「残す」のか「除く」のか曖昧になりやすい。`select`/`exclude` や `keepIf`/`removeIf` を検討する

---

## 3. コメント

コメントの目的は、コードだけでは伝わらない情報を読み手に渡すこと。

### 書かないもの
- コードを読めば分かること
  ```ts
  // ユーザーを保存する
  userRepository.save(user);
  ```
- ひどい名前を補うためのコメント。名前を直す
- 読みにくいロジックの逐語的な解説。ロジックを直す

### 書くもの
- **意図と理由**: なぜこの実装か、なぜ他の案ではないか
  ```ts
  // 外部 API のレート制限が 10 req/s のため、余裕を持たせて 8 に抑える
  const MAX_REQUESTS_PER_SEC = 8;
  ```
- **定数の根拠**
- **落とし穴**: 「この関数は外部 API を叩くので、ループ内で呼ぶと遅い」
- **全体像と要約**: 長い関数やブロックの先頭に、何をするかを一言で
  ```ts
  async function generateMonthlyReport(userId: UserId) {
    // 1. 対象ユーザーのロックを取る
    // 2. 当月の取引を集計する
    // 3. PDF を生成してストレージに置く
    // 4. ロックを解放する
    ...
  }
  ```
- **意図のコメント**: 処理のなぞりではなく目的を書く
  ```ts
  // Bad:  配列を逆順にループする
  // Good: 価格の高い順に表示する
  ```
- **既知の課題**

| タグ | 意味 |
|---|---|
| `TODO:` | あとで手をつける |
| `FIXME:` | 既知の不具合がある |
| `HACK:` | きれいではない解決策 |
| `XXX:` | 危険。大きな問題がある |

### 正確で簡潔に
- 「それ」「これ」など指す先が曖昧な代名詞を避ける
- 関数の挙動は、入出力の具体例を1つ添えると正確に伝わる
  ```ts
  // 例: stripPrefix("abc_def", "abc_") → "def"
  ```
- ロジックを変えたら、同じコミットでコメントも更新する。古いコメントはないコメントより有害

---

## 4. 制御フロー

### 比較の左右
左に「調べている値(変化する)」、右に「比較基準(あまり変化しない)」を置く。
`if (retryCount < MAX_RETRIES)` は自然に読めるが、`if (MAX_RETRIES > retryCount)` は一瞬止まる。

### if/else の順序
- 否定より肯定の条件を先に
- 単純なケースを先に(else を見失わない)
- 関心を引く・重要なケースを先に

これらは衝突することがある。その場合は読み手が一番迷わない順を選ぶ。

### 三項演算子
単純な値の選択なら簡潔になる。条件や分岐先が複雑なら if/else にする。
```ts
const label = isAdmin ? "管理者" : "一般";              // OK
const fee = a && !b ? (c > 3 ? calc(x) : 0) : fallback(y); // 分解する
```

### 早期 return とガード節
```ts
// Before: 条件の真偽を頭に保持しながら読む必要がある
function memberPrice(user: User | null, order: Order): number {
  if (user) {
    if (user.isMember) {
      if (order.total >= 5000) {
        return order.total * 0.9;
      }
    }
  }
  return order.total;
}

// After
const MEMBER_DISCOUNT_MIN_TOTAL = 5000;
const MEMBER_PRICE_RATE = 0.9;

function memberPrice(user: User | null, order: Order): number {
  if (!user?.isMember) return order.total;
  if (order.total < MEMBER_DISCOUNT_MIN_TOTAL) return order.total;
  return order.total * MEMBER_PRICE_RATE;
}
```
関数から複数回 return するのは悪いことではない。

### ループ内のネスト
```ts
for (const order of orders) {
  if (order.isCancelled) continue;
  if (!order.isPaid) continue;
  ship(order);
}
```

### 追いにくい制御構造
イベント、コールバックの連鎖、例外による分岐、スレッド、`do/while`、`goto` は実行順を追いにくい。
使う必然性があるときだけ使い、乱用しない。

---

## 5. 巨大な式を分割する

### 説明変数
式の中身に名前を付ける。
```ts
// Before
if (req.headers.authorization?.split(" ")[1] && verify(req.headers.authorization.split(" ")[1]).role === "admin") { ... }

// After
const bearerToken = req.headers.authorization?.split(" ")[1];
const isAdmin = bearerToken !== undefined && verify(bearerToken).role === "admin";
if (isAdmin) { ... }
```

### 要約変数
短くても、概念をまとめると読み手の負担が減る。
```ts
const userOwnsDocument = request.user.id === document.ownerId;
if (userOwnsDocument) { /* 編集可 */ }
```

### ド・モルガンの法則
`!(a && b)` より `!a || !b` の方が読みやすいことが多い。どちらが自然に読めるかで選ぶ。

### 短絡評価の乱用
`found || (found = compute())` のような「賢い」1行は、読み手を止める。素直な if にする。

### 同じ式の繰り返し
同じ部分式が何度も出てくるなら、変数や関数に括り出す。typo による片方だけの修正漏れも防げる。

---

## 6. 変数

変数が増えるほど、スコープが広いほど、値が変わるほど、読み手が追うものが増える。

### 不要な変数を削除する
次の条件をすべて満たす変数は消す候補。
- 複雑な式を分割していない
- 名前が説明として機能していない
- 1回しか使われていない
```ts
// Before
const now = new Date();
session.lastAccessedAt = now;

// After
session.lastAccessedAt = new Date();
```
説明変数・要約変数との違いは「読み手の理解を助けているか」。

### 制御フロー変数を消す
```ts
// Before
let done = false;
for (const user of users) {
  if (!done && user.id === targetId) {
    notify(user);
    done = true;
  }
}

// After
const target = users.find((user) => user.id === targetId);
if (target) notify(target);
```

### スコープを縮める
- グローバル変数 → モジュール内 → クラスのメンバ → ローカル、と狭い方へ
- 1つのメソッドでしか使わないメンバ変数はローカル変数にする
- 宣言は使う直前に置く

### 一度だけ書き込む
`const`/`final` で宣言された変数は、読み手が「どこかで書き換わるかも」と疑わなくて済む。

---

## 7. コードの再編成

### 無関係の下位問題を抽出する
手順:
1. この関数の高レベルの目標は何かを考える
2. 各行・各ブロックが、その目標に直接効いているか、無関係の下位問題を解いているかを見る
3. 無関係の下位問題を解くコードがまとまった量あれば、別関数に抽出する

```ts
// Before: 請求書を作る関数の中に、日付整形という下位問題が混ざっている
function buildInvoice(order: Order): Invoice {
  const d = order.orderedAt;
  const issuedOn =
    `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}-${String(d.getDate()).padStart(2, "0")}`;
  ...
}

// After
function formatYmd(date: Date): string {
  const month = String(date.getMonth() + 1).padStart(2, "0");
  const day = String(date.getDate()).padStart(2, "0");
  return `${date.getFullYear()}-${month}-${day}`;
}

function buildInvoice(order: Order): Invoice {
  const issuedOn = formatYmd(order.orderedAt);
  ...
}
```
抽出した関数はプロジェクト固有の事情を知らない汎用コードになり、テストも再利用もしやすくなる。

注意: 抽出しすぎると、読み手が関数の間を行き来する羽目になる。数行で自明なものや、再利用も説明効果もないものは無理に抽出しない。

### 一度に1つのことを
「入力をパースし、検証し、保存し、通知する」を1関数でやると、どこで何が起きているか追えない。
1. コードがやっているタスクを列挙する
2. タスクごとに関数、少なくとも別の段落に分ける

```ts
async function handleSignup(raw: unknown) {
  const input = parseSignupInput(raw);
  const user = createUser(input);      // 検証は User の生成時に行う
  await userRepository.save(user);
  await sendWelcomeMail(user);
}
```

### 言葉で説明してから書く
1. コードの動作を、同僚に話すように簡単な言葉で説明する
2. 説明に出てくるキーワードに注目する
3. 説明の構造どおりにコードを書く

```ts
// 説明:「返金できるのは、支払い済みで、発送前か、発送後 7 日以内の注文」
// → キーワード: 支払い済み / 発送前 / 発送後 7 日以内

const REFUND_WINDOW_DAYS = 7;

function canRefund(order: Order, today: Date): boolean {
  if (!order.isPaid) return false;
  if (order.shippedAt === null) return true;
  return daysBetween(order.shippedAt, today) <= REFUND_WINDOW_DAYS;
}
```
説明できないコードは、書いている本人も整理できていないサイン。

### 短いコードを書く
一番読みやすいコードは、存在しないコード。
- 本当に必要な機能か確認する(YAGNI)
- 標準ライブラリや既存の仕組みで済まないか確認する
- 使われていないコードは削除する
