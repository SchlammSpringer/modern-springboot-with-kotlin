
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

<-->

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
account.name = "New Name" // does not compile!
// mutable property 
var account = Account(...)
// clone property with only a few new informations
val changedAccount = account.copy(name = "New Name")
```

NOTE:
`copy` ist der idiomatische Weg!

<-->

- API Design mit Immutability und expliziten Null Checks für Pflichtfelder 
- bei REST Calls/Jackson BAD REQUEST

```kotlin
val a: Int = null // does not compile!
val a: Int? = null

val b: String? = ...
b.length // does not compile!
b?.length
```

<--->

## <span class="words"><p class="words-line revert">list</p><p class="words-line"><span class="cleartxt anim-text-flow">functions</span></p></span>

<-->

Berechnungen innerhalb statt Delegation an `Stream.collect`
```kotlin
// Ermittlung Gesamtbetrag aller Konten auf Basis der Tagessalden
fun List<DailyBalance>.sumLatestTotalsByIban() = groupBy { it.iban }
  .map { (_,value) -> value.maxBy { it.day } }
  .sumOf { it.total }
```

<-->

Java Streaming API

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

Java `for each` Kaskade
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

<-->

Sehr mächtige API Funktionen statt Java `for each` Kaskade
```kotlin
// Relative Veränderung des Kontostands mit Startsaldo
fun List<BigDecimal>.convertAbsolutesToDeltas(totalStart: BigDecimal) =
  // ergänzt Salden um Startsaldo
  (listOf(totalStart) + this) 
    // bildet WertPaare
    // berechnet Delta aus vorherigen und aktuellen Wert 
    .zipWithNext { current, next -> next - current } 

// Absolute Veränderung des Kontostands mit Startsaldo
fun List<BigDecimal>.convertDeltasToAbsolutes(totalStart: BigDecimal) =
  // addiert den vorherigen Wert auf den aktuellen auf
  runningReduce { previous, current -> current + previous } 
  // absoluten Anfangswert aufaddieren
  .map { it + totalStart } 
```

<--->

## <span class="words"><p class="words-line revert">kotlin</p><p class="words-line"><span class="cleartxt anim-text-flow">language</span></p></span>

<-->

Expression functions für weniger Boilerplate

```kotlin
// block body
fun helloWorld(): String {
  return "Hello World"
}

// expression with type inference
fun helloWorld() = "Hello World"

```
Note:
- weniger refactoring pain durch type inference bei Änderung von Rückgabewerten

<-->

Domain Language durch Extension functions

```kotlin
private fun BigDecimal.toEuroCent() = 
  multiply(BigDecimal(100)).toLong()

val total = (value1 - value2).toEuroCent()
```

```java
private long toEuroCent(BigDecimal value) {
  return value.multiply(new BigDecimal(100)).longValue();
}

var total = toEuroCent(value1.subtract(value2));
```

Note:
- Kotlin Listen Funktionen sind alles extension functions

<--->

## <span class="words"><p class="words-line revert">Kotest</p><p class="words-line"><span class="cleartxt anim-text-flow">testing</span></p></span>

<-->

BDD Strukturierung der Tests

```kotlin
  describe("API for Payment Plan") {
    describe("has POST") {
      it("returns new plan") {...}

      it("rejects new plan with wrong identifier") {...}
    }

    describe("has GET") {...}

    describe("has PUT {id}") {...}

    describe("has DELETE {id}") {...}
  }
```

<-->

höhere Konfidenz durch Property Based Testing

```log
Property failed after 671 attempts
Arg 0: (1980-11-29, 1972-10-30..2007-09-27)
Arg 1: Step(width=1, unit=Months)
Repeat this test by using seed 6737234594828379639
Caused by: 97 elements passed but expected 98
The following elements passed:
1972-10-30
1972-11-30
1972-12-30
1973-01-30
1973-02-28
1973-03-30
1973-04-30
1973-05-30
1973-06-30
1973-07-30
... and 87 more passed elements
The following elements failed:
1980-11-30 => 1980-11-30 should be <= 1980-11-29
```

<-->

```kotlin
it("supports a pre-emptive end") {
  val endBeforeRangeEnd = arbDateAndRange.filter { (end, range) -> 
    range.contains(end) && end < range.endInclusive 
  }
  checkAll(endBeforeRangeEnd, arbStep) { (end, range), step ->
    range.temporalSequence(step, range.startInclusive, end).forAll {
      it shouldBeLessThanOrEqualTo end
    }
  }
}
```

<-->

Kotest Generatoren für Property Tests

```kotlin
it("functions are isomorphic") {
  checkAll(Arb.bigDecimal(), Arb.list(Arb.bigDecimal())) { total, list ->
    list.convertDeltasToAbsolutes(total).convertAbsolutesToDeltas(total) 
        shouldEqualIgnoringTrailingZeros list
    list.convertAbsolutesToDeltas(total).convertDeltasToAbsolutes(total) 
        shouldEqualIgnoringTrailingZeros list
  }
}
```

<--->


## <span class="words"><p class="words-line revert">fazit</p></span>

<-->

Wir wollen nicht mehr zurück nach Java 😉

<--->

https://schlammspringer.github.io/modern-springboot-with-kotlin

- <cite>[Spring Boot Kotlin](https://spring.io/guides/tutorials/spring-boot-kotlin/)</cite>
- <cite>[Höherer Durchsatz als Spring](https://medium.com/@filia.aleks/microservice-performance-battle-spring-mvc-vs-webflux-80d39fd81bf0)</cite>
- <cite>[Example from Kotlin Guide to Coroutines](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/jvm/test/guide/example-basic-06.kt)</cite>
- <cite>[Flux 3 - Hopping Threads and Schedulers](https://spring.io/blog/2019/12/13/flight-of-the-flux-3-hopping-threads-and-schedulers)</cite>
- <cite>[Spring, Coroutines and Kotlin Flow](https://spring.io/blog/2019/04/12/going-reactive-with-spring-coroutines-and-kotlin-flow)</cite>
- <cite>[Kotest Generators](https://kotest.io/docs/proptest/property-test-generators.html#arbitrary)</cite>
- <cite>[Kotest](https://kotest.io/)</cite>
- <cite>[MockK - Mocking Library für Kotlin inkl. Coroutinen](https://mockk.io/)</cite>
