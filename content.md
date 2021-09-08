
# <span class="simple anim-text-flow">Modernes reaktives Spring-Boot</span> 
<i class="fa fa-user"></i>&nbsp;Daniel Hörner, <i class="fa fa-user"></i>&nbsp;Christoph Welcz, <i class="fa fa-user"></i>&nbsp;Valentin Petras
<--->

## <span class="words"><p class="words-line revert">motivation</p></span>

<-->

- Anzeige von Liquiditätsinformationen
- Sammeln und Aggregieren von Daten aus unterschiedlichen Quellen
- Erste produktive App in Kotlin

<--->
## <span class="words"><p class="words-line revert">reactive</p><p class="words-line"><span class="cleartxt anim-text-flow">coding</span></p></span>
<-->

- vielfach höherer Durchsatz als mit klassischem Spring
- Entkopplung & Parallelisierung von Aufrufen

<--->
## <span class="words"><p class="words-line revert">Kotlin</p><p class="words-line"><span class="cleartxt anim-text-flow">Coroutines</span></p></span>
<-->

- Programmieren im gewohnten imperativen Stil
- Einbetten aller Objekte in `Mono` nicht nötig!
- Einheitliche Programmierung über Stacks (RxJava, Project Reactor, Android) hinweg

<--->
## <span class="words"><p class="words-line revert">Data</p><p class="words-line"><span class="cleartxt anim-text-flow">Classes</span></p></span>

<-->

- Weniger Boilerplate
- Immutability vermeidet Race Conditions
- API Design mit Immutability und expliziten Null Checks für Pflichtfelder

<-->

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

- Berechnungen innerhalb statt Delegation an `Stream.collect`
- Sehr mächtige API Funktionen

<--->
## <span class="words"><p class="words-line revert">fazit</p></span>

<-->

- Reduktion des Codes von 24k LoC auf 3,8k
- &gt; 95 % Testabdeckung und TDD
- Wir wollen nicht mehr zurück nach Java 😉
