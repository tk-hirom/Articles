# 同期処理、非同期処理、並行処理、並列処理の違い

よくジュニアメンバーの方たちから、このあたりの誤解、混同した会話が聞こえてくる。

自分も初め混乱したなぁと思いつつ、お伝えした内容をまとめておく。

## 同期処理
一つ一つ処理していく形式。
前の処理の終了しないと次には進まない

ホールスタッフのAさんが一つの注文を受け取り、厨房にそれを伝える。厨房から料理が出てくるまでぼーっとしてる

料理が待ち時間なく厨房から提供されるなら良いが、それなりに待つならとても効率が悪い。

実装としては

```kotlin
// 同期処理の例
fun fetchUserSync(userId: Int): User {
    val response = fetchFromAPI(userId) // ここで待つ
    println(response)
    processData(response) // これはresponseが来るまで実行されない
    return response
}
```

のようになり、APIやDBからレスポンスが来ないとそれ以降の処理はしないという物

テキトーに書くと大体同期処理になっちゃいます

## 非同期処理とは
前の処理結果を待たずに、次の処理を進める処理方式

## 並行処理とは
同時に複数のタスクをこなしているように見せる処理方式。非同期処理を実現するために用いられる。
同時に複数タスクを実行してはいない。

ホールスタッフAさんが厨房からの料理提供時間を待ってる間、他の注文を取りに行く。
これが非同期処理である。

代表的な利用ケースはI/Oなどだ

実装としては

```kotlin
// 並行処理の例（Kotlin Coroutine）
suspend fun fetchUserAsync(userId: Int): User {
    val response = fetchFromAPIAsync(userId) // ここで待つが、他の処理は続く
    println(response)
    processData(response)
    return response
}

// 複数の非同期処理を並行実行
GlobalScope.launch {
    val user1 = async { fetchUserAsync(1) }
    val user2 = async { fetchUserAsync(2) }
    val user3 = async { fetchUserAsync(3) }
    
    // すべての結果を待つ（または個別に結果を取得）
    val results = awaitAll(user1, user2, user3)
    println(results)
}
```

となる。お互いに依存関係のない処理なので待たずに行っている。

前述の通り、実際は複数のタスクを切り替えながら、待ち時間で他タスクを行なっているだけである。

### 代表的な並行処理の技術
- Kotlin Coroutine✖️WebFlux（Spring Reactor）
- Python asyncio

## 並列処理とは
複数スレッドを立てるなどして、実際に同時に複数のタスクを処理する形式。
非同期処理を実現するために用いられる。

ホールスタッフの例で言えば、Aさん以外にB,Cさんにも任せる形になると並列処理といえる
ただ、並行処理とは別物なので一人一人が厨房からの料理の提供をボーッと待ってても並列処理といえる

実装としては

```kotlin
// 並列処理の例（Kotlin マルチスレッド）
val executor = Executors.newFixedThreadPool(3)

executor.submit {
    fetchUserAndProcess(1)
}
executor.submit {
    fetchUserAndProcess(2)
}
executor.submit {
    fetchUserAndProcess(3)
}

executor.shutdown()
executor.awaitTermination(1, TimeUnit.MINUTES)
```

### 代表的な並列処理の技術
- Kotlin マルチスレッド（Thread）
- Java ExecutorService
- Python multiprocessing

## それぞれいつ使うべきか？

| 処理方式 | 使用場面 | メリット | デメリット |
|---------|---------|---------|---------|
| **同期処理** | 処理の依存関係が強い場合、シンプルな実装 | 実装が簡単、デバッグが容易 | 待機時間が多いと効率が悪い |
| **並行処理** | I/O待機が多い場合（API呼び出し、DB検索など）、単一スレッド環境 | リソース効率が良い、実装も比較的簡単 | CPU集約的なタスクには不向き |
| **並列処理** | CPU集約的な処理（大量計算、画像処理など）、複数スレッド環境対応 | 本当の意味で同時実行、高速化が期待できる | 実装が複雑、スレッドセーフ対策が必要 |

## まとめ
重要な概念なのに混同しやすいのでまとめてみました。

以上の説明のホールスタッフ　= スレッドと読みかえ、再度上から読み直せば技術の話としても理解できるでしょう。