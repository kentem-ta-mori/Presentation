---
marp: true
---

# “シン”我々はなぜEFCoreを使うのか 
～DDDとEFCoreから考える値オブジェクトのすゝめ～

---

## 書籍の紹介

この番組は、ご覧の本の影響で、お送りいたします。

- ドメイン駆動設計をはじめよう

![image.png](image.png)

- 良いコード/悪いコードで学ぶ設計入門

![image.png](image%201.png)

### フロントエンドの方へ

- 関数型ドメインモデリング
    - DDDの出自が現実世界の業務をオブジェクト指向言語で自然に設計に落とし込むというところにあり、そのため今回の発表はTypeScriptなどの関数型言語では利用しづらい部分があります。
    - この点、TSなどにおけるDDD実践はこちらの本が参考になりそうです。今回の発表はC#なので少々わかりづらいかもしれませんが、FEエンジニアの方にも役立つ内容になるかと思います。
    
    ![image.png](image%202.png)
    

## 振り返り

- EFCoreを利用するのは、DB中心からオブジェクト中心への移行が目的だった
- オブジェクト中心にすることで、より複雑なエンティティの関係性を記述しやすくなる
- しかし、さらに複雑な業務ロジックが存在するプロダクトでは、エンティティ同士の関係をオブジェクトとして記述するだけでは不十分である

---

# イントロダクション

- 皆さんは、コードを書いているときと、ドキュメントを整理しているとき、どっちが楽しいですか？
- では、以下のような仕事をしてみたいと思いますか？
    - ほとんどコードをそのまま書き写したようなExcelの設計書を書く、メンテする
    - 影響範囲になりそうなキーワードをGrep検索して書き出し、すべての行に対して変更の要否を書き出してレビュー依頼する。（レビュー依頼される）
    - 変更する予定のソースから、画面の影響箇所までの呼び出しツリーをすべて書き出す。（そして判定条件網羅のE2Eテストを画面から行う。）
- やりたくないですよね？そんな仕事ばっかりさせられたら・・・
    - アザッス、俺、会社辞めます
    
    ![シグマ.jpg](%E3%82%B7%E3%82%B0%E3%83%9E.jpg)
    
- 開発生産性としてもマイナスすぎます
    - 俺は…弱い…！
    
    ![俺は弱い.jpg](%E4%BF%BA%E3%81%AF%E5%BC%B1%E3%81%84.jpg)
    

今日はそんな未来を回避するために、
エンジニアによるエンジニアのための戦略・戦術をお伝えします。

--- 

## なぜ面倒な仕事が生まれたか

- 私の経験上、長期的に事業的に成功しているシステムのコード≒レガシーコードの現場では、
たいてい上記のような面倒な仕事がつきものでした。
- 成功した事業のシステムの特徴
    - 事業が成功すればするほど、事業のルールが次々に変更・追加される
    - 成功した事業は長く続き、開発メンバーの入れ替えが必ず発生し、業務ルールの暗黙知が必ず喪失する
    - はじめはシンプルな機能でも、局所最適解的にスピードとコスト最小を求めた改修（経営判断としては正しい）を行ううちに、大規模で複雑になっていく
- その結果、きわめて合理的な判断において先に述べたようなマネジメント戦略にたどり着きます
    - 成功した事業のシステム「開発」の特徴
        - 複雑化した（そして名無しの）業務ロジックをコードから読み取ろうとすると認知負荷が高い
        →仕様のドキュメント化（Wikiや設計書）へ逃げる
        - 業務ロジックが凝集せず、あちこちに散らばっている
        →変更箇所の漏れが多発し、力業でマネジメントするしかなくなる
        - 業務の本質を理解せず、場当たり的な実装を行ったためにロジックがカオス化する
        →変更の副作用を担保するために、すべての流れをトレースする必要がある

--- 

## どのように面倒な仕事を回避できそうか

### 課題①：複雑化した（そして名無しの）業務ロジックをコードから読み取ろうとすると認知負荷が高い→仕様のドキュメント化（Wikiや設計書）へ逃げる

- ビジネスの知識を（ドキュメントではなく）コードで雄弁に表現する
    - ドキュメントやコード内コメントはメンテナンスコストが高く、劣化コピーになりがち。
    - コードこそが仕様を表現する唯一の真実です。

### BAD

```csharp
var diff = (placedAt - shippedAt).TotalMinutes;

// 120って何？単位は？どのドメインの制約？
if (diff > 120) throw new Exception("NG");
```

### GOOD

```csharp

// ファクトリメソッド内でチェック
var time = TransportTime.From(shippedAt, placedAt); // 120分超ならここで失敗（仕様が型にある）

```

--- 

### 課題②：業務ロジックが凝集せず、あちこちに散らばっている
→変更箇所の漏れが多発し、力業でマネジメントするしかなくなる

- 業務のルールを凝集させ、適切にカプセル化する
    - 一つのルールは一つの個所にだけ実装すればよいようにする
    - それ以外の場所からは触らせないように守る

### BAD

```csharp
// API POST: 120分チェック
if ((placedAt - shippedAt).TotalMinutes > 120) return BadRequest();
・・・

// API PUT: 120分チェック
if ((placedAt - shippedAt).TotalMinutes > 120) return BadRequest();
・・・

// API PATCH: 120分チェック
if ((placedAt - shippedAt).TotalMinutes > 120) return BadRequest();
・・・

// Batch: 120分チェック
status = (placedAt - shippedAt).TotalMinutes <= 120 ? "OK" : "NG";
・・・
```

### GOOD

```csharp
// どのAPIでもBatchでも、TransportTime型を利用する時点でチェックが適用される
var time = TransportTime.From(shippedAt, placedAt);
```

---

### 課題③：業務の本質を理解せず、場当たり的な実装を行ったためにロジックがカオス化する
→変更の副作用を担保するために、すべての流れをトレースする必要がある

- 偶発的複雑性を排除し、本質的複雑さにのみフォーカスする
    - 偶発的複雑性：コードの都合で生まれる複雑性（フレームワークの都合が入り込んで複雑になる、非同期の都合が入り込んで複雑になる、説明が十分でない名前のせいで複雑になる）
        - システムの価値に影響しない複雑性です
    - 本質的複雑性：システム化の対象とする業務ルールがそもそも複雑
        - こちらは仕方のないこと（というかシステムの提供する価値そのもの）です

### Bad

```csharp
public async Task PlaceAsync(Guid id, DateTime placedAt)
{
    var e = await db.Deliveries.FindAsync(id);

    // DB列（int）を直接いじる＝“仕様”がどこにもいない
    e.TransportMinutes = (int)(placedAt - e.ShippedAt).TotalMinutes;

    // あちこちにifが増える（例外条件が増えるたびに追記）
    if (e.TransportMinutes > 120 && e.CustomerCode != "A") throw new Exception("NG");

    await db.SaveChangesAsync();
}
```

### Good

```csharp
public class ConcreteDelivery
{
    public Guid Id { get; private set; }
    public DateTime ShippedAt { get; private set; }
    public TransportTime TransportTime { get; private set; } = default!;

    private ConcreteDelivery() { } // EF用

    public ConcreteDelivery(Guid id, DateTime shippedAt)
    {
        Id = id;
        ShippedAt = shippedAt;
    }

    public void MarkPlaced(DateTime placedAt)
        => TransportTime = TransportTime.From(ShippedAt, placedAt); // 不正なら生成できない
}

// DbContext
modelBuilder.Entity<ConcreteDelivery>()
    .ComplexProperty(x => x.TransportTime, ct =>
        ct.Property(p => p.Minutes).HasColumnName("TransportMinutes"));
```

---

## つまり・・・？

将来の自分のためにちゃんと自己文書化されたコードになるように設計しよう

というモチベーションが湧いてきます（動けばいいわけではない）

　湧いてきましたよね？その体で進めていきます。

---

# 本質的複雑性に向き合う

新規事業を立ち上げるとして、どちらが長期的に儲かりそうか、考えてみてください

- 単純な業務課題を解決する or 複雑な業務課題を解決する
- 良く知られた方法で解決できる or 他社とは違う新しい方法でしか解決できない

- 我々が扱うビジネス（業務領域）は往々にして複雑である
    - シンプルな（複雑ではない）課題は参入障壁が低かったり、そもそも解決するまでも無かったりする
    - 困難を解決するからこそ、ビジネス（お金を払ってやってもらいたい）になる
- さらに、自社で開発する以上は、他に解決できる一般化される方法がないと言える
    - 他社と比較した際の差別化要因

### 業務ロジック

- 例えば、「コンクリートは、工場で練り混ぜられてから120分以内に打設されなければならない」ただし、「気温が25度を超える場合は90分以内に打設しなければならない」、みたいなもの
    - 業務に固有で、複雑になりがち
        - → これをシステムで計算して価値（面倒ごとの解決）として提供する
- 独特で複雑な業務ロジックに、自ら向き合うとき、それ以上の複雑性を持ち込みたくない
    - 例えば、永続化（RDB）の都合
    - 例えば、View（公開するAPI）の都合
- フレームワークに依存しない純粋なオブジェクト（POCO）で、業務ロジックのみを記述したい
    - 例えば、RDBやViewなど存在せず、すべてがメモリ上で実現できると考えてください
    - あなたは受け取った情報に対して、どのようなルールを実現するか＝業務ロジック**だけ**に集中できます
- このような考えこそ、ドメイン駆動設計の戦略の始まりです

---

# ドメイン駆動設計の戦略

- 業務ロジックの複雑さ、競合他社との差別化　の2軸のマトリクスで区分分けする
    - 一般的な業務領域
    - 補完的な業務領域
    - 一般または補完的な業務領域
    - 中核の業務領域


---

### クイズ

- Q1-1：あなたは高級料理店の支配人です。以下の業務をマトリクスの中に振り分けてください
    - 食材の下ごしらえ　→　補完的な業務領域
    - 複雑な給与計算　→　一般的な業務領域
    - 創造的なレシピ開発　→　中核的な業務領域
- Q1-2：以下の業務領域について、あなたはどんなリソースで解決しますか？結び付けてください
    - 食材の下ごしらえ　→　見習いの新人
    - 複雑な給与計算　→　外部の人材（税理士）
    - 創造的なレシピ開発　→　エース料理長
- 独特だがシンプルな作業は、自社の低コスト人材の教育の機会とする
- 複雑な領域に自社の高コストなリソースを振り分けるべき
- 複雑だが一般的な課題は外部に解決策がある
- Q2-1：あなたは新規に作成するECサイトのアーキテクトです。以下の業務をマトリクスの中に振り分けてください
    - 商品マスタの登録　→　補完的な業務領域
    - 顧客の趣向に合わせた商品のレコメンド機能　→　コア業務領域
    - 決済機能　→　一般的な業務領域
- Q2-2：以下の業務領域について、あなたはどんなリソースで解決しますか？
    - 商品マスタの登録機能　→　若手の新人
    - 顧客の趣向に合わせた商品のレコメンド機能　→　エースエンジニア
    - 決済機能　→　外部ライブラリ


---

# アーキテクチャの位置付け

- トランザクションスクリプト・アクティブレコードの振り返り
    - トランザクションスクリプトは、すべてを手続き的に記述する
    - アクティブレコードは、１レコードを１オブジェクトとして表現する
- 複雑性と独自性のマトリクスと、アーキテクチャ(トランザクションスクリプト、アクティブレコード、ドメインモデル)を関連づける
    - アクティブレコード（トランザクションスクリプト）は業務ロジックに外部の複雑さが持ち込まれる
        - レイヤードアーキテクチャになりがちで、主に永続化の都合が持ち込まれる
    - ドメインモデルでは業務をモデリングしてPOCOで業務ロジックを記述する
        - 公開するAPIや、永続化の都合は外部に追いやり、依存性を逆転させる
            - クリーンアーキテクチャ（オニオンアーキテクチャ、ヘキサゴナルアーキテクチャ）等による
            - アプリケーション層：業務ロジックこそがアプリケーションの本質だという表明
        - POCOで業務ロジックを実現できれば、業務ロジックだけに集中できる
- 実装する機能が、DDDマトリクスのどこに位置するかを考えてアーキテクチャを選定する→これがドメイン駆動開発の戦略
- Q3：以下の機能について、どのような方法で実装しますか？
    - 商品マスタの登録機能　→　アクティブレコード（シンプルなCRUD機能）
    - 顧客の趣向に合わせた商品のレコメンド機能　→　ドメインモデル（複雑な業務ロジック）
    - 決済機能　→　外部ライブラリ（すでに一般的な解決法が外部に提供されている）
- 補完的な領域では、業務ルールがシンプルなので、アクティブレコード（トランザクションスクリプト）で十分
    - ただし、機能開発を繰り返すうちに、複雑になってきたら注意
    - 補完的業務領域が、中核的業務領域に成長することもある
- ドメインモデルはコストが高いため、中核的な業務領域で複雑さに向き合うときにのみ利用する

---

# ドメイン駆動開発の戦術

- では複雑で独自性の高い分野はどのように実装すべきか→ドメインモデル
- ドメインモデルの実装には理解すべき前提が多い(ユビキタス言語・区切られた文脈・集約・エンティティ・値オブジェクト...)
- さらに理解が浅いまま単に真似するだけでは、むしろ複雑性を持ち込むだけに終わりそう
- まずは、いつでも使える値オブジェクトから始めよう

---

## 値オブジェクト

- あるエンティティ内のプロパティを、不変のモデリングされた値として表現すること
- お金をintで扱わずに、int Money型

### BAD①：業務（ドメイン）上の意味が読み取れないロジック

```csharp
var diff = (placedAt - shippedAt).TotalMinutes;

// 120って何？単位は？どのドメインの制約？
if (diff > 120) throw new Exception("NG");
```

### BAD②：あちこちに散らばった業務知識

```csharp
// Controller 
const int LimitMinutes = 120
// API POST: 120分チェック
if ((placedAt - shippedAt).TotalMinutes > LimitMinutes) return BadRequest();
・・・

// API PUT: 120分チェック
if ((placedAt - shippedAt).TotalMinutes > LimitMinutes) return BadRequest();
・・・

// API PATCH: 120分チェック
if ((placedAt - shippedAt).TotalMinutes > LimitMinutes) return BadRequest();
・・・

// BATCH
const int LimitMinutes = 120
status = (placedAt - shippedAt).TotalMinutes <= LimitMinutes? "OK" : "NG";
・・・
```

- こうなるとドキュメントで業務知識を補完するしかなくなる。

---

### GOOD：コードから業務知識が読み取れる

```csharp

// 利用箇所
var time = TransportTime.From(shippedAt, placedAt); // 120分超ならここで失敗（仕様が型にある）

// 定義
public sealed class TransportTime
{
    public int Minutes { get; private set; }
    public const int LimitMinutes = 120;

    private TransportTime() { } // EF用

    private TransportTime(int minutes)
    {
		    // ルールは一か所に凝集する
        if (minutes < 0) throw new ArgumentOutOfRangeException(nameof(minutes));
        if (minutes > LimitMinutes) throw new InvalidOperationException($"運搬時間は{LimitMinutes}分以内。");
        // もし夏季は60分以内で運搬する、というルールが発生した場合、このクラスにまとめる
        Minutes = minutes; // 常に不正ではない値だけが初期化される
    }

    public static TransportTime From(DateTime shippedAt, DateTime placedAt)
        => new((int)Math.Ceiling((placedAt - shippedAt).TotalMinutes));
}
```

- Q1：このような状態をなんという？
    - 高凝集→正解
    - 密結合→不正解
- 値オブジェクトを利用することで凝集度を高くすることができる
    - チェックロジックがあちこちに散らばっている状態を低凝集という

---

- ？：staticな値Helper（値Util）クラスじゃダメなの？
    - ①メソッド化しても、呼びだされなければ意味が無いので、リスクは残る
    - ②そのロジックが複雑に絡み合い、どんどん複雑化する可能性
        - 凝集レベルでいうと論理的凝集に当たり、7つの凝集度の中で下から２番目

### BAD①：呼び忘れる

```csharp
public static class DeliveryRules
{
    public static void EnsureWithin120(DateTime shippedAt, DateTime placedAt)
    {
        if ((placedAt - shippedAt).TotalMinutes > 120)
            throw new InvalidOperationException("運搬時間NG");
    }
}

// API PUT: ちゃんと呼ぶ
DeliveryRules.EnsureWithin120(e.ShippedAt, placedAt);
e.TransportMinutes = (int)(placedAt - e.ShippedAt).TotalMinutes;

// API POST: 呼び忘れてもコンパイルは通る（事故が起きる）
e.TransportMinutes = (int)(placedAt - e.ShippedAt).TotalMinutes;
```

### BAD②：論理的凝集に陥る

```csharp
public static class DeliveryRules
{
    public static void EnsureWithin120(...) { ... }
    public static void EnsurePhotoCount(...) { ... }
    public static void EnsureCustomerException(...) { ... }
    // “配送”に関するものが何でも入る → 増えるほど追えない
}
```

- しかも、同じクラスに置かれていることで、switchでの分岐やif elseの増加につながる
    - ここまでくると密結合

---

## 値オブジェクトで書かれたロジック

- Q：どちらが複雑？

### BAD：3つの自由な値

```csharp
public sealed class DeliveryBad
{
    public DateTime ShippedAt { get; init; }
    public DateTime PlacedAt  { get; init; }
    public int ElapsedMinutes { get; init; } // ←これも自由に入れられてしまう

    public bool IsWithin120()
        => ElapsedMinutes <= 120; // でも ShippedAt/PlacedAt と一致する保証がない
}

var bad = new DeliveryBad
{
    ShippedAt = new DateTime(2025, 12, 16,  9,  0, 0),
    PlacedAt  = new DateTime(2025, 12, 16, 12,  0, 0),//12-9で3時間（180分）
    ElapsedMinutes = 120　// 時間差は180分なのに ElapsedMinutes=120 が入ってしまう（嘘の状態）

};
```

### GOOD：2つの値と１つの計算される値

```csharp
public sealed class ConcreteDelivery
{
    public DateTime ShippedAt { get; }
    public DateTime PlacedAt  { get; }

    public ConcreteDelivery(DateTime shippedAt, DateTime placedAt)
    {
        if (placedAt < shippedAt) throw new ArgumentException("打設時刻は出荷時刻以降である必要があります。");
        ShippedAt = shippedAt;
        PlacedAt  = placedAt;
    }

    // “かかった時間”は状態として持たない。常に計算される。
    public TimeSpan TransportDuration => PlacedAt - ShippedAt;

    public void EnsureWithin120Minutes()
    {
        if (TransportDuration > TimeSpan.FromMinutes(120))
            throw new InvalidOperationException("運搬時間は120分以内でなければなりません。");
    }
}

// 利用例
var delivery = new ConcreteDelivery(
    shippedAt: new DateTime(2025, 12, 16,  9,  0, 0),
    placedAt:  new DateTime(2025, 12, 16, 10, 10, 0)
);

delivery.EnsureWithin120Minutes();

```

- 内部状態に関するすべてのロジックを値オブジェクトの中に記述する
- 値の自由度を下げ、複雑さを低減する
    - この考え方は、Reactにおいて計算によってStateをできるだけ減らそうという教えと同じ
    - 姓・名がある場合にはフルネームは不要…みたいな

---

# EFCoreで表現する値オブジェクト

## 翻って、EFCoreとは何だったか

- DBの都合を隠蔽し、文字列のSQLではなく、「オブジェクト」を中心とするものであった
    - ①オブジェクト中心であることで、レコード同士の関連を表現しやすくなる
    - ②(New)オブジェクト中心であることで、複雑な「業務ロジック」を表現（モデル化）しやすくなる
- 中核的領域では、業務ロジックは「本質的に複雑」であるため、オブジェクトを中心にする必要がある(DBを抽象化する)
- しかし、ただEFCoreを使うだけだと、トランザクションスクリプトとアクティブレコードの悪いとこ取りになる
    - レコードは**どこからでも変更**でき、しかも**手続き的に記述**されており、SQLの隠蔽だけが残る
    - これは貧血ドメインモデルと呼ばれる
        - 貧血ドメインモデルは“トランザクションスクリプト”と“アクティブレコード”両方の性質を持つ♣️
            
            ![ヒソカ.jpg](%E3%83%92%E3%82%BD%E3%82%AB.jpg)
            
    - 値オブジェクトによって、値にまつわるルールがカプセル化され、利用するロジックは宣言的になる

---

## Complex TypesとFluent APIで値オブジェクトをDBカラムに紐づける

- POCOの値オブジェクトをどうやってDBに紐づけるのか？ マッパーの独自実装が必要？→その必要は無い
- Complex Typesを利用して、値オブジェクトをマッピングする
- 値オブジェクトはPOCOである必要があり、フレームワークに依存させてはいけない
    - コンクリートの配送エンティティをPOCOで表現した例
    - おまじないはあるが、EFのフレームワークには依存していない

### Step1 POCOで値オブジェクトを書く
```csharp
public sealed class TransportTime
{
    public int Minutes { get; private set; }
    public const int LimitMinutes = 120;

    private TransportTime() { } // EF用

    private TransportTime(int minutes)
    {
		    // ルールは一か所に凝集する
        if (minutes < 0) throw new ArgumentOutOfRangeException(nameof(minutes));
        if (minutes > LimitMinutes) throw new InvalidOperationException($"運搬時間は{LimitMinutes}分以内。");
        // もし夏季は60分以内で運搬する、というルールが発生した場合、このクラスにまとめる
        Minutes = minutes; // 常に不正ではない値だけが初期化される
    }

    public static TransportTime From(DateTime shippedAt, DateTime placedAt)
        => new((int)Math.Ceiling((placedAt - shippedAt).TotalMinutes));
}
```

### Step2 Complex TypeとしてFluent APIでDBに結び付ける
- コンクリートの配送(ConcreteDelivery)というエンティティの中に、
    配送時間(TranceportTime)がある場合の例
```csharp
// DbContext
modelBuilder.Entity<ConcreteDelivery>()
    .ComplexProperty(x => x.TransportTime, ct =>
        ct.Property(p => p.Minutes).HasColumnName("TransportMinutes"));
```

---

- 注意点：ListのComplex Typesはサポートされていない

```csharp
List<TransportTime> TrancsportTimes // これはそのままだとテーブルに紐づけられない
```

- なぜか…？

### 値オブジェクトとエンティティ

- 値オブジェクト↔エンティティ
    - 不変↔可変
    - IDが無い↔IDがある
        - 値オブジェクトは、値が一緒であれば、同じものであるとみなす
- エンティティの一部として値オブジェクトがある
    - Complex TypesがListをサポートしていないことについて困った場合、
    それは値オブジェクトではなく、エンティティかも

```csharp
public class ConcreteDelivery // Aggregate Root(Entity)
{
    public Guid Id { get; private set; }
    public DateTime ShippedAt { get; private set; }

    // これはエンティティ
    public List<TransportCheckpoint> Checkpoints { get; private set; } = new();

    private ConcreteDelivery() { } // EF

    public void MarkPlaced(DateTime placedAt)
    {
        CurrentTransportTime = TransportTime.From(ShippedAt, placedAt); // ルールはVOに集約
        AddCheckpoint(placedAt);
    }

    public void AddCheckpoint(string kind, DateTime at)
    {
        var t = TransportTime.From(ShippedAt, at); // 120分ルール
        Checkpoints.Add(TransportCheckpoint.New(kind, at, t));
    }
}

public class TransportCheckpoint // Entity (履歴の1件)
{
    public Guid Id { get; private set; }   // ←識別子がある（つまりエンティティ）
    public Guid DeliveryId { get; private set; }
    public DateTime RecordedAt { get; private set; }
    public TransportTime TransportTime { get; private set; } = default!; // ここに値オブジェクトが現れる

    private TransportCheckpoint() { } // EF
    private TransportCheckpoint(Guid id, Guid deliveryId, DateTime at, TransportTime t)
        => (Id, DeliveryId, RecordedAt, TransportTime) = (id, deliveryId, at, t);

    public static TransportCheckpoint New(string kind, DateTime at, TransportTime t)
        => new(Guid.NewGuid(), Guid.Empty, at, t);
}

```

---

### OwnsManyを利用して、中間テーブルを値オブジェクトとみなす

- どうしても値オブジェクトのリストを持つべきだと思われるとき
    - Listの中身が重複せず、値が同じなら同じものであるとみなせるもの
    - 中間テーブルを介してマスタを参照するようなテーブル構造のもの
- 例えば：記事に対するタグ、エンジニアに対する資格一覧、工事に対する所属者IDなど
    - 「エンジニア」が持つ「資格」の例
    - IDという値を持つ値オブジェクトだと考えられる

---

```csharp
public readonly record struct QualificationId(string Value);

public sealed class Engineer
{
    public Guid Id { get; private set; }

    private readonly List<EngineerQualification> _qualifications = new();
    public IReadOnlyCollection<EngineerQualification> Qualifications => _qualifications;

    private Engineer() { } // EF

    public Engineer(Guid id) => Id = id;

    public void AddQualification(QualificationId qualificationId)
    {
        if (_qualifications.Any(x => x.QualificationId == qualificationId)) return;
        _qualifications.Add(new EngineerQualification(qualificationId));
    }

    public void RemoveQualification(QualificationId qualificationId)
        => _qualifications.RemoveAll(x => x.QualificationId == qualificationId);
}
```
``` csharp
// “エンジニアと資格の紐づき”だけを表す値オブジェクト（追加情報なしの中間テーブル）
public sealed class EngineerQualification
{
    public QualificationId QualificationId { get; private set; }

    private EngineerQualification() { } // EF
    public EngineerQualification(QualificationId qualificationId) => QualificationId = qualificationId;
}

// 資格情報マスタ
public sealed class QualificationMaster
{
    public string Id { get; private set; } = default!; // 例: "基本情報", "応用情報"
    public string Name { get; private set; } = default!;
}

```
    
- DbContextの定義

```csharp
using Microsoft.EntityFrameworkCore;

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Engineer>(b =>
    {
        b.HasKey(x => x.Id);

        b.OwnsMany(x => x.Qualifications, q =>
        {
            q.ToTable("EngineerQualifications");      // ← 中間テーブル
            q.WithOwner().HasForeignKey("EngineerId");

            // 値オブジェクトの中身を1列にマップ
            q.OwnsOne(x => x.QualificationId, id =>
            {
                id.Property(p => p.Value)
                    .HasColumnName("QualificationId")
                    .IsRequired();
            });

            // 中間テーブルなので複合キーにする（永続化上のキーであって、ドメインIDではない）
            q.HasKey("EngineerId", "QualificationId");

            // 同一エンジニア内での重複防止にもなる
        });
    });

    modelBuilder.Entity<QualificationMaster>(b =>
    {
        b.ToTable("QualificationMasters");
        b.HasKey(x => x.Id);
    });
}

```
    
- DB上はエンティティであるものをOwned Manyを用いて値オブジェクトとして定義可能
    - この値オブジェクトに振る舞いを持たせることはあまりないかもしれないが…
        - 他のプロパティが値オブジェクトになっているときに一貫性が出る
        - 集約的にうれしい（集約ルートからしか制御させない、トランザクションの境界）

---

## 値オブジェクトを利用するときの注意点

- 良い名前を付ける
    - 良い名前とは・・・？
    - DDDは**業務**の複雑さに向き合うものである
        - 業務エキスパートと同じ言葉を使う
- 同じ言葉（ユビキタス言語）
    - フロントエンドとバックエンドでもできる同じ言葉を使うようにする
- 別のものを同じにしてはいけない
    - 文脈によって意味が変わるものは別のものである
        - この時はこのルール、この時はこのルールとすると結局密結合・低凝集（論理的凝集）
    
    ### BAD：違うものを同じところに押し込める
    
    ```csharp
    public enum EmailUsage { Billing, Community }
    
    public sealed record Email(string Value, EmailUsage Usage)
    {
        public static Email Create(string value, EmailUsage usage)
        {
            // “メールアドレス”という同じ言葉に、文脈の差分を押し込めている
            return usage switch
            {
                EmailUsage.Billing   => value.EndsWith("@company.com") ? new(value, usage) : throw new Exception("請求文脈の制約NG"),
                EmailUsage.Community => !value.EndsWith("@company.com") ? new(value, usage) : throw new Exception("コミュニティ文脈の制約NG"),
                _ => throw new NotSupportedException()
            };
        }
    }
    
    ```
    
    ### GOOD：区切られた文脈によって、モジュールを分割する
    
    ```csharp
    namespace Billing; // 請求（Billing）という区切られた文脈
    
    public sealed record Email(string Value)
    {
        public static Email Create(string value)
            => value.EndsWith("@company.com")
               ? new(value)
               : throw new Exception("請求文脈：社用ドメイン必須");
    }
    ```
    
    ```csharp
    namespace Community; // コミュニティという区切られた文脈
    
    public sealed record Email(string Value)
    {
        public static Email Create(string value)
            => !value.EndsWith("@company.com")
               ? new(value)
               : throw new Exception("コミュニティ文脈：社用ドメイン禁止");
    }
    
    ```

--- 

- 注意：形容詞をつけて区別すると、同じ言葉ではなくなる
    - 業務エキスパートは、いちいち「請求の」「コミュニティの」アドレスと言うか？

### Warning：形容詞で区別してしまう

```csharp
public sealed record BillingEmail(string Value)
{
    public static BillingEmail Create(string value)
        => value.EndsWith("@company.com")
            ? new(value)
            : throw new Exception("請求先は社用メール必須");
}

public sealed record CommunityEmail(string Value)
{
    public static CommunityEmail Create(string value)
        => !value.EndsWith("@company.com")
            ? new(value)
            : throw new Exception("コミュニティは社用メール禁止");
}

```

---

- 値オブジェクトは、ドメイン駆動設計やEFCoreに限らず
オブジェクト指向言語であれば汎用的に利用可能な実装パターンである
- プリミティブ値に固執せず、型の情報と振る舞いの定義を与え、物体（オブジェクト）として型にする
- 業務上の言葉を**モデル化**する行為ともいえる

---

# ドメインモデル

- 値オブジェクト、エンティティ、（そして集約）はドメインモデルを構成する部品である
- ドメインモデルとは、業務をモデル化することがその本質である。
    - モデル化とは、リアルの物体やプロセスに対して、その場で役に立つ部分だけ抜き出して抽象化することである。
    - 一つのモデルですべてを表現するのはモデルではない（役に立たない）
        - 例えば地図は、現実世界をそのままではなく、必要な文脈に応じて、必要な情報だけが表現されている。
            - 地球儀・メルカトル図法・正距方位図法

Q：引っ越しの時に、冷蔵庫が家のドアを通れるか確認したいです。あなたはどうやって確認しますか？

- A:段ボールを冷蔵庫の底面の大きさに切って、それで確かめる
    - この段ボールがモデルである
    - このモデルに、例えば重さも表現したいからといって重りを乗せ始めたら、モデルとして役に立たなくなる
- 役に立つモデルを探索するために、業務エキスパートと、ユビキタス言語で会話し、区切られた文脈を発見しよう

---

## ドメインモデルをいつやるか

- 値オブジェクトやドメインモデルについての率直な感想
    - ファイルが増えて追いずらそう、めんどい、やったことない、学習コスト・・・
- **どこでも使えばいいってものではない**
- 戦略的DDDによれば、優秀なエンジニアは、ドメインモデルを利用して中核的業務領域の本質的な複雑さに向き合うものである
    - 皆さんは十分すぎるほど優秀で、そして事業価値をもたらすべき存在です。
- 今は中核的業務領域と呼べるほど複雑でなくても、必要な時はいつやってくるかわからない
    - サブスクリプションビジネスにおいては、スピーディーで機能改善を**継続的に**行うことが必須
    - 今はシンプルでも、その製品が成功すればするほどに、いつかは複雑になっていく
- 我々のアプリケーションは、少しずつ汚れてきていないか？
    - 汚れたコードはこれまで価値を提供してきた証拠でもある、恥じることはない…
        - 嫌な（しかし役に立つ）経験則→泥団子のコードは中核的な業務領域である
- 拡大期にあるわが社、エンジニアの流動化は必至
（FE→BE転向や採用による新規アサイン、育休産休・介護休）
    - あなたの頭の中にある業務知識は、いつかは引き継がなければならない
    - あなたはいつか、見ず知らずのコードに飛び込む必要がある
- →だから業務のモデル化やコードの自己文書化が必要

---

- つまり、いつかは複雑さに**真正面から**取り組むべき時が来る
    - 業務ロジックが凝集され、カオスにならないコードベース
        - 美しいまな板のようなもの
    - 業務エキスパートと深く相互理解された洗練されたユビキタス言語・それらが生み出す役に立つ（業務ロジックを雄弁に語ることができる）モデル
        - 切れ味のいい包丁のようなもの
    - これらが整うことで、いつでも新しく魅力的な機能（料理）を産み出すことができる
- ただEFCoreを使うだけだと、汚れたまな板とさびた包丁で料理するようなことになっていく
    - どんなに料理人（エンジニア）の腕が良くても、スピード（生産性）が落ち、出来上がる料理（機能）の品質は低下する

---

# まとめ

- 自己文書化された雄弁なコードにより
    - 将来の辛く苦しい仕事を回避すべき
    - スピーディーな開発生産性を維持できる
- ドメインモデルの実践により、POCOで業務ロジックを記述することに集中できる
    - CRUDではなく、ビジネスそのものの複雑さに
- EFCoreを利用しているのは、複雑なリレーションをオブジェクトとして表現し、さらにオブジェクトを値オブジェクトとして業務ロジックを表現するため
- 我々が実装すべきなのは、本質的に複雑な中核的領域であり、DDDが効く範囲
    - どこでもやるというわけではない
- まずは、値オブジェクトを使っていこう