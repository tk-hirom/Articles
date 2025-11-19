---
title: "Kotlin Coroutineを噛み砕いて理解する"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kotlin", "coroutine", "android"]
published: false
---

## はじめに

現在、遊撃的にWebFluxの刷新しており、Kotlin Coroutineを活用しています。Coroutineを使えば同期的に書けるコードでありながら、並行処理のメリットを享受できる点に大きな魅力を感じていました。しかし、実際に使っていく中で「雰囲気で触っているだけで、本質を理解できていないのでは？」という疑問が湧いてきました。そこで、公式ドキュメントや関連資料を改めてさらい、基礎から学び直すことにしました。

この記事では、Kotlinのコルーチン（Coroutine）の基本概念を、**Kotlinでバックエンド開発をする方**を想定読者として、実践的な例を使って噛み砕いて説明します。Spring WebFluxなどのフレームワークを使っている方、これから使おうと考えている方の理解の助けになれば幸いです。

## コルーチンの概要と必要性

### 同期的なコードの問題

まず、最もシンプルな同期的なコードから見てみましょう。WebAPIでデータベースからユーザー情報を取得する処理を考えてみます：

```kotlin
@RestController
class UserController(private val userRepository: UserRepository) {
    
    @GetMapping("/users/{id}")
    fun getUser(@PathVariable id: Long): User {
        val user = userRepository.findById(id) // DB問い合わせ: 100msかかる
        return user
    }
}
```

このコードは一見シンプルで分かりやすいですが、致命的な問題があります。`findById()` が完了するまでの100ms間、**リクエストを処理しているスレッドが完全にブロック**されてしまうのです。

Tomcatのデフォルト設定では200スレッドしかないため、200件の同時リクエストでスレッドプールが枯渇します。201件目のリクエストは待たされることになり、スループットが大きく制限されます。

この問題を解決するには、時間のかかる処理を**非同期**で実行する必要があります。つまり、I/O待機中にスレッドを解放し、他のリクエスト処理に使えるようにするということです。

### 従来のアプローチ：スレッドによる非同期処理

この問題に対する伝統的な解決策は、スレッドプールを使って処理を別のスレッドで実行することでした：

```kotlin
@RestController
class UserController(
    private val userRepository: UserRepository,
    private val executorService: ExecutorService
) {
    @GetMapping("/users/{id}")
    fun getUser(@PathVariable id: Long): CompletableFuture<User> {
        return CompletableFuture.supplyAsync({
            userRepository.findById(id) // 別スレッドで実行
        }, executorService)
    }
}
```

この方法で、リクエストハンドリングスレッドをブロックせずに処理を実行できます。しかし、スレッドベースのアプローチには深刻な制約があります。

まず、**リソースの制約**です。スレッドは非常に「重い」リソースで、1つあたり約1MB〜2MBのメモリを消費します。そのため、実用的には数百個〜数千個程度しか作成できません。また、スレッド間の切り替え（コンテキストスイッチ）はCPUに負荷をかけ、大量のスレッドがあるとパフォーマンスが急激に悪化します。

さらに、**開発の複雑さ**も問題です。スレッド間の通信は煩雑で、バックエンドでもUIスレッド（メインスレッド）への切り戻しを明示的に行う必要があります。加えて、エラーハンドリングやキャンセル処理が複雑になりがちで、デッドロックやレースコンディションといったバグを生みやすくなります。

### スレッドの限界：スケーラビリティの問題

例えば、1万件の同時接続を処理する必要がある場合（いわゆるC10K問題）を考えてみましょう。各接続に1つのスレッドを割り当てるアプローチでは：

- **メモリ消費**: 10,000スレッド × 2MB = 約20GBのメモリが必要
- **コンテキストスイッチのオーバーヘッド**: 大量のスレッド間の切り替えでCPUリソースが枯渇
- **スケーラビリティの限界**: 接続数が増えるほどパフォーマンスが急激に悪化

この問題に対処するため、従来はイベントループやノンブロッキングI/Oといった複雑なアーキテクチャパターンが必要でした。しかし、これらのアプローチはコードを複雑にし、理解や保守を困難にします。

これらの問題に対処できるのがコルーチンです。

### コルーチンによる軽量な非同期処理

Kotlin Coroutineは、スレッドの制約を克服しつつ、シンプルなコードで非同期処理を実現する仕組みです。簡単に言えば「途中で一時停止・再開できる、軽量な処理の単位」です。

コルーチンがスレッドの問題を解決できる理由は、3つの大きな特徴にあります。

**特徴1：圧倒的な軽量性**

スレッドが1つあたり1MB〜2MBのメモリを消費するのに対し、コルーチンは1つあたり数十バイト程度しか消費しません。この違いにより、数万〜数十万個のコルーチンを同時に実行可能で、10万件の同時接続も現実的に処理できるようになります。

**特徴2：効率的なリソース利用**

コルーチンはI/O待ち時にスレッドを占有しません。待機中のスレッドは他のコルーチンが利用できるため、少数のスレッド（例えば数十個）で大量の並行処理を実現できます。これにより、スレッドプールが枯渇する問題から解放されます。

**特徴3：構造化された並行性**

コルーチンには「親コルーチンがキャンセルされると、子も自動的にキャンセルされる」という仕組みが組み込まれています。これにより、メモリリークを防ぐライフサイクル管理が自動的に行われ、リソースの適切なクリーンアップが保証されます。

これらの特徴により、コルーチンを使うことで、スレッドのような複雑さを避けながら、スケーラブルで効率的な非同期処理を実現できます。次のセクションでは、コルーチンの基本構成要素を詳しく見ていきましょう。

## コルーチンの基本構成要素

コルーチンを使いこなすには、3つの基本要素を理解する必要があります。順番に見ていきましょう。

### 1. コルーチンビルダー：コルーチンを「起動」する関数

コルーチンを起動するための関数がコルーチンビルダーです。主に3つあります。

#### launch：「起動して、結果は気にしない」

`launch`は「とにかく実行して！結果は要らない！」という時に使います。

```kotlin
fun main() = runBlocking {
    println("プログラム開始")
    
    // launchで新しいコルーチンを起動
    launch {
        delay(1000) // 1秒待つ
        println("1秒後のメッセージ")
    }
    
    println("launchの後すぐ実行される")
    // プログラム開始
    // launchの後すぐ実行される
    // （1秒後）1秒後のメッセージ
}
```

`launch`には3つの重要な特徴があります。まず、戻り値として`Job`（処理の管理チケットのようなもの）を返します。次に、処理の結果を返さない（返せない）設計になっています。そのため、「ログ出力」「データ保存」など、結果が不要な処理に最適です。

戻り値の`Job`を使うと、起動したコルーチンを制御できます：

```kotlin
val job = launch {
    repeat(10) {
        println("処理中... $it")
        delay(500)
    }
}

// 他の処理...

job.cancel() // 処理をキャンセル
// または
job.join() // 処理の完了を待つ
// または
job.cancelAndJoin() // キャンセルして完了を待つ
```

#### async / await：「起動して、結果を受け取りたい」

`async`は「実行して、後で結果が欲しい！」という時に使います。

```kotlin
fun main() = runBlocking {
    println("計算開始")
    
    // asyncで計算を開始（Deferredを返す）
    val deferred = async {
        delay(1000) // 重い計算を模擬
        42 // 計算結果
    }
    
    println("計算は別で動いてる。他の処理もできる")
    delay(500)
    println("500msec経過")
    
    // awaitで結果を取得（まだ終わってなければ待つ）
    val result = deferred.await()
    println("計算結果: $result")
}
```

**出力**：

```
計算開始
計算は別で動いてる。他の処理もできる
500msec経過
計算結果: 42
```

**async/awaitの使い分け**：

```kotlin
// ❌ こうすると遅い（逐次実行）
suspend fun fetchDataSlow(): Pair<User, List<Post>> {
    val user = fetchUser() // 1秒かかる
    val posts = fetchPosts() // 1秒かかる
    return user to posts
    // 合計2秒
}

// ✅ asyncを使うと速い（並列実行）
suspend fun fetchDataFast(): Pair<User, List<Post>> = coroutineScope {
    // 2つを「同時に」開始
    val userDeferred = async { fetchUser() } // 1秒かかる
    val postsDeferred = async { fetchPosts() } // 1秒かかる
    
    // 両方の結果を待つ
    val user = userDeferred.await()
    val posts = postsDeferred.await()
    
    return@coroutineScope user to posts
    // 合計1秒（並列実行されるから）
}
```

**レストランの例で理解する**：

- `launch`: 「料理を作っておいて（結果は要らない）」
- `async/await`: 「料理を作っておいて、できたら持ってきて（結果が欲しい）」

#### runBlocking：「普通の世界とコルーチンの世界を繋ぐ」

`runBlocking`は特殊なビルダーで、「コルーチンじゃない普通の関数」から「コルーチンの世界」に入るための橋渡しです。

```kotlin
// 普通の関数（コルーチンじゃない）
fun main() = runBlocking { // ← ここでコルーチンの世界に入る
    // ここからコルーチンの世界
    launch {
        delay(1000)
        println("完了")
    }
    println("待機中")
} // ← ここで、中のコルーチンが全部終わるまで待つ
```

`runBlocking`の最も重要な特徴は、中のコルーチンが全て終わるまで、呼び出したスレッドをブロック（待機）することです。この特性から、**本番のアプリコードでは使わない**のが鉄則です。

ただし、以下の3つの場面では適切に使用できます。1つ目は`main`関数（学習用のサンプルコード）、2つ目はテストコード、3つ目は既存の同期的なコードとの橋渡しです。

では、なぜ本番コードで避けるべきなのでしょうか？

`runBlocking`は名前の通り、スレッドを「ブロック」します。特にWebアプリケーションでリクエストハンドリングスレッドで`runBlocking`を使うと、スレッドプールが枯渇する原因になります。

```kotlin
// ❌ 悪い例：リクエストスレッドをブロックする
@RestController
class UserController(private val userService: UserService) {
    
    @GetMapping("/users/{id}")
    fun getUser(@PathVariable id: Long): User {
        return runBlocking { // リクエストスレッドが3秒間ブロック！
            delay(3000)
            userService.findById(id)
        }
    }
}

// ✅ 良い例：サスペンド関数を使う
@RestController
class UserController(private val userService: UserService) {
    
    @GetMapping("/users/{id}")
    suspend fun getUser(@PathVariable id: Long): User {
        delay(3000) // スレッドはブロックされず、他のリクエストを処理できる
        return userService.findById(id)
    }
}
```

**使用例：テストコード（これは適切な使用例）**

```kotlin
@Test
fun testSuspendFunction() = runBlocking {
    // テストでは問題ない（テストスレッドをブロックするだけ）
    val result = fetchUser("123")
    assertEquals("太郎", result.name)
}
```

### 2. CoroutineScope：コルーチンの「実行範囲」を定義

#### CoroutineScopeとは？

CoroutineScopeは、**コルーチンの寿命を管理する枠組み**です。

前のセクションで学んだ`launch`や`async`といったコルーチンビルダーは、実は必ず「どこかのScope」の中で実行する必要があります。例えば：

```kotlin
// ❌ これはエラー！Scopeがない
fun someFunction() {
    launch { // コンパイルエラー：Unresolved reference: launch
        // 処理
    }
}

// ✅ これはOK：Scopeの中で実行
fun someFunction() = runBlocking { // runBlockingがScopeを提供
    launch { // Scopeがあるので起動できる
        // 処理
    }
}
```

つまり、**`launch`や`async`といったコルーチンビルダーは、必ず「どこかのScope」の中でしか使えない**という関係になっています。

:::details launchはCoroutineScopeの拡張関数
Kotlinの機能で、既存のクラスを変更せずに新しい関数を追加できる仕組みです。例えば：

```kotlin
// String型に新しい関数を追加する例
fun String.addPrefix(): String = "PREFIX_$this"

// 使い方
"test".addPrefix() // "PREFIX_test"
```

`launch`は`CoroutineScope`の拡張関数として定義されているため、実際には`CoroutineScope.launch()`のように「Scopeに対して」呼び出す形になります。そのため、Scopeがない場所では`launch`を呼び出せません。
:::

Scopeには次のような役割があります：

**役割1：コルーチンの実行範囲を定める**

Scopeは「このコルーチンはどの範囲で実行されるべきか」を定義します。例えば：

- リクエストのスコープ：リクエスト処理が終わったら終了
- サービスのスコープ：アプリケーションが動いている間は有効

**役割2：親子関係による自動管理**

Scopeの中で起動されたコルーチンは、親子関係を持ちます：

```kotlin
scope.launch { // 親コルーチン
    launch { // 子コルーチン1
        // 処理A
    }
    launch { // 子コルーチン2
        // 処理B
    }
}
```

この親子関係により、次のような自動管理が行われます：

- 親がキャンセルされると、子も自動的にキャンセルされる
- 子のどれかが例外を投げると、親に伝播する
- 親は全ての子が完了するまで待つ

**役割3：リソースリークの防止**

Scopeを適切に使うことで、コルーチンが不要になったときに自動的にクリーンアップされます。これにより、メモリリークやCPUの無駄遣いを防げます。

では、適切にScopeを切らないとどんな問題が起こるのでしょうか？次のセクションで具体例を見てみましょう。

#### 適切にScopeを切らないとどうなるか？

例えば、WebAPIでバックグラウンドで大量のデータ処理を開始したとします：

```kotlin
// ❌ 悪い例
@RestController
class DataProcessController {
    
    @PostMapping("/process")
    fun startProcessing(): String {
        GlobalScope.launch {
            val data = fetchLargeDataSet() // 10分かかる
            processData(data) // データを処理
            saveResults(data) // 結果を保存
        }
        return "Processing started"
    }
}
```

**このコードは何をしているのか？**

このコードで使われている`GlobalScope.launch { ... }`という構文について説明します。

- `launch`：前述したコルーチンビルダーで、コルーチンを起動して`Job`を返す関数です
- `GlobalScope`：「アプリケーション全体」というスコープ（実行範囲）を表します
- `{ }`（波括弧）の中：バックグラウンドで非同期に実行される処理

つまり、`GlobalScope.launch { ここの処理 }`は「アプリケーション全体の寿命で、バックグラウンド処理を実行する」という意味です。前述の通り、`launch`は結果を返さず、起動したコルーチンの管理用の`Job`を返します。

このAPIエンドポイントの動作の流れ：

1. クライアントから`/process`にPOSTリクエストが届く
2. `GlobalScope.launch`でバックグラウンド処理を起動（コルーチンを開始）
3. **すぐに**「Processing started」というレスポンスをクライアントに返す（処理の完了を待たない）
4. その間、バックグラウンドでは波括弧内の処理（データ取得→処理→保存）が10分かけて実行される

一見すると「重い処理をバックグラウンドで実行しているから問題ない」ように見えますが、実はこのコードには深刻な問題が隠れています。特に`GlobalScope`という「アプリケーション全体のスコープ」を使っている点が致命的です。

**何が問題なのか？**

このコードには深刻な問題があります。特に問題になるのは、アプリケーションをシャットダウンしようとしたときです。実際の運用シーンを想定して、タイムラインで何が起こるか見てみましょう：

```text
時刻 0:00 - クライアントがAPIリクエストを送信
時刻 0:01 - サーバーがリクエストを受け取り、GlobalScope.launchで処理を開始
時刻 0:02 - クライアントに "Processing started" というレスポンスを返す
時刻 0:05 - データ取得処理が進行中...（fetchLargeDataSet実行中）
時刻 0:10 - 運用チームがアプリケーションの再起動を決定（デプロイのため）
時刻 0:11 - アプリケーションのシャットダウンシグナルが送られる
```

このタイミングで、4つの深刻な問題が同時に発生します：

**問題1：コルーチンがまだ実行中**

データ取得処理は10分かかる予定で、まだ8分残っています。`GlobalScope`で起動されたコルーチンはアプリケーション全体のライフサイクルに紐づいているため、シャットダウンシグナルを受け取っても自動的にはキャンセルされません。

**問題2：アプリケーションが終了できない**

Springなどのフレームワークは、実行中のタスクがある場合、終了を待ちます。最悪の場合、8分間シャットダウンがブロックされ、運用チームは「なぜアプリが終了しないのか？」と困惑することになります。

**問題3：強制終了するとデータ不整合**

待ちきれずにタイムアウトして強制終了すると、処理が中断されます。その結果、データが半端な状態で残る可能性があります。例えば、データは取得したが保存されていない、保存は始まったが完了していない、トランザクションがコミットされていない、といった状態です。

**問題4：リソースリーク**

リクエストが終了しても、コルーチンは動き続けます。同じような処理が何百件も動いていたら、メモリとCPUが無駄に消費され続けることになります。

さらに悪いことに、この問題は1つのリクエストだけでは顕在化しません。以下のように複数のリクエストが来た場合を考えてみましょう：

```kotlin
// 100件のリクエストが来たとする
repeat(100) {
    GlobalScope.launch {
        val data = fetchLargeDataSet() // 各10分
        processData(data)
        saveResults(data)
    }
}
// リクエスト処理は即座に終了
// でも、100個のコルーチンが10分間動き続ける
// = 大量のメモリとCPUリソースが消費される
```

このコードの怖いところは、APIは正常にレスポンスを返すため、一見問題がないように見える点です。しかし、バックグラウンドでは100個のコルーチンが動き続け、サーバーのリソースを圧迫し続けているのです。

#### 解決策：CoroutineScopeで寿命を管理

Spring WebFluxでは、適切なスコープ管理が重要です：

```kotlin
// ✅ 良い例 - Spring WebFluxの場合
@Service
class DataProcessService : CoroutineScope {
    private val job = SupervisorJob()
    override val coroutineContext: CoroutineContext = Dispatchers.IO + job
    
    suspend fun processData(data: Data): Result {
        return withContext(coroutineContext) {
            // この処理はServiceのライフサイクルに紐付いている
            val processed = heavyProcessing(data)
            saveToDatabase(processed)
            Result.success(processed)
        }
    }
    
    @PreDestroy
    fun cleanup() {
        job.cancel() // サービス終了時に全コルーチンをキャンセル
    }
}
```

#### Spring WebFluxでの実践的な使い方

Spring WebFluxでは、コルーチンを使ったサスペンド関数がそのまま使えます：

```kotlin
@RestController
class UserController(private val userService: UserService) {
    
    // サスペンド関数として定義するだけ
    @GetMapping("/users/{id}")
    suspend fun getUser(@PathVariable id: Long): User {
        return userService.findById(id) // 自動的に適切なスコープで実行される
    }
    
    @PostMapping("/users")
    suspend fun createUser(@RequestBody request: CreateUserRequest): User {
        return userService.create(request)
    }
}

@Service
class UserService(private val userRepository: UserRepository) {
    
    // Repositoryもサスペンド関数にできる
    suspend fun findById(id: Long): User {
        return userRepository.findById(id) ?: throw UserNotFoundException(id)
    }
}
```

**覚えておくべきこと**：

- `GlobalScope`：**⚠️ 本番コードでは使用禁止**（理由は後述）
- Spring WebFlux：サスペンド関数を使えば自動的にリクエストスコープで実行
- 長時間実行されるバックグラウンドタスク：独自のScopeを作って`@PreDestroy`で適切にクリーンアップ

#### 主なCoroutineScopeの種類

実際の開発では、いくつかの代表的なScopeを使い分けます：

**1. GlobalScope（アプリケーション全体のスコープ）**

```kotlin
GlobalScope.launch {
    // アプリケーションが終了するまで動き続ける
}
```

- **使用場面**：ほぼ使わない（本番コードでは非推奨）
- **特徴**：アプリケーション全体の寿命と同じ
- **問題点**：ライフサイクル管理ができず、メモリリークの原因になる

**2. coroutineScope（構造化された一時的なスコープ）**

```kotlin
suspend fun processData() = coroutineScope {
    val result1 = async { fetchData1() }
    val result2 = async { fetchData2() }
    
    // 両方が完了するまで待つ
    combineResults(result1.await(), result2.await())
}
```

- **使用場面**：suspend関数内で複数の並列処理をまとめたい時
- **特徴**：すべての子コルーチンが完了するまで関数が終わらない
- **メリット**：構造化された並行性により、リソースリークを防げる

**3. カスタムScope（独自のライフサイクルを持つスコープ）**

```kotlin
class MyService : CoroutineScope {
    private val job = SupervisorJob()
    override val coroutineContext = Dispatchers.Default + job
    
    fun startBackgroundTask() {
        launch {
            // このServiceのライフサイクルに紐付いた処理
        }
    }
    
    fun cleanup() {
        job.cancel() // Serviceの終了時に全コルーチンをキャンセル
    }
}
```

- **使用場面**：サービスやコンポーネントに紐付いた長期実行タスク
- **特徴**：ライフサイクルを明示的に管理できる
- **メリット**：適切なタイミングでクリーンアップできる

**4. Spring WebFluxの自動管理（リクエストスコープ）**

```kotlin
@RestController
class UserController {
    @GetMapping("/users/{id}")
    suspend fun getUser(@PathVariable id: Long): User {
        // Spring WebFluxが自動的にリクエストスコープで実行
        return userService.findById(id)
    }
}
```

- **使用場面**：Spring WebFluxでのWeb APIエンドポイント
- **特徴**：フレームワークが自動的にスコープを管理
- **メリット**：明示的なScope管理が不要

### 3. suspend関数：「一時停止できる関数」の目印

`suspend` キーワードは「この関数は途中で一時停止するかもしれない」という目印です。

そして、一時停止している間は**スレッドを解放**できるという重要な特徴があります。これにより、そのスレッドは他のコルーチンが利用できるようになります。

#### なぜsuspendが必要なのか？

ネットワーク通信のように「結果が返ってくるまで待つ」処理では、その待ち時間にスレッドを占有するのはもったいないですよね。

```kotlin
// suspend関数（途中で一時停止できる）
suspend fun fetchUserData(): User {
    return withContext(Dispatchers.IO) {
        delay(1000) // ← ここで一時停止（スレッドは解放される）
        apiService.getUser() // ネットワーク通信
    }
}
```

#### suspend関数の重要なルール

**ルール1**: suspend関数は、他のsuspend関数かコルーチンの中からしか呼べない

```kotlin
// ❌ これはエラー！
fun normalFunction() {
    fetchUserData() // コンパイルエラー：suspend関数は普通の関数から呼べない
}

// ✅ これはOK
suspend fun anotherSuspendFunction() {
    fetchUserData() // suspend関数同士ならOK
}

// ✅ これもOK（実際のWebアプリの場合）
@Service
class UserService(private val userRepository: UserRepository) {
    suspend fun loadUserData(id: Long): User {
        return userRepository.findById(id) // コルーチンの中ならOK
    }
}
```

**なぜこのルールがあるの？**
suspend関数は「一時停止」という特殊な処理をします。普通の関数はそれを理解できないので、コルーチンという特別な環境の中でしか実行できないのです。

#### 実用例：外部API呼び出し

```kotlin
// ユーザー情報を取得する（suspend関数）
suspend fun fetchUser(id: Long): User {
    // withContextで「IO処理用のスレッド」に切り替え
    return withContext(Dispatchers.IO) {
        // 外部API呼び出し（時間がかかる）
        externalApiClient.getUser(id) // ここで一時停止
    }
}

// 投稿一覧を取得する（suspend関数）
suspend fun fetchPosts(userId: Long): List<Post> {
    return withContext(Dispatchers.IO) {
        externalApiClient.getPosts(userId) // ここで一時停止
    }
}

// これらを使う（Spring WebFluxのControllerの場合）
@RestController
class UserController(
    private val userService: UserService,
    private val postService: PostService
) {
    @GetMapping("/users/{id}/with-posts")
    suspend fun getUserWithPosts(@PathVariable id: Long): UserWithPosts {
        val user = fetchUser(id) // 一時停止して待つ
        val posts = fetchPosts(user.id) // 一時停止して待つ
        return UserWithPosts(user, posts)
    }
}
```

この例のポイント：

- `fetchUser`と`fetchPosts`は両方とも suspend関数
- 呼び出し側から見ると、普通の関数のように「順番に」書ける
- でも内部では、待っている間スレッドを解放して他の処理ができる

## ディスパッチャー：「どこで」実行するかを制御

ディスパッチャー（Dispatcher）は「このコルーチンをどのスレッドで実行するか」を決める仕組みです。

### なぜディスパッチャーが必要？

Webアプリケーションを例に考えましょう：

- **CPU集約的な処理**：暗号化、大量のデータ処理など、専用のスレッドプールで実行すべき
- **I/O処理**：データベースアクセス、外部API呼び出しなど、I/O専用のスレッドプールで実行すべき
- **軽量な処理**：簡単な計算やデータ変換など、現在のスレッドで実行すれば十分

つまり、処理内容によって「適切なスレッドプール」が違うんです。

### 4つの主要なディスパッチャー

#### 1. Dispatchers.Default（CPU集約的な処理用）

デフォルトのスレッドプールです。CPU集約的な処理（計算、ソート、JSON解析など）に適しています。

```kotlin
withContext(Dispatchers.Default) {
    // CPU集約的な処理
    val result = complexCalculation(largeDataSet)
    val sorted = data.sortedByDescending { it.value }
}
```

`Dispatchers.Default`を使うべきなのは、CPUを集中的に使う処理です。具体的には、大量のデータの処理・変換、複雑な計算、JSON/XMLのパース、暗号化/復号化などが該当します。これらの処理は「計算そのもの」に時間がかかるため、CPU専用のスレッドプールで実行するのが適切です。

#### 2. Dispatchers.IO（入出力処理用）

ネットワーク通信、ファイル読み書き、データベース操作用です。

```kotlin
@Service
class UserService(
    private val externalApiClient: ExternalApiClient,
    private val userRepository: UserRepository
) {
    suspend fun loadUserFromExternalApi(id: Long): User {
        return withContext(Dispatchers.IO) {
            // I/O処理専用のスレッドプールで実行
            externalApiClient.getUser(id)
        }
    }
    
    suspend fun findById(id: Long): User {
        return withContext(Dispatchers.IO) {
            userRepository.findById(id) ?: throw NotFoundException()
        }
    }
}
```

`Dispatchers.IO`を使うべきなのは、入出力を伴う処理です。具体的には、外部APIの呼び出し、ファイルの読み書き、データベースのクエリ、Redis/キャッシュへのアクセスなどです。これらの処理は「待ち時間」が主体で、CPUをほとんど使わないのが特徴です。

`Dispatchers.IO`は専用のスレッドプールで管理されており、デフォルトで最大64個までスレッドを使用できます。I/O処理は待ち時間が多いため、1つのスレッドを多くのコルーチンで共有でき、効率的にリソースを活用できます。

#### 3. Dispatchers.Unconfined（特殊用途）

呼び出し元のスレッドで開始し、最初のサスペンドポイント後は再開したスレッドで継続します。通常は使用しません。

#### 4. Dispatchers.Unconfined（特殊用途）

最初は呼び出したスレッドで実行され、一時停止後は再開したスレッドで実行されます。

```kotlin
launch(Dispatchers.Unconfined) {
    println("1: ${Thread.currentThread().name}") // 呼び出し元のスレッド
    delay(100)
    println("2: ${Thread.currentThread().name}") // 別のスレッド
}
```

**注意**：特殊な用途以外では使わないでください。

### ディスパッチャーの切り替え：withContext

`withContext`を使うと、一時的に別のディスパッチャーに切り替えられます。

```kotlin
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val externalApiClient: ExternalApiClient,
    private val notificationService: NotificationService
) {
    suspend fun processOrder(orderId: Long) {
        // I/O処理でデータ取得
        val order = withContext(Dispatchers.IO) {
            orderRepository.findById(orderId)
        }
        
        // CPU集約的な処理で計算
        val total = withContext(Dispatchers.Default) {
            calculateComplexTotal(order)
        }
        
        // 再びI/O処理で外部API呼び出し
        withContext(Dispatchers.IO) {
            externalApiClient.confirmPayment(order, total)
        }
        
        // I/O処理でデータベース更新
        withContext(Dispatchers.IO) {
            orderRepository.updateStatus(orderId, "COMPLETED")
        }
        
        // 非同期で通知を送信（結果を待たない）
        notificationService.sendOrderConfirmation(orderId)
    }
    
    private fun calculateComplexTotal(order: Order): BigDecimal {
        // CPU集約的な計算
        return order.items.map { it.price * it.quantity }
            .fold(BigDecimal.ZERO) { acc, price -> acc + price }
    }
}
```

`withContext`には3つの便利なポイントがあります。

1. 処理が終わると自動的に元のディスパッチャーに戻る
2. コードが読みやすい（どこで何が実行されるか明確）
3. エラーが起きても正しく元のスレッドに戻る

### 実例：複数APIの並列呼び出し

```kotlin
@RestController
class DashboardController(private val dashboardService: DashboardService) {
    
    @GetMapping("/dashboard")
    suspend fun getDashboard(): DashboardData {
        return dashboardService.loadDashboardData()
    }
}

@Service
class DashboardService(
    private val userApiClient: UserApiClient,
    private val orderApiClient: OrderApiClient,
    private val notificationApiClient: NotificationApiClient
) {
    suspend fun loadDashboardData(): DashboardData = coroutineScope {
        // 3つの外部APIを同時に呼び出し
        val userDeferred = async(Dispatchers.IO) { userApiClient.getUser() }
        val ordersDeferred = async(Dispatchers.IO) { orderApiClient.getOrders() }
        val notificationsDeferred = async(Dispatchers.IO) { notificationApiClient.getNotifications() }
        
        // 全ての結果を待つ（並列実行なので速い）
        val user = userDeferred.await()
        val orders = ordersDeferred.await()
        val notifications = notificationsDeferred.await()
        
        DashboardData(user, orders, notifications)
    }
}
```

### ディスパッチャーの選び方フローチャート

```text
時間のかかる処理？
├─ Yes → どんな処理？
│   ├─ ネットワーク/DB/ファイル → Dispatchers.IO
│   └─ 計算/データ処理 → Dispatchers.Default
└─ No → ディスパッチャー指定不要（現在のスレッドで実行）
```

## 実践例：複数のAPI呼び出しを速くする

ここまでの知識を使って、実際によくある「複数の外部APIを呼ぶ」場面を最適化してみましょう。

### シナリオ：統合APIエンドポイントの実装

統合APIエンドポイントを実装するには、以下の3つの外部サービスから情報を取得する必要があります：

1. ユーザーサービスからユーザー情報（1秒かかる）
2. 注文サービスから注文履歴（1秒かかる）
3. ポイントサービスからポイント残高（1秒かかる）

### パターン1：逐次実行（遅い、初心者がやりがち）

```kotlin
@Service
class UserProfileService(
    private val userApiClient: UserApiClient,
    private val orderApiClient: OrderApiClient,
    private val pointApiClient: PointApiClient
) {
    // ❌ 悪い例：順番に待つので遅い
    suspend fun getUserProfile(userId: Long): UserProfileData {
        // 1. ユーザー情報を取得（1秒）
        val user = withContext(Dispatchers.IO) {
            userApiClient.getUser(userId)
        }
        
        // 2. 注文履歴を取得（1秒）
        val orders = withContext(Dispatchers.IO) {
            orderApiClient.getOrders(userId)
        }
        
        // 3. ポイント残高を取得（1秒）
        val points = withContext(Dispatchers.IO) {
            pointApiClient.getPoints(userId)
        }
        
        return UserProfileData(user, orders, points)
        // 合計：3秒かかる
    }
}
```

**問題点**：

- ユーザー情報を取得するまで、注文履歴の取得を開始できない
- 注文履歴を取得するまで、ポイント残高の取得を開始できない
- 実際には3つの処理は独立しているので、同時に実行できるはず

### パターン2：並列実行（速い、推奨）

```kotlin
@Service
class UserProfileService(
    private val userApiClient: UserApiClient,
    private val orderApiClient: OrderApiClient,
    private val pointApiClient: PointApiClient
) {
    // ✅ 良い例：並列実行で速い
    suspend fun getUserProfile(userId: Long): UserProfileData = coroutineScope {
        // 3つの外部APIを「同時に」開始
        val userDeferred = async(Dispatchers.IO) {
            userApiClient.getUser(userId)
        }
        
        val ordersDeferred = async(Dispatchers.IO) {
            orderApiClient.getOrders(userId)
        }
        
        val pointsDeferred = async(Dispatchers.IO) {
            pointApiClient.getPoints(userId)
        }
        
        // 全ての結果を待つ（3つとも並列で実行されている）
        val user = userDeferred.await()
        val orders = ordersDeferred.await()
        val points = pointsDeferred.await()
        
        UserProfileData(user, orders, points)
        // 合計：1秒（最も遅い処理の時間）
    }
}
```

**このコードの動き（タイムライン）**：

```text
時刻0秒：3つのasyncを全て開始
  ├─ async1: ユーザー情報取得開始
  ├─ async2: 注文履歴取得開始
  └─ async3: ポイント残高取得開始

時刻1秒：3つとも完了（並列実行されたから）
  ├─ async1: 完了 ✓
  ├─ async2: 完了 ✓
  └─ async3: 完了 ✓

await()で結果を受け取る（既に完了しているので即座に返る）
```

### パターン3：依存関係がある場合

「ユーザー情報を取得してから、そのユーザーの設定に基づいて別の処理を行う」のように、依存関係がある場合はどうするか？

```kotlin
@Service
class RecommendationService(
    private val userApiClient: UserApiClient,
    private val productApiClient: ProductApiClient,
    private val categoryApiClient: CategoryApiClient
) {
    // 依存関係がある場合の処理
    suspend fun getRecommendations(userId: Long): RecommendationData = coroutineScope {
        // まずユーザー情報を取得（これは待つ必要がある）
        val user = withContext(Dispatchers.IO) {
            userApiClient.getUser(userId)
        }
        
        // ユーザー情報から興味のあるカテゴリーを取得
        val favoriteCategories = user.preferences.favoriteCategories
        
        // ここから並列実行できる処理
        val productsDeferred = async(Dispatchers.IO) {
            productApiClient.getProducts(favoriteCategories)
        }
        
        val categoriesDeferred = async(Dispatchers.IO) {
            categoryApiClient.getCategories(favoriteCategories)
        }
        
        // user.membershipTypeに基づいて処理を分岐
        val premiumContentDeferred = async(Dispatchers.IO) {
            if (user.isPremiumMember) {
                productApiClient.getPremiumProducts()
            } else {
                emptyList()
            }
        }
        
        // 並列実行した結果を待つ
        RecommendationData(
            user = user,
            products = productsDeferred.await(),
            categories = categoriesDeferred.await(),
            premiumContent = premiumContentDeferred.await()
        )
    }
}
```

**ポイント**：

1. 依存関係がある処理は順番に実行（`withContext`を使う）
2. 独立した処理は並列実行（`async`を使う）
3. できるだけ並列実行の部分を多くすると速くなる

### パターン4：エラーハンドリング付き

```kotlin
@Service
class UserProfileService(
    private val userApiClient: UserApiClient,
    private val orderApiClient: OrderApiClient,
    private val pointApiClient: PointApiClient
) {
    // 実践的なエラーハンドリング
    suspend fun getUserProfile(userId: Long): Result<UserProfileData> = coroutineScope {
        try {
            // 並列実行
            val userDeferred = async(Dispatchers.IO) { userApiClient.getUser(userId) }
            val ordersDeferred = async(Dispatchers.IO) { orderApiClient.getOrders(userId) }
            val pointsDeferred = async(Dispatchers.IO) { 
                pointApiClient.getPoints(userId) 
            }
            
            // 結果を取得
            val data = UserProfileData(
                user = userDeferred.await(),
                orders = ordersDeferred.await(),
                points = pointsDeferred.await()
            )
            
            Result.success(data)
            
        } catch (e: HttpClientException) {
            // 外部APIエラー
            Result.failure(Exception("外部サービスとの通信に失敗しました"))
        } catch (e: HttpServerException) {
            // APIサーバーエラー
            Result.failure(Exception("外部サービスでエラーが発生しました: ${e.message}"))
        } catch (e: Exception) {
            // その他のエラー
            Result.failure(Exception("予期しないエラーが発生しました"))
        }
    }
}

@RestController
class UserProfileController(private val service: UserProfileService) {
    
    @GetMapping("/users/{id}/profile")
    suspend fun getUserProfile(@PathVariable id: Long): ResponseEntity<UserProfileData> {
        val result = service.getUserProfile(id)
        
        return when {
            result.isSuccess -> ResponseEntity.ok(result.getOrNull()!!)
            result.isFailure -> {
                val error = result.exceptionOrNull()?.message ?: "エラー"
                ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body(null)
            }
            else -> ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(null)
        }
    }
}
```

### 並列実行の効果を測定

実際にどれくらい速くなるか、タイマーで測ってみましょう：

```kotlin
import kotlin.system.measureTimeMillis

suspend fun comparePerformance() {
    // 逐次実行の時間を測定
    val sequentialTime = measureTimeMillis {
        loadUserScreenSequential("123")
    }
    
    // 並列実行の時間を測定
    val parallelTime = measureTimeMillis {
        loadUserScreen("123")
    }
    
    println("逐次実行: ${sequentialTime}ms")  // 約3000ms
    println("並列実行: ${parallelTime}ms")    // 約1000ms
    println("改善率: ${(sequentialTime.toFloat() / parallelTime * 100).toInt()}%") // 約300%
}
```

### まとめ：並列実行のコツ

並列実行を効果的に使うには、4つのポイントを押さえましょう。

**ポイント1：独立した処理を見つける**

まず、処理間の依存関係を確認します。「Aの結果がなくてもBを実行できる」なら並列化可能です。逆に「Aの結果を使ってBを実行する」なら逐次実行が必要です。

**ポイント2：asyncを使って同時に開始**

独立した処理は`async`で同時に開始します：

```kotlin
val task1 = async { doSomething1() }
val task2 = async { doSomething2() }
```

**ポイント3：await()で結果を受け取る**

全てのタスクを開始した後、`await()`で結果を受け取ります：

```kotlin
val result1 = task1.await()
val result2 = task2.await()
```

**ポイント4：エラーハンドリングを忘れずに**

重要な注意点として、1つのタスクがエラーになると、他のタスクも自動的にキャンセルされます。そのため、try-catchで適切にエラーを処理する必要があります。

## エラーハンドリング

```kotlin
fun loadData() = viewModelScope.launch {
    try {
        val data = fetchData()
        _uiState.value = Success(data)
    } catch (e: IOException) {
        _uiState.value = Error("ネットワークエラー")
    } catch (e: Exception) {
        _uiState.value = Error("予期しないエラー")
    }
}
```

## 構造化された同時実行性

親コルーチンがキャンセルされると、子コルーチンも自動的にキャンセルされます。

```kotlin
val parentJob = launch {
    val child1 = launch {
        repeat(1000) {
            delay(100)
            println("Child 1: $it")
        }
    }
    
    val child2 = launch {
        repeat(1000) {
            delay(100)
            println("Child 2: $it")
        }
    }
}

delay(500)
parentJob.cancel() // child1とchild2も自動的にキャンセルされる
```

## よくある間違いと落とし穴

コルーチンを学び始めると、誰もが一度は踏んでしまう「よくある間違い」があります。これらは一見動いているように見えても、実は深刻な問題を引き起こす可能性があります。1つずつ見ていきましょう。

### 1. GlobalScopeの使用（最も危険な間違い）

#### 何が問題なのか？

初心者が最もやりがちな間違いが `GlobalScope` の使用です。「とりあえず動けばいいや」と思って使ってしまうと、後で大変なことになります。

#### 具体的な問題シナリオ

想像してください。以下のようなWebアプリケーションの動作を考えてみましょう：

1. クライアントからAPI /users/{id}にリクエストが来る
2. データ取得処理が始まる（10秒かかる）
3. 3秒後、クライアントがタイムアウトして接続を切断
4. リクエストは終了
5. でも、バックグラウンドではまだデータ取得が続いている...（あと7秒）
6. データ取得完了！
7. **レスポンスを返そうとする → エラー！**（接続がもう存在しない）

```kotlin
@Service
class UserService {
    // ❌ 最悪な例：絶対にやってはいけない
    fun loadUserDataInBackground(userId: Long) {
        GlobalScope.launch {
            // この処理は10秒かかる
            val data = externalApiClient.getUser(userId)
            
            // ここで問題発生！
            // リクエストが終了していても処理が続いている
            // → リソースの無駄遣い、メモリリーク
            logger.info("Data loaded: ${data.name}")
        }
    }
}
```

#### 何が起きているのか？

- `GlobalScope` はアプリケーション全体のライフサイクルに紐づいている
- つまり、リクエストが終了しても、アプリケーションが終了するまでコルーチンは動き続ける
- リクエストが終了した後も処理が継続される → **リソースリーク**
- 不要な処理が蓄積されていく → **パフォーマンス劣化**

#### 正しい方法

適切なスコープを使って、リクエストやサービスのライフサイクルに合わせたコルーチン管理を行います。

**パターン1: suspend関数を使う（推奨）**

```kotlin
@Service
class UserService(private val externalApiClient: ExternalApiClient) {
    
    // ✅ 良い例：suspend関数を使う
    suspend fun loadUserData(userId: Long): User {
        // リクエストがキャンセルされたら、このコルーチンも自動的にキャンセル
        return withContext(Dispatchers.IO) {
            externalApiClient.getUser(userId)
        }
    }
}

@RestController
class UserController(private val userService: UserService) {
    
    @GetMapping("/users/{id}")
    suspend fun getUser(@PathVariable id: Long): User {
        // リクエストスコープで管理される
        return try {
            userService.loadUserData(id)
        } catch (e: CancellationException) {
            // リクエストがキャンセルされた時の処理
            logger.debug("Request was cancelled")
            throw ResponseStatusException(HttpStatus.REQUEST_TIMEOUT)
        }
    }
}
```

**パターン2: サービス専用のCoroutineScopeを作成する場合**

```kotlin
@Service
class BackgroundJobService : DisposableBean {
    // サービス専用のスコープ
    private val serviceScope = CoroutineScope(SupervisorJob() + Dispatchers.Default)
    
    fun processInBackground(userId: Long) {
        serviceScope.launch {
            try {
                // サービスが破棄されたら、このコルーチンも自動的にキャンセル
                val data = externalApiClient.getUser(userId)
                logger.info("Processed: ${data.name}")
            } catch (e: CancellationException) {
                logger.debug("Service scope was cancelled")
            }
        }
    }
    
    // サービス終了時にすべてのコルーチンをキャンセル
    override fun destroy() {
        serviceScope.cancel()
    }
}
```

#### なぜこれで問題が解決する？

- suspend関数は呼び出し元のスコープに紐づいている
- リクエストが終了すると、自動的にコルーチンもキャンセルされる
- 不要な処理は即座に止まる → リソース節約、メモリリーク防止

### 2. suspend関数の誤用：suspendを付ければいいと思っている

#### 何が問題なのか？

「コルーチンを使うなら全部suspendを付けておけばいいや」と思っていませんか？実は、不要な場所に `suspend` を付けると、コードが読みにくくなり、パフォーマンスも悪化します。

#### 間違った例

```kotlin
// ❌ 悪い例：suspendの意味がない
suspend fun calculate(a: Int, b: Int): Int {
    return a + b // 瞬時に終わる計算にsuspendは不要
}

suspend fun getCurrentTime(): Long {
    return System.currentTimeMillis() // これも瞬時に終わる
}

suspend fun formatText(text: String): String {
    return text.uppercase() // 文字列操作にsuspendは不要
}
```

#### なぜダメなのか？

1. **意味がない**: 一時停止する処理が何もないのに `suspend` を付けている
2. **制約が増える**: suspend関数は他のsuspend関数やコルーチン内からしか呼べない
3. **誤解を招く**: コードを読む人が「これは時間のかかる処理だな」と誤解する
4. **パフォーマンス**: わずかながらオーバーヘッドがある

#### 正しい使い方

`suspend` を付けるべきなのは、**実際に一時停止する処理がある場合のみ**です。

```kotlin
// ✅ 良い例：実際に一時停止する処理
suspend fun fetchFromNetwork(): User {
    // withContextで別スレッドに切り替える（一時停止ポイント）
    return withContext(Dispatchers.IO) {
        apiService.getUser() // ネットワーク通信（一時停止ポイント）
    }
}

suspend fun saveToDatabase(user: User) {
    // データベース操作（一時停止ポイント）
    withContext(Dispatchers.IO) {
        database.insertUser(user)
    }
}

suspend fun processWithDelay() {
    delay(1000) // 明示的な一時停止
    println("1秒後に実行")
}

// ✅ suspendが不要な例：普通の関数でOK
fun calculate(a: Int, b: Int): Int {
    return a + b // 瞬時に終わるのでsuspendは不要
}

fun formatUserName(user: User): String {
    return "${user.firstName} ${user.lastName}"
}
```

#### 判断基準

`suspend`を付けるべきかどうか迷ったら、以下のチェックリストを使いましょう。4つのうちいずれかに当てはまる場合のみ`suspend`を付けます：

1. `delay()`を使う場合
2. 他のsuspend関数を呼ぶ場合
3. `withContext()`でスレッドを切り替える場合
4. ネットワーク通信、データベースアクセス、ファイルI/Oを行う場合

これらに当てはまらない処理（単純な計算や文字列操作など）には、`suspend`は不要です。

### 3. withContextの乱用：二重に使ってしまう

#### 何が問題なのか？

`withContext` を理解し始めると、「念のため」と思って二重、三重に使ってしまうことがあります。これは不要なオーバーヘッドを生み、コードも読みにくくなります。

#### 間違った例

```kotlin
// ❌ 悪い例：withContextの二重使用
suspend fun loadUserData(): User {
    return withContext(Dispatchers.IO) {
        // 既にIOスレッドなのに、さらにwithContextを使っている
        val userData = withContext(Dispatchers.IO) {
            apiService.getUser()
        }
        
        // これも不要
        val profileData = withContext(Dispatchers.IO) {
            apiService.getProfile()
        }
        
        User(userData, profileData)
    }
}
```

#### なぜダメなのか？

- 既に `Dispatchers.IO` の中にいるのに、さらに切り替えようとしている
- スレッドの切り替えにはコストがかかる（わずかだが積み重なる）
- コードが読みにくくなる

#### 正しい使い方

```kotlin
// ✅ 良い例：withContextは一度だけ
suspend fun loadUserData(): User {
    return withContext(Dispatchers.IO) {
        // この中は全てIOスレッドで実行される
        val userData = apiService.getUser()
        val profileData = apiService.getProfile()
        User(userData, profileData)
    }
}

// ✅ または、関数を分けてそれぞれにwithContextを使う
suspend fun getUserData(): UserData {
    return withContext(Dispatchers.IO) {
        apiService.getUser()
    }
}

suspend fun getProfileData(): ProfileData {
    return withContext(Dispatchers.IO) {
        apiService.getProfile()
    }
}

// 呼び出し側
suspend fun loadUserData(): User {
    val userData = getUserData() // 内部でIOスレッドに切り替え
    val profileData = getProfileData() // 内部でIOスレッドに切り替え
    return User(userData, profileData)
}
```

#### 実用的なパターン

異なるディスパッチャーが必要な場合のみ、複数の `withContext` を使います：

```kotlin
// ✅ 良い例：異なるディスパッチャーを使い分ける
suspend fun processData(): Result {
    // 1. IOスレッドでデータ取得
    val rawData = withContext(Dispatchers.IO) {
        apiService.getData()
    }
    
    // 2. Defaultスレッドで重い計算処理
    val processedData = withContext(Dispatchers.Default) {
        complexCalculation(rawData) // CPU負荷が高い
    }
    
    // 3. IOスレッドでデータベースに保存
    withContext(Dispatchers.IO) {
        database.save(processedData)
    }
    
    return Result.Success
}
```

### 4. runBlockingの誤用：本番コードで使ってしまう

#### 何が問題なのか？

`runBlocking` は「ブロッキング」という名前の通り、スレッドを完全に止めてしまいます。特にAndroidのメインスレッドで使うと、アプリが固まります。

#### 間違った例とその影響

```kotlin
// ❌ 最悪な例：UIスレッドをブロック
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        // ここでメインスレッドが3秒間完全に止まる
        runBlocking {
            delay(3000)
            loadData()
        }
        
        // スレッドプールが枯渇し、他のリクエストも処理できなくなる
        // 最悪の場合、アプリケーション全体が応答しなくなる
    }
}
```

#### ユーザー体験への影響

1. クライアントがAPIリクエストを送信
2. サーバーがリクエストを受け取る
3. **3秒間、そのスレッドがブロックされる**（他のリクエストを処理できない）
4. スレッドプールが枯渇すると、新しいリクエストが待たされる
5. 最悪の場合、タイムアウトエラーやサーバーダウンにつながる

#### 正しい方法

suspend関数を使って、スレッドをブロックせずに非同期処理を行います。

```kotlin
@RestController
class OrderController(private val orderService: OrderService) {
    
    @PostMapping("/orders")
    suspend fun createOrder(@RequestBody request: OrderRequest): Order {
        // ✅ 良い例：スレッドをブロックしない
        // suspend関数なので、待機中は他のリクエストを処理できる
        delay(3000) // シミュレーション
        return orderService.createOrder(request)
    }
}

@Service
class OrderService {
    suspend fun createOrder(request: OrderRequest): Order {
        // データベース処理など
        return withContext(Dispatchers.IO) {
            // I/O処理中もスレッドをブロックしない
            orderRepository.save(request.toEntity())
        }
    }
}
```

#### runBlockingが適切な場面

`runBlocking` が適切なのは、以下の場合のみです：

**1. テストコード**

```kotlin
// ✅ テストでの使用は問題なし
@Test
fun testDataLoading() = runBlocking {
    // テストスレッドをブロックするだけなので問題ない
    val repository = UserRepository()
    val user = repository.getUser(123L)
    assertEquals("太郎", user.name)
}
```

**2. main関数（学習用コード）**

```kotlin
// ✅ main関数での使用（サンプルコード）
fun main() = runBlocking {
    println("プログラム開始")
    delay(1000)
    println("1秒後")
}
```

**3. 既存の同期的なコードとの橋渡し（非推奨だが、どうしても必要な場合）**

```kotlin
// △ 既存のレガシーコードを変更できない場合（最後の手段）
class LegacyService {
    // この関数は同期的に結果を返す必要がある（変更不可）
    fun getUserSync(): User = runBlocking {
        // 新しいコルーチンベースのAPIを呼ぶ
        userRepository.getUser()
    }
}
// 注：可能な限り、呼び出し側もsuspend関数に変更すべき
```

#### まとめ：runBlockingのルール

`runBlocking`の使用ルールをまとめると、以下のようになります。

本番のアプリケーションコード（Activity、Fragment、ViewModel）やメインスレッドでは使わないでください。これらの場所で使うと、アプリが固まったりスレッドプールが枯渇したりします。

一方、テストコードやmain関数（学習用のサンプルコード）では使ってOKです。これらの場所では、スレッドをブロックしても問題ありません。

レガシーコードとの橋渡しで、どうしても必要な場合のみ使うことができますが、これは最終手段と考えてください。可能な限り、呼び出し側もsuspend関数に変更するのが望ましいです。

## コルーチンのキャンセル

コルーチンは協調的なキャンセルをサポートしています。

### キャンセル可能なコルーチン

```kotlin
val job = launch {
    repeat(1000) { i ->
        println("job: I'm sleeping $i ...")
        delay(500)
    }
}

delay(1300) // 1.3秒待つ
println("main: I'm tired of waiting!")
job.cancel() // ジョブをキャンセル
job.join() // キャンセル完了を待つ
println("main: Now I can quit.")
```

### キャンセルをチェックする

CPU負荷の高い処理では `isActive` でキャンセルをチェック：

```kotlin
val job = launch(Dispatchers.Default) {
    var nextPrintTime = System.currentTimeMillis()
    var i = 0
    while (isActive) { // キャンセルをチェック
        if (System.currentTimeMillis() >= nextPrintTime) {
            println("job: I'm sleeping ${i++} ...")
            nextPrintTime += 500
        }
    }
}

delay(1300)
job.cancelAndJoin() // cancel + join
```

### finallyブロックでクリーンアップ

```kotlin
val job = launch {
    try {
        repeat(1000) { i ->
            println("job: I'm sleeping $i ...")
            delay(500)
        }
    } finally {
        println("job: I'm running finally")
        // リソースのクリーンアップ処理
    }
}

delay(1300)
job.cancelAndJoin()
```

## タイムアウト

一定時間で処理を打ち切りたい場合：

```kotlin
// TimeoutCancellationExceptionをスロー
withTimeout(1300) {
    repeat(1000) { i ->
        println("I'm sleeping $i ...")
        delay(500)
    }
}

// nullを返す（例外をスローしない）
val result = withTimeoutOrNull(1300) {
    repeat(1000) { i ->
        println("I'm sleeping $i ...")
        delay(500)
    }
    "Done"
}
println(result) // null
```

## Flow：非同期データストリーム

Flowは複数の値を順次返す非同期ストリームです。

### 基本的なFlow

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // 非同期に値を生成
        emit(i) // 値を送出
    }
}

fun main() = runBlocking {
    simple().collect { value ->
        println(value)
    }
}
// 出力：1, 2, 3（それぞれ100ms間隔）
```

### Flowの演算子

```kotlin
fun main() = runBlocking {
    (1..5).asFlow()
        .filter { it % 2 == 0 } // 偶数のみ
        .map { "Value: $it" }    // 変換
        .collect { println(it) }
}
// 出力：
// Value: 2
// Value: 4
```

### Flowのコンテキスト

```kotlin
fun simple(): Flow<Int> = flow {
    println("Started flow on ${Thread.currentThread().name}")
    for (i in 1..3) {
        emit(i)
    }
}.flowOn(Dispatchers.Default) // Flowの実行コンテキストを変更

fun main() = runBlocking {
    simple()
        .collect { value ->
            println("Collected $value on ${Thread.currentThread().name}")
        }
}
```

### StateFlowとSharedFlow

UIの状態管理に便利：

```kotlin
class ViewModel {
    // 状態を保持（最新の値を常に持つ）
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    // イベントストリーム（最新の値を保持しない）
    private val _events = MutableSharedFlow<Event>()
    val events: SharedFlow<Event> = _events.asSharedFlow()
    
    fun loadData() {
        viewModelScope.launch {
            try {
                val data = fetchData()
                _uiState.value = UiState.Success(data)
            } catch (e: Exception) {
                _events.emit(Event.ShowError(e.message))
            }
        }
    }
}
```

## チャネル：コルーチン間の通信

チャネルは値のストリームを送受信するための手段です。

```kotlin
val channel = Channel<Int>()

launch {
    for (x in 1..5) {
        channel.send(x * x) // 値を送信
    }
    channel.close() // 送信完了
}

launch {
    for (y in channel) { // 値を受信
        println(y)
    }
    println("Done!")
}
```

### プロデューサー・コンシューマーパターン

```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) {
        send(x++) // 無限に数値を生成
        delay(100)
    }
}

fun CoroutineScope.square(numbers: ReceiveChannel<Int>) = produce<Int> {
    for (x in numbers) {
        send(x * x)
    }
}

fun main() = runBlocking {
    val numbers = produceNumbers()
    val squares = square(numbers)
    
    repeat(5) {
        println(squares.receive())
    }
    
    coroutineContext.cancelChildren() // 子コルーチンをキャンセル
}
```

## コルーチンコンテキストとディスパッチャー

### コンテキストの要素

コルーチンコンテキストは複数の要素を持ちます：

```kotlin
launch(Dispatchers.Default + CoroutineName("MyCoroutine")) {
    println("Running in ${coroutineContext[CoroutineName]}")
    println("Running on ${Thread.currentThread().name}")
}
```

### コンテキストの継承

```kotlin
val scope = CoroutineScope(Job() + Dispatchers.Main)

scope.launch {
    println("Parent context: ${coroutineContext[Job]}")
    
    launch {
        // 親のコンテキストを継承
        println("Child context: ${coroutineContext[Job]}")
    }
}
```

## 例外ハンドリングの詳細

### CoroutineExceptionHandler

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    println("Caught $exception")
}

val scope = CoroutineScope(Job() + handler)

scope.launch {
    throw AssertionError() // ハンドラーでキャッチされる
}

scope.launch {
    // async内の例外はawaitで発生する
    val deferred = async {
        throw ArithmeticException()
    }
    deferred.await() // ここで例外がスロー
}
```

### SupervisorJob

子コルーチンの失敗が親に伝播しないようにする：

```kotlin
val supervisor = SupervisorJob()

with(CoroutineScope(coroutineContext + supervisor)) {
    val child1 = launch {
        println("Child 1: starting")
        delay(100)
        throw Exception("Child 1 failed!")
    }
    
    val child2 = launch {
        println("Child 2: starting")
        delay(200)
        println("Child 2: completed")
    }
}
// child1が失敗してもchild2は継続する
```

### supervisorScope

特定のスコープ内だけSupervisorの振る舞いにする：

```kotlin
suspend fun doWork() {
    supervisorScope {
        val child1 = launch {
            println("Child 1 failed")
            throw Exception()
        }
        
        val child2 = launch {
            delay(1000)
            println("Child 2 completed")
        }
    }
}
```

## Select式：複数のコルーチンを待つ

複数の suspend 関数の中で最初に完了したものを選択：

```kotlin
suspend fun selectFastestApi(): String {
    val api1 = async { fetchFromApi1() }
    val api2 = async { fetchFromApi2() }
    
    return select {
        api1.onAwait { "API1: $it" }
        api2.onAwait { "API2: $it" }
    }
}
```

## デバッグのコツ

### コルーチン名を付ける

```kotlin
launch(CoroutineName("データロード")) {
    println("${coroutineContext[CoroutineName]?.name}を実行中")
}
```

### ログ出力でスレッド名を確認

```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() = runBlocking {
    launch(Dispatchers.Default) {
        log("実行中")
    }
}
```

## まとめ

Kotlinコルーチンの要点：

1. **軽量**: 大量の並行処理が可能
2. **一時停止可能**: `suspend` 関数でスレッドをブロックせず待機
3. **構造化**: スコープによるライフサイクル管理
4. **シンプル**: 非同期処理を同期的なコードのように書ける
5. **キャンセル可能**: 協調的なキャンセルメカニズム
6. **Flow**: 複数の値を扱う非同期ストリーム
7. **チャネル**: コルーチン間の通信手段
8. **例外処理**: 構造化された例外ハンドリング

コルーチンは最初は難しく感じるかもしれませんが、基本を理解すれば非常に強力なツールです。少しずつ使ってみて、慣れていきましょう！

## 参考資料

- [Kotlin Coroutines 公式ドキュメント](https://kotlinlang.org/docs/coroutines-guide.html)
- [Android Developers: Kotlin coroutines on Android](https://developer.android.com/kotlin/coroutines)
