---
title: "Kotlin Coroutineを噛み砕いて理解する"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kotlin", "coroutine", "android"]
published: false
---

# はじめに

Kotlinのコルーチン（Coroutine）は、非同期処理を簡潔に書けるKotlinの強力な機能です。公式ドキュメントを読んでも「よくわからない...」と感じる方も多いと思います。この記事では、コルーチンの基本概念を身近な例を使って噛み砕いて説明します。

## コルーチンって何？なぜ必要なの？

### 問題：時間のかかる処理でアプリが固まる

例えば、ネットワークからデータを取得する処理を考えてみましょう。普通に書くとこうなります：

```kotlin
fun loadUserData() {
    val data = fetchFromNetwork() // 3秒かかる
    showData(data)
}
```

この問題点は、`fetchFromNetwork()` の3秒間、アプリが完全にフリーズしてしまうことです。ユーザーは何も操作できず、「アプリが壊れた？」と思ってしまいます。

### 従来の解決策：スレッド

従来はスレッドを使って別の処理フローで実行していました：

```kotlin
// 普通のスレッド（重い）
thread {
    val data = fetchFromNetwork() // 別スレッドで実行
    // でも、ここでUIを更新できない！（メインスレッドじゃないから）
    runOnUiThread {
        showData(data) // UIスレッドに戻る必要がある
    }
}
```

スレッドには以下の問題があります：
- **重い**: スレッド1つで約1MB〜2MBのメモリを消費
- **数に限界**: せいぜい数百個しか作れない
- **切り替えコスト**: スレッド間の切り替えは時間がかかる
- **複雑**: スレッド間の通信やUIスレッドへの戻し方が面倒

### コルーチンの解決策

コルーチンを使うとこうなります：

```kotlin
// コルーチン（軽い＆シンプル）
// 注：実際のアプリではviewModelScopeやlifecycleScopeを使う
viewModelScope.launch {
    val data = fetchFromNetwork() // 一時停止して待つ
    showData(data) // 自動的に適切なスレッドで実行
}
```

**コルーチンを一言で表すと**：「途中で一時停止・再開できる、めちゃくちゃ軽量な処理の単位」です。

### コルーチンの3つの魔法

1. **超軽量**
   - 1つあたり数十バイト程度
   - 数万〜数十万個を同時に動かせる
   - 例：10万件の処理を並行実行しても問題なし

2. **一時停止機能**
   - `delay(1000)` で1秒待つ間、スレッドを占有しない
   - その間に他のコルーチンが動ける
   - レストランの例：注文を受けて料理ができるまで待つ間、他のお客さんの対応ができる

3. **構造化された並行性**
   - 親コルーチンがキャンセルされると、子も自動的にキャンセル
   - メモリリークの心配が減る
   - 画面を閉じたら、その画面で動いていた処理も自動的に止まる

### 具体的な比較

同じ「1秒待つ」処理でも、内部では全く違うことが起きています：

```kotlin
// Thread.sleep(1000)の場合
// → スレッドを1秒間「占有」= その間そのスレッドは何もできない
// → 1000個のスレッドでやったらメモリが1〜2GB必要

thread {
    Thread.sleep(1000) // このスレッドは1秒間寝る
    println("完了")
}

// delay(1000)の場合
// → スレッドは「解放」= その間他のコルーチンが使える
// → 10万個のコルーチンでやってもメモリは数MB程度

launch {
    delay(1000) // スレッドを手放して1秒後に再開
    println("完了")
}
```

## コルーチンの基本構成要素

コルーチンを使いこなすには、3つの基本要素を理解する必要があります。順番に見ていきましょう。

### 1. suspend関数：「一時停止できる関数」の目印

`suspend` キーワードは「この関数は途中で一時停止するかもしれない」という目印です。

#### なぜsuspendが必要なのか？

普通の関数は「開始したら終わるまで一気に実行」されます。でも、ネットワーク通信のように「結果が返ってくるまで待つ」処理では、その待ち時間にスレッドを占有するのはもったいないですよね。

```kotlin
// 普通の関数（一気に実行される）
fun add(a: Int, b: Int): Int {
    return a + b // 瞬時に終わる
}

// suspend関数（途中で一時停止できる）
suspend fun fetchUserData(): User {
    delay(1000) // ← ここで一時停止（スレッドは解放される）
    return User("太郎") // 1秒後に再開してここが実行される
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

// ✅ これもOK（実際のアプリの場合）
class MyViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch { // コルーチンの中
            fetchUserData() // コルーチンの中ならOK
        }
    }
}
```

**なぜこのルールがあるの？**
suspend関数は「一時停止」という特殊な処理をします。普通の関数はそれを理解できないので、コルーチンという特別な環境の中でしか実行できないのです。

#### 実用例：ネットワーク処理

```kotlin
// ユーザー情報を取得する（suspend関数）
suspend fun fetchUser(id: String): User {
    // withContextで「IO処理用のスレッド」に切り替え
    return withContext(Dispatchers.IO) {
        // ネットワークAPI呼び出し（時間がかかる）
        apiService.getUser(id) // ここで一時停止
    }
}

// 投稿一覧を取得する（suspend関数）
suspend fun fetchPosts(userId: String): List<Post> {
    return withContext(Dispatchers.IO) {
        apiService.getPosts(userId) // ここで一時停止
    }
}

// これらを使う（ViewModelの場合）
class UserViewModel : ViewModel() {
    fun loadData() {
        // viewModelScopeを使う（推奨）
        viewModelScope.launch {
            val user = fetchUser("123") // 一時停止して待つ
            val posts = fetchPosts(user.id) // 一時停止して待つ
            showUserAndPosts(user, posts) // データを表示
        }
    }
}
```

この例のポイント：
- `fetchUser`と`fetchPosts`は両方とも suspend関数
- 呼び出し側から見ると、普通の関数のように「順番に」書ける
- でも内部では、待っている間スレッドを解放して他の処理ができる

### 2. コルーチンビルダー：コルーチンを「起動」する関数

suspend関数を呼ぶには、まずコルーチンを起動する必要があります。そのための関数がコルーチンビルダーです。主に3つあります。

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

**launchの特徴**：
- 戻り値は `Job`（ジョブ＝処理の管理チケット）
- 処理の結果を返さない（返せない）
- 「ログ出力」「データ保存」など、結果が不要な処理に最適

**Jobで何ができるか**：

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

**runBlockingの特徴と使用上の注意**：
- 中のコルーチンが全て終わるまで、呼び出したスレッドをブロック（待機）する
- **⚠️ 本番のアプリコードでは使わない**（スレッドをブロックするため）
- **使うべき場面**：
  - `main`関数（学習用のサンプルコード）
  - テストコード
  - 既存の同期的なコードとの橋渡し

**なぜ本番コードで避けるべき？**

`runBlocking`は名前の通り、スレッドを「ブロック」します。特にAndroidアプリでメインスレッドで`runBlocking`を使うと、UIがフリーズする原因になります。

```kotlin
// ❌ 悪い例：メインスレッドをブロックする
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        runBlocking { // メインスレッドが3秒間フリーズ！
            delay(3000)
            loadData()
        }
    }
}

// ✅ 良い例：適切なスコープを使う
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        lifecycleScope.launch { // メインスレッドをブロックしない
            delay(3000)
            loadData()
        }
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

### 3. CoroutineScope：コルーチンの「実行範囲」を定義

CoroutineScopeは「コルーチンの寿命を管理する枠組み」です。これが理解できると、メモリリークを防げます。

#### なぜScopeが必要なのか？

例えば、Androidアプリで画面を開いた時にデータ取得を開始したとします：

```kotlin
// ❌ 悪い例
class UserActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        GlobalScope.launch {
            val data = fetchUserData() // 10秒かかる
            showData(data) // データを表示
        }
    }
}
```

問題：ユーザーが3秒後に画面を閉じたら？
- コルーチンはまだ動いている（あと7秒残ってる）
- 画面はもう無い
- `showData(data)`を呼ぶとクラッシュ！

**解決策：CoroutineScopeで寿命を管理**

```kotlin
// ✅ 良い例
class UserActivity : AppCompatActivity(), CoroutineScope {
    // このActivityのためのJobを作る
    private val job = Job()
    
    // CoroutineScopeの実装（Mainスレッドで実行）
    override val coroutineContext = Dispatchers.Main + job
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // このActivityのスコープでコルーチンを起動
        launch {
            val data = fetchUserData() // 10秒かかる
            showData(data)
        }
    }
    
    override fun onDestroy() {
        super.onDestroy()
        job.cancel() // Activityが破棄される時に全コルーチンをキャンセル
    }
}
```

これで、画面を閉じると自動的にコルーチンもキャンセルされます。

#### ViewModelScopeとLifecycleScopeの便利さ

Androidでは、もっと簡単な方法が用意されています：

```kotlin
// ViewModelの場合
class UserViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch { // ViewModelが破棄されると自動キャンセル
            val data = fetchUserData()
            _uiState.value = data
        }
    }
}

// Activity/Fragmentの場合
class UserFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        lifecycleScope.launch { // Fragmentのライフサイクルに連動
            val data = fetchUserData()
            showData(data)
        }
    }
}
```

**覚えておくべきこと**：
- `GlobalScope`：**⚠️ 本番コードでは使用禁止**（理由は後述）
- `viewModelScope`：ViewModelが破棄されたら自動キャンセル（推奨）
- `lifecycleScope`：画面のライフサイクルに連動して自動キャンセル（推奨）
- 独自のScope：自分で`Job()`を作って管理する

#### GlobalScopeを使ってはいけない理由

`GlobalScope`は「アプリケーションのライフサイクルと同じ寿命を持つスコープ」です。つまり、アプリが終了するまでキャンセルされません。

**問題点**：

1. **メモリリークの原因**
```kotlin
// ❌ 悪い例
class UserFragment : Fragment() {
    fun loadData() {
        GlobalScope.launch {
            val data = fetchUserData() // 10秒かかる
            updateUI(data) // Fragmentが既に破棄されていたらクラッシュ
        }
    }
}
// ユーザーが画面を閉じても、コルーチンは動き続ける
// Fragmentへの参照が残り、メモリリークが発生
```

2. **リソースの無駄遣い**
```kotlin
// ❌ 悪い例
GlobalScope.launch {
    while (true) {
        val data = fetchData() // 1秒ごとにデータ取得
        updateUI(data)
        delay(1000)
    }
}
// ユーザーが画面を閉じても、永遠にAPIを叩き続ける
// バッテリーとネットワーク帯域の無駄
```

3. **テストが困難**
```kotlin
// GlobalScopeはテスト時にコントロールできない
// テストが終了してもコルーチンが動き続ける可能性がある
```

**正しい代替案**：

```kotlin
// ✅ ViewModelの場合
class UserViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch {
            // ViewModelが破棄されたら自動的にキャンセル
            val data = fetchUserData()
            _uiState.value = data
        }
    }
}

// ✅ Fragment/Activityの場合
class UserFragment : Fragment() {
    fun loadData() {
        lifecycleScope.launch {
            // Fragmentのライフサイクルに連動
            val data = fetchUserData()
            updateUI(data)
        }
    }
}

// ✅ カスタムクラスの場合
class DataRepository {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    
    fun loadData() {
        scope.launch {
            val data = fetchUserData()
            processData(data)
        }
    }
    
    fun cleanup() {
        scope.cancel() // 明示的にキャンセル
    }
}
```

**GlobalScopeの唯一の適切な使用例**：

アプリ全体のライフサイクルで動き続けるべき処理（非常に稀）：

```kotlin
// アプリケーションクラスでの初期化処理など
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        // これは例外的にOK（アプリの起動時にログを送信など）
        GlobalScope.launch(Dispatchers.IO) {
            sendStartupLogs()
        }
    }
}
```

ただし、これも実際には `applicationScope` など、より適切なスコープを作成して使うべきです。

## ディスパッチャー：「どこで」実行するかを制御

ディスパッチャー（Dispatcher）は「このコルーチンをどのスレッドで実行するか」を決める仕組みです。

### なぜディスパッチャーが必要？

Androidアプリを例に考えましょう：
- **UIの更新**：必ずメインスレッドで行う必要がある
- **ネットワーク通信**：メインスレッドでやるとアプリが固まる
- **重い計算**：メインスレッドでやるとアプリがカクつく

つまり、処理内容によって「適切なスレッド」が違うんです。

### 4つの主要なディスパッチャー

#### 1. Dispatchers.Main（UIスレッド）

UIの更新など、メインスレッドで実行すべき処理用です。

```kotlin
launch(Dispatchers.Main) {
    // UIスレッドで実行される
    progressBar.visibility = View.VISIBLE
    textView.text = "読込中..."
}
```

**使うべき場面**：
- View（ボタン、テキストなど）の更新
- Toastやダイアログの表示
- UIに関わる全ての操作

#### 2. Dispatchers.IO（入出力処理用）

ネットワーク通信、ファイル読み書き、データベース操作用です。

```kotlin
suspend fun loadUserFromNetwork(): User {
    return withContext(Dispatchers.IO) {
        // ネットワークスレッドで実行される
        apiService.getUser()
    }
}
```

**使うべき場面**：
- REST APIの呼び出し
- ファイルの読み書き
- データベースのクエリ
- SharedPreferencesの読み書き

**特徴**：
- スレッドプールで管理されている（最大64個まで）
- I/O処理は「待ち時間」が多いので、多くのコルーチンで共有できる

#### 3. Dispatchers.Default（CPU処理用）

計算処理、画像加工、JSONパースなど、CPU負荷が高い処理用です。

```kotlin
suspend fun processLargeData(data: List<Int>): List<Int> {
    return withContext(Dispatchers.Default) {
        // CPUを使う処理
        data.map { it * it }.filter { it > 100 }.sorted()
    }
}
```

**使うべき場面**：
- 大量のデータの並べ替え
- 画像のリサイズや加工
- JSONのパース
- 複雑な計算

**特徴**：
- CPUのコア数と同じだけスレッドを用意（例：4コアなら4スレッド）
- CPU密集型の処理に最適化されている

#### 4. Dispatchers.Unconfined（特殊用途）

最初は呼び出したスレッドで実行され、一時停止後は再開したスレッドで実行されます。

```kotlin
launch(Dispatchers.Unconfined) {
    println("1: ${Thread.currentThread().name}") // メインスレッド
    delay(100)
    println("2: ${Thread.currentThread().name}") // 別のスレッド
}
```

**注意**：特殊な用途以外では使わないでください。

### ディスパッチャーの切り替え：withContext

`withContext`を使うと、一時的に別のディスパッチャーに切り替えられます。

```kotlin
// 実用的な例：画面でのデータ読み込み
fun loadData() = viewModelScope.launch(Dispatchers.Main) {
    // メインスレッドで開始
    showLoading(true)
    
    try {
        // IOスレッドに切り替えてネットワーク通信
        val user = withContext(Dispatchers.IO) {
            apiService.getUser()
        }
        
        // 自動的にメインスレッドに戻る
        showUser(user) // UIを更新
        
        // 再びIOスレッドに切り替えてデータベースに保存
        withContext(Dispatchers.IO) {
            database.saveUser(user)
        }
        
        // 自動的にメインスレッドに戻る
        showMessage("保存しました")
        
    } finally {
        // メインスレッド
        showLoading(false)
    }
}
```

**withContextの便利なポイント**：
1. 処理が終わると自動的に元のディスパッチャーに戻る
2. コードが読みやすい（どこで何が実行されるか明確）
3. エラーが起きても正しくメインスレッドに戻る

### 実例：複数APIの並列呼び出し

```kotlin
suspend fun loadDashboardData(): DashboardData = withContext(Dispatchers.Main) {
    showLoading(true)
    
    try {
        // 3つのAPIを同時に呼び出し
        val userData = async(Dispatchers.IO) { apiService.getUser() }
        val postsData = async(Dispatchers.IO) { apiService.getPosts() }
        val notificationsData = async(Dispatchers.IO) { apiService.getNotifications() }
        
        // 全ての結果を待つ（並列実行なので速い）
        val user = userData.await()
        val posts = postsData.await()
        val notifications = notificationsData.await()
        
        // メインスレッドに自動的に戻ってUIを更新
        DashboardData(user, posts, notifications)
        
    } finally {
        showLoading(false)
    }
}
```

### ディスパッチャーの選び方フローチャート

```
UIを触る？
├─ Yes → Dispatchers.Main
└─ No → 時間のかかる処理？
    ├─ Yes → どんな処理？
    │   ├─ ネットワーク/DB/ファイル → Dispatchers.IO
    │   └─ 計算/データ処理 → Dispatchers.Default
    └─ No → Dispatchers.Main（デフォルト）
```

## 実践例：複数のAPI呼び出しを速くする

ここまでの知識を使って、実際によくある「複数のAPIを呼ぶ」場面を最適化してみましょう。

### シナリオ：ユーザー画面の表示

ユーザー画面を表示するには、以下の3つの情報が必要です：
1. ユーザー情報（1秒かかる）
2. ユーザーの投稿一覧（1秒かかる）
3. フォロワー数（1秒かかる）

### パターン1：逐次実行（遅い、初心者がやりがち）

```kotlin
// ❌ 悪い例：順番に待つので遅い
suspend fun loadUserScreen(userId: String): UserScreenData {
    // 1. ユーザー情報を取得（1秒）
    val user = withContext(Dispatchers.IO) {
        apiService.getUser(userId)
    }
    
    // 2. 投稿を取得（1秒）
    val posts = withContext(Dispatchers.IO) {
        apiService.getPosts(userId)
    }
    
    // 3. フォロワー数を取得（1秒）
    val followerCount = withContext(Dispatchers.IO) {
        apiService.getFollowerCount(userId)
    }
    
    return UserScreenData(user, posts, followerCount)
    // 合計：3秒かかる
}
```

**問題点**：
- ユーザー情報を取得するまで、投稿の取得を開始できない
- 投稿を取得するまで、フォロワー数の取得を開始できない
- 実際には3つの処理は独立しているので、同時に実行できるはず

### パターン2：並列実行（速い、推奨）

```kotlin
// ✅ 良い例：並列実行で速い
suspend fun loadUserScreen(userId: String): UserScreenData = coroutineScope {
    // 3つを「同時に」開始
    val userDeferred = async(Dispatchers.IO) {
        apiService.getUser(userId)
    }
    
    val postsDeferred = async(Dispatchers.IO) {
        apiService.getPosts(userId)
    }
    
    val followerCountDeferred = async(Dispatchers.IO) {
        apiService.getFollowerCount(userId)
    }
    
    // 全ての結果を待つ（3つとも並列で実行されている）
    val user = userDeferred.await()
    val posts = postsDeferred.await()
    val followerCount = followerCountDeferred.await()
    
    UserScreenData(user, posts, followerCount)
    // 合計：1秒（最も遅い処理の時間）
}
```

**このコードの動き（タイムライン）**：

```
時刻0秒：3つのasyncを全て開始
  ├─ async1: ユーザー情報取得開始
  ├─ async2: 投稿取得開始
  └─ async3: フォロワー数取得開始

時刻1秒：3つとも完了（並列実行されたから）
  ├─ async1: 完了 ✓
  ├─ async2: 完了 ✓
  └─ async3: 完了 ✓

await()で結果を受け取る（既に完了しているので即座に返る）
```

### パターン3：依存関係がある場合

「ユーザー情報を取得してから、そのユーザーの投稿を取得する」のように、依存関係がある場合はどうするか？

```kotlin
// 依存関係がある場合の処理
suspend fun loadUserDetailsScreen(userId: String): UserDetailsData = coroutineScope {
    // まずユーザー情報を取得（これは待つ必要がある）
    val user = withContext(Dispatchers.IO) {
        apiService.getUser(userId)
    }
    
    // ユーザー情報が必要な処理（user.settingsを使う）
    val theme = user.settings.theme
    
    // ここから並列実行できる処理
    val postsDeferred = async(Dispatchers.IO) {
        apiService.getPosts(userId)
    }
    
    val friendsDeferred = async(Dispatchers.IO) {
        apiService.getFriends(userId)
    }
    
    // user.premiumStatusに基づいて処理を分岐
    val adsDeferred = async(Dispatchers.IO) {
        if (user.isPremium) {
            emptyList() // プレミアムユーザーは広告なし
        } else {
            apiService.getAds()
        }
    }
    
    // 並列実行した結果を待つ
    UserDetailsData(
        user = user,
        posts = postsDeferred.await(),
        friends = friendsDeferred.await(),
        ads = adsDeferred.await(),
        theme = theme
    )
}
```

**ポイント**：
1. 依存関係がある処理は順番に実行（`withContext`を使う）
2. 独立した処理は並列実行（`async`を使う）
3. できるだけ並列実行の部分を多くすると速くなる

### パターン4：エラーハンドリング付き

```kotlin
// 実践的なエラーハンドリング
suspend fun loadUserScreen(userId: String): Result<UserScreenData> = coroutineScope {
    try {
        // 並列実行
        val userDeferred = async(Dispatchers.IO) { apiService.getUser(userId) }
        val postsDeferred = async(Dispatchers.IO) { apiService.getPosts(userId) }
        val followerCountDeferred = async(Dispatchers.IO) { 
            apiService.getFollowerCount(userId) 
        }
        
        // 結果を取得
        val data = UserScreenData(
            user = userDeferred.await(),
            posts = postsDeferred.await(),
            followerCount = followerCountDeferred.await()
        )
        
        Result.success(data)
        
    } catch (e: IOException) {
        // ネットワークエラー
        Result.failure(Exception("ネットワークに接続できません"))
    } catch (e: HttpException) {
        // APIエラー
        Result.failure(Exception("サーバーエラー: ${e.code()}"))
    } catch (e: Exception) {
        // その他のエラー
        Result.failure(Exception("予期しないエラーが発生しました"))
    }
}

// 使い方
fun loadData() = viewModelScope.launch {
    _uiState.value = UiState.Loading
    
    val result = loadUserScreen(userId)
    
    _uiState.value = when {
        result.isSuccess -> UiState.Success(result.getOrNull()!!)
        result.isFailure -> UiState.Error(result.exceptionOrNull()?.message ?: "エラー")
        else -> UiState.Error("不明なエラー")
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

1. **独立した処理を見つける**
   - 「A の結果がなくても B を実行できる」なら並列化可能
   - 「A の結果を使って B を実行する」なら逐次実行が必要

2. **async を使って同時に開始**
   ```kotlin
   val task1 = async { doSomething1() }
   val task2 = async { doSomething2() }
   ```

3. **await() で結果を受け取る**
   ```kotlin
   val result1 = task1.await()
   val result2 = task2.await()
   ```

4. **エラーハンドリングを忘れずに**
   - 1つのタスクがエラーになると、他のタスクもキャンセルされる
   - try-catch で適切に処理する

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

想像してください。ユーザーがアプリで以下のような操作をしたとします：

1. ユーザー詳細画面を開く
2. データの読み込みが始まる（10秒かかる）
3. 3秒後、ユーザーが「やっぱりいいや」と画面を閉じる
4. 画面は閉じられた
5. でも、バックグラウンドではまだデータ取得が続いている...（あと7秒）
6. データ取得完了！
7. **画面にデータを表示しようとする → クラッシュ！**（画面がもう存在しない）

```kotlin
// ❌ 最悪な例：絶対にやってはいけない
class UserActivity : AppCompatActivity() {
    fun loadUserData() {
        GlobalScope.launch {
            // この処理は10秒かかる
            val data = apiService.getUser()
            
            // ここで問題発生！
            // Activityが破棄されていたら、textViewは存在しない
            // → NullPointerException または IllegalStateException
            textView.text = data.name
        }
    }
}
```

#### 何が起きているのか？

- `GlobalScope` はアプリ全体のライフサイクルに紐づいている
- つまり、画面を閉じても、アプリを終了するまでコルーチンは動き続ける
- 画面が閉じられた後も、`textView` への参照が残り続ける → **メモリリーク**
- 存在しないViewにアクセスしようとする → **クラッシュ**

#### 正しい方法

**パターン1: ViewModelを使う場合（推奨）**

```kotlin
// ✅ 良い例：ViewModelScopeを使う
class UserViewModel : ViewModel() {
    private val _userData = MutableLiveData<User>()
    val userData: LiveData<User> = _userData
    
    fun loadUserData() {
        viewModelScope.launch {
            try {
                // ViewModelが破棄されたら、このコルーチンも自動的にキャンセル
                val data = apiService.getUser()
                _userData.value = data
            } catch (e: CancellationException) {
                // キャンセルされた時の処理
                Log.d("ViewModel", "コルーチンがキャンセルされました")
            }
        }
    }
}

// Activityから使う
class UserActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        viewModel.userData.observe(this) { user ->
            textView.text = user.name
        }
        
        viewModel.loadUserData()
    }
}
```

**パターン2: Activity/Fragmentで直接使う場合**

```kotlin
// ✅ 良い例：lifecycleScopeを使う
class UserActivity : AppCompatActivity() {
    fun loadUserData() {
        lifecycleScope.launch {
            try {
                // Activityが破棄されたら、このコルーチンも自動的にキャンセル
                val data = apiService.getUser()
                textView.text = data.name
            } catch (e: CancellationException) {
                // 画面を閉じた時に自動的にキャンセルされる
                Log.d("Activity", "画面が閉じられたのでキャンセル")
            }
        }
    }
}
```

#### なぜこれで問題が解決する？

- `viewModelScope` や `lifecycleScope` は、画面のライフサイクルに連動している
- 画面が閉じられると、自動的にコルーチンもキャンセルされる
- 不要な処理は即座に止まる → バッテリー節約、メモリリーク防止

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

以下のいずれかに当てはまる場合のみ `suspend` を付けましょう：
- ✅ `delay()` を使う
- ✅ 他のsuspend関数を呼ぶ
- ✅ `withContext()` でスレッドを切り替える
- ✅ ネットワーク通信、データベースアクセス、ファイルI/Oを行う

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
        
        // ユーザーには「アプリが固まった」ように見える
        // ANR (Application Not Responding) エラーが発生する可能性も
    }
}
```

#### ユーザー体験への影響

1. アプリを開く
2. 画面が表示される
3. **3秒間、何も操作できない**（ボタンを押しても反応しない）
4. 「アプリが壊れた？」とユーザーは思う
5. 最悪の場合、Androidが「アプリが応答しません」ダイアログを表示

#### 正しい方法

**パターン1: lifecycleScopeを使う**

```kotlin
// ✅ 良い例：適切なスコープを使う
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        // メインスレッドをブロックしない
        lifecycleScope.launch {
            // ローディング表示
            progressBar.visibility = View.VISIBLE
            
            delay(3000)
            loadData()
            
            // ローディング非表示
            progressBar.visibility = View.GONE
        }
        
        // この行はすぐに実行される（ブロックされない）
        setupUI()
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
    val user = repository.getUser("123")
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

- ❌ 本番のAndroidアプリコード（Activity、Fragment、ViewModel）では使わない
- ❌ メインスレッドで使わない
- ✅ テストコードでは使ってOK
- ✅ main関数（学習用）では使ってOK
- △ レガシーコードとの橋渡しで、どうしても必要な場合のみ（最終手段）

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
