
# <span class="simple anim-text-flow">Modernes reaktives Spring-Boot</span> 
<i class="fa fa-user"></i>&nbsp;Daniel Hörner
<i class="fa fa-user"></i>&nbsp;Christoph Welcz
<i class="fa fa-user"></i>&nbsp;Valentin Petras

<span class="small">https://schlammspringer.github.io/modern-springboot-with-kotlin</span>
<--->

## <span class="words"><p class="words-line revert">motivation</p></span>

<-->

- Anzeige von Liquiditätsinformationen
- Sammeln und Aggregieren von Daten aus unterschiedlichen Quellen
- Erste produktive App in Kotlin

<--->

## <span class="words"><p class="words-line revert">reactive</p><p class="words-line"><span class="cleartxt anim-text-flow">coding</span></p></span>

<-->

### <span class="words"><p class="words-line revert">better</p><p class="words-line"><span class="cleartxt anim-text-flow">performance</span></p></span>

<-->

> "... Spring Webflux with WebClient and Apache clients wins in all cases. The most significant difference (4 times faster than blocking Servlet) when underlying service is slow (500ms)."

Note:
- economic & ecologic benefits

<-->

Laufzeitentkopplung

```kotlin
@PutMapping(...)
suspend fun upload(@RequestBody uploaded: Flow<Item>) {
  // this runs in background
  CoroutineScope(Dispatchers.IO).launch {
    collector.collect(uploaded.toList())
  }
  // immediately returned
  return "🤞"
}
```

```java
@PutMapping(...)
public Mono<String> upload(@RequestBody Flux<Item> uploaded) {
  // this runs in background
  uploaded.collectList()
          .publishOn(Schedulers.elastic())
          .doOnSuccess(collector::collect)
          .then()
          // Monos are cold!
          .subscribe();
  // immediately returned
  return Mono.just("🤞");
}
```

<-->

Parallelisierung von Aufrufen

```kotlin
= coroutineScope {
  // pseudo-imperative style
  val accountsFromA = async { 
    readAccountsByApiFromDomainA(consultant, client) 
  }
  val accountsFromB = async { 
    readAccountsByApiFromDomainB(consultant, client) 
  }
  combineAccounts(accountsFromA.await(), accountsFromB.await())
}
```

```java
  // Functional Reactive Programming
  return readAccountsByApiFromDomainA(consultant, client)
    .collectList()
    .publishOn(Schedulers.elastic())
    .zipWith(
      readAccountsByApiFromDomainB(consultant, client)
        .collectList()
        .publishOn(Schedulers.elastic()), 
      (accountsFromA, accountsFromB) -> 
        combineAccounts(accountsFromA, accountsFromB)
    );
```

<--->
## <span class="words"><p class="words-line revert">Kotlin</p><p class="words-line"><span class="cleartxt anim-text-flow">Coroutines</span></p></span>

<-->

- Superset von `async`/`await`
- Programmieren im gewohnten imperativen Stil
- Einbetten aller Objekte in `Mono` nicht nötig!
- Einheitliche Programmierung über Stacks (RxJava, Project Reactor, Android) hinweg

<-->

```kotlin
fun main() = runBlocking {
  repeat(100_000) { // launch a lot of coroutines
    launch {
      delay(5000L)
      print(".")
    }
  }
}
```
```kotlin
fun main() = runBlocking {
    doWorld()
}

suspend fun doWorld() = coroutineScope {  // this: CoroutineScope
    launch {
        delay(1000L)
        println("World!")
    }
    println("Hello")
}
```

<--->

## <span class="words"><p class="words-line revert">Data</p><p class="words-line"><span class="cleartxt anim-text-flow">Classes</span></p></span>

<-->

Weniger Boilerplate

```kotlin
data class Account(
  val id: ObjectId? = null,
  val consultantNumber: Long,
  val clientNumber: Long,
  val name: String,
  val iban: String,
)
```

<-->

Immutability vermeidet Race Conditions

```kotlin
val account = Account(...)
account = Account(...) // does not compile!
account.name = "Neuer Name" // does not compile!
// mutable property 
var account = Account(...)
// clone property with only a few new informations
val changedAccount = account.copy(name = "Neuer Name")
```

NOTE:
`copy` ist der idiomatische Weg!

<-->

API Design mit Immutability und expliziten Null Checks für Pflichtfelder (bei REST Calls/Jackson BAD REQUEST)

```kotlin
val a: Int = null // does not compile!
// val a: Int? = null

val b: String? = ...
b.length // does not compile!
// b?.length
```

<--->

## <span class="words"><p class="words-line revert">Kotest</p><p class="words-line"><span class="cleartxt anim-text-flow">testing</span></p></span>

<-->

- Property Based Testing
- BDD Strukturierung der Tests
- Testen von Coroutines

<--->

## <span class="words"><p class="words-line revert">list</p><p class="words-line"><span class="cleartxt anim-text-flow">functions</span></p></span>

<-->

Sehr mächtige API Funktionen statt Java `for each` Kaskade 
```kotlin
// Relative Veränderung des Kontostands mit Startsaldo
fun List<BigDecimal>.convertAbsolutesToDeltas(totalStart: BigDecimal) =
  (listOf(totalStart) + this) // ergänzt Salden um Startsaldo
  .zipWithNext { // bildet WertPaare
    current, next -> next - current // berechnet Delta aus vorherigen und aktuellen Wert 
  } 


// Absolute Veränderung des Kontostands mit Startsaldo
fun List<BigDecimal>.convertDeltasToAbsolutes(totalStart: BigDecimal) =
  runningReduce { previous, current -> current + previous } // addiert den vorherigen Wert auf den aktuellen auf
  .map { it + totalStart } // Absoluter Anfangswert aufaddieren
```
<-->

Berechnungen innerhalb statt Delegation an `Stream.collect`
```kotlin
// Ermittlung Gesamtbetrag aller Konten auf Basis der Tagessalden
fun List<DailyBalance>.sumLatestTotalsByIban() = groupBy { it.iban }
  .map { (_,value) -> value.maxBy { it.day } }
  .sumOf { it.total }
```
```java
// Ermittlung Gesamtbetrag aller Konten auf Basis der Tagessalden
public BigDecimal sumLatestTotalsByIban(List<DailyBalance> dailyBalances) {
    return dailyBalances
        .stream()
        .collect(groupingBy(DailyBalance::getIban))
        .values()
        .stream()
        .flatMap(balances -> balances.stream()
                                     .max(comparing(DailyBalance::getDay))
                                     .stream())
        .map(DailyBalance::getTotal)
        .reduce(BigDecimal.ZERO, BigDecimal::add);
  }
```

<-->

Java `for each` Kaskade, statt Kotlin oder Java streaming
```java
// Ermittlung Gesamtbetrag aller Konten auf Basis der Tagessalden
public BigDecimal sumLatestTotalsByIban(List<DailyBalance> dailyBalances) {
  var acc = BigDecimal.ZERO;
  var balancesGroupedByIban = new HashMap<String, List<DailyBalance>>();
  for (var balance : dailyBalances) {
    balancesGroupedByIban
        .computeIfAbsent(balance.getIban(), k -> new ArrayList<>())
        .add(balance);
  }
  for (var balances : balancesGroupedByIban.values()) {
    DailyBalance latestDailyBalance = null;
    for (var balance : balances) {
      if (latestDailyBalance == null) {
        latestDailyBalance = balance;
      }
      else if (balance.getDay().isAfter(latestDailyBalance.getDay())) {
        latestDailyBalance = balance;
      }
    }
    var total = Objects.requireNonNull(latestDailyBalance).getTotal();
    acc = acc.add(total);
  }
  return acc;
}
```

<--->

## <span class="words"><p class="words-line revert">kotlin</p><p class="words-line"><span class="cleartxt anim-text-flow">language</span></p></span>

<-->

- Expression functions für weniger Boilerplate
- Domain Language durch Extension functions

NOTE: 
- Gegenüberstellung Lesbarkeit von Code
- Vermeidung von Util-Klassen und Feature Envy

<--->

## <span class="words"><p class="words-line revert">fazit</p></span>

<-->

- Reduktion des Codes von 24k LoC auf 3,8k
- &gt; 95 % Testabdeckung und TDD
- Wir wollen nicht mehr zurück nach Java 😉

https://schlammspringer.github.io/modern-springboot-with-kotlin

NOTE: Reduktion gegenüber eines Prototyps in Java

<--->
- <cite>[Höherer Durchsatz als Spring](https://medium.com/@filia.aleks/microservice-performance-battle-spring-mvc-vs-webflux-80d39fd81bf0)</cite>
- <cite>[Example from Kotlin Guide to Coroutines](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/jvm/test/guide/example-basic-06.kt)</cite>
- <cite>[Flux 3 - Hopping Threads and Schedulers](https://spring.io/blog/2019/12/13/flight-of-the-flux-3-hopping-threads-and-schedulers)</cite>
- <cite>[Spring, Coroutines and Kotlin Flow](https://spring.io/blog/2019/04/12/going-reactive-with-spring-coroutines-and-kotlin-flow)</cite>
