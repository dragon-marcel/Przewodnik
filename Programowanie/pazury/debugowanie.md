# Jak debugować kod R? 

Często zdarza się, że kod R działa inaczej niż się spodziewamy.

Lub, że kiedyś działał tak jak chcieliśmy, ale na skutek zmiany jakiejś zależności przestał działać.

Czasem naprawdę trudno zrozumieć co jest przyczyną błędu, ponieważ nawet (wydawałoby się) podstawowe operacje mogą prowadzić do dziwnych zachowań.

Prześledźmy poniższy przykład. Czy nie widząc odpowiedzi, łatwo było przewidzieć jakie litery się wyświetlą na koniec?


```r
(sekwencja <- seq(0.7, 1, by=0.1))
```

```
## [1] 0.7 0.8 0.9 1.0
```

```r
(indeksy <- sekwencja*10 - 6)
```

```
## [1] 1 2 3 4
```

```r
LETTERS[indeksy]
```

```
## [1] "A" "A" "C" "D"
```

O co chodzi? Otóż nie należy indeksować wektorów liczbami rzeczywistymi, bo to igranie z precyzją numeryczną.


## Gdy coś pójdzie źle poza konsolą?

RStudio ułatwia debugowanie kodu. Ale co zrobić, jeżeli problem występuje w pracy z kodem zrównoleglonym, lub uruchomionym na jakimś serwerze HPC bez graficznego środowiska?

W takim przypadku użyteczną funkcją jest `dump.frames()`. W momencie uruchomienia zapisuje wszystkie otwarte ramki do pliku. 

Funkcja `option()` pozwala na określenie rożnych globalnych parametrów. Akurat parametr `error` opisuje co ma się wydarzyć, jeżeli interpreter R napotka na błąd. Poniższy przykład definiuje nowe zachowanie: jeżeli wystąpi błąd, wszystkie informacje o błędzie należy zapisać do pliku `errorDump`.

```
options(error=quote(dump.frames("errorDump",TRUE)))
```

Poniższy przykład definiuje funkcje, która wygeneruje błąd jeżeli za argument podać napis.

```
funkcja <- function(x) {
  log(x)
}
funkcja("napis")
## log: Using log base e.
## Error in log(x) : Non-numeric argument to mathematical function
## Execution halted
```

Efektem ubocznym jest utworzenie pliku `errorDump.rda` z zapisanymi przestrzeniami nazw w chwili gdy wystąpił błąd. Te dane można wczytać funkcją `load()` a następnie funkcją `debugger()` można odtworzyć stan z chwili gdy wystąpił błąd.

```
load("errorDump.rda")
debugger(errorDump)
```

Aby wyczyścić instrukcje zapisujące przestrzeń nazw przy każdym błędzie wystarczy za argument `error` podać `NULL`.

```
options(error=NULL)
```

## Automatyczny start debugera

Jeżeli pracujemy w trybie interaktywnym i każdy błąd lubimy przeanalizować, to wygodne będzie ustawienie funkcji `recover()` jako funkcji do wywołania po wystąpieniu błędu.

```
options(error = recover)
```

W najnowszych wersjach RStudio automatycznie pojawia się opcja włączenia interaktywnego debugera jeżeli tylko wystąpi jakiś błąd. Ale w starczych, lub gdy pracujemy na zdalnej konsoli, ta opcja jest wciąż przydatna.


## traceback()

Funkcja `traceback()` pozwala na analizę *post mortem*. Błąd się wydarzył, nasz program się przerwał, używając tej funkcji możemy wypisać listę ramek otwartych w chwili gdy pojawił się błąd.

## debug() / undebug()

Jeżeli funkcja nie generuje błędu, ale zachowuje się inaczej niż byśmy chcieli, to do prześledzenia jej wykonania krok po kroku można wykorzystać funkcję `debug()` (debugowanie wyłącza się funkcją `undebug()`). 

Funkcja `debug()` dodaje specjalny znacznik debugowania do funkcji. Za każdym razem gdy ta funkcja zostanie wywołana, włącza się interaktywne przetwarzanie linia po linii wybranej funkcji. 

```
funkcja2 <- function(x,y) {
  funkcja(x)
  funkcja(y)
}
debug(funkcja2)
funkcja2(1, "jeden")
```

## try() / tryCatch()

Wystąpienia błędu czasem nie sposób przewidzieć.
Ale to co można zrobić to określić jak ma wyglądać zachowanie R gdy błąd się pojawi.

Do tego celu można wykorzystać funkcję `try()` lub `tryCatch()`.

Jest ona wygodna, gdy np. uruchamiamy określony fragment kodu wielokrotnie, np. równolegle, oraz liczymy się z tym, że któreś uruchomienie może zakończyć się błędem (pobieranie złej strony www).
Jeżeli nie chcemy by błąd przerywał całość obliczeń, to dobrym rozwiązaniem jest jego przechwycenie.

Można do tego wykorzystać funkcja `try()` i `tryCatch()`.

Poniższa instrukcja wywoła w sposób bezpieczny funkcję `funkcja2(1, "jeden")`. Argument `silent=TRUE` oznacza, że nawet jeżeli błąd wystąpi to należy go zignorować. W takim jednak przypadku wynikiem funkcji `try()` będzie obiekt klasy `try-error`.

```
try(funkcja2(1, "jeden"), silent=TRUE)
```

## browser()

Funkcję `browser()` można umieszczać w środku innych funkcji lub bloku kodu. Gdy zostanie uruchomiona w wyniku otworzy bieżące środowisko i pozwoli na odpytanie stanu zmiennych w nim.


## Śledzenie z użyciem funkcji trace

Bardzo potężne możliwości debugowania kodu daje funkcja `trace()`.
Umożliwia ona wstrzyknięcie do dowolnej funkcji dowolnej instrukcji / wyrażenia. Ułatwia to debugowanie funkcji. 

Prześledźmy kilka typowych przykładów użycia.

Wywołanie funkcji `trace()` z tylko jednym argumentem - nazwą funkcji do śledzenia, powoduje, że za każdym razem gdy ta funkcja będzie wywołana wyświetli się komunikat.

Przykładowo, zobaczmy ile razy wywoływana jest funkcja `sum()` gdy liczy się histogram. Funkcja `untrace()` usuwa informacje o śledzeniu.


```r
trace(sum)
tmp <- hist(rnorm(100), plot = FALSE) 
```

```
## trace: sum
## trace: sum
## trace: sum
```

```r
untrace(sum)
```

Możliwości funkcji `trace()` są znacznie większe. Pozwala ona na wstrzyknięcie dowolnego kodu R w dowolne miejsce dowolnej funkcji. 

Przedstawimy to na przykładzie. Zacznijmy od stworzenia w miarę prostej funkcji


```r
f <- function(x) {
  y <- x + runif(1)
  z <- y + runif(1)
  w <- z + runif(1)
  w
}
```

Dodajemy informacje śledzące do funkcji `f()`. Przed trzecią linią wstrzykniemy kod `quote(browser())` a na koniec funkcji wstrzykniemy kod `quote(cat("Argumenty", x, y, z, w))`.
Możemy więc w dowolnym miejscu podejrzeć stan argumentów danej funkcji.


```r
trace("f", quote(browser()),
      at = 3,
      exit = quote(cat("Argumenty", x, y, z, w)))
```

```
## [1] "f"
```

Aby przekonać się co zrobiła funkcja `trace()` zobaczmy jak teraz wygląda ciało funkcji `f()`. Od razu widać kod śledzący.


```r
body(f)
```

```
## {
##     on.exit(.doTrace(cat("Argumenty", x, y, z, w), "on exit"))
##     {
##         y <- x + runif(1)
##         {
##             .doTrace(browser(), "step 3")
##             z <- y + runif(1)
##         }
##         w <- z + runif(1)
##         w
##     }
## }
```

Na koniec zawsze usuwamy informację o śledzeniu.


```r
untrace("f")
```


## Debugowanie w RStudio

RStudio posiada rozbudowane wsparcie dla debugowania, czy to zwykłych skryptów czy np. aplikacji shiny. W dowolnej linii kodu R można założyć pułapkę, gdy interpreter przeczyta ten fragment kodu to przerwanie skryptu zostanie przerwane i użytkownik ma opcje by prześledzić co się dzieje w określonym miejscy.

Więcej o debugowaniu w RStudio można przeczytać w pliku https://support.rstudio.com/hc/en-us/articles/205612627-Debugging-with-RStudio

## Pokrycie testów

Dobrym sposobem na możliwie rzadkie używanie debugera, jest pisanie dobrych i wielu testów jednostkowych.

Jak tworzyć testy - pisaliśmy to w rozdziale o pakietach. 

Aby sprawdzić jak dokładnie testy pokrywają kod pakietu, można wykorzystać pakiet `covr` i funkcje `package_coverage()`. Przedstawia ona tekstową analizę jaka część funkcji jest weryfikowana. Funkcje, które nie są wystarczająco pokryte testami warto uzupełnić.


```r
library(covr)
x <- package_coverage("~/GitHub/archivist")
x
```

```
## archivist Coverage: 39.51%
```

```
## R/addArchivistHooks.R: 0.00%
```

```
## R/addTagsRepo.R: 0.00%
```

```
## R/ahistory.R: 0.00%
```

```
## R/asession.R: 0.00%
```

```
## R/atags.R: 0.00%
```

```
## R/atrace.R: 0.00%
```

```
## R/cache.R: 0.00%
```

```
## R/getTags.R: 0.00%
```

```
## R/restoreLibraries.R: 0.00%
```

```
## R/rmFromRepo.R: 0.00%
```

```
## R/shinySearchInLocalRepo.R: 0.00%
```

```
## R/extractData.R: 1.16%
```

```
## R/extractMiniature.R: 20.00%
```

```
## R/extractTags.R: 22.39%
```

```
## R/loadFromRepo.R: 52.33%
```

```
## R/summaryRepo.R: 53.19%
```

```
## R/getRemoteHook.R: 58.62%
```

```
## R/zipRepo.R: 60.00%
```

```
## R/splitTags.R: 72.22%
```

```
## R/asearch.R: 74.19%
```

```
## R/zzz.R: 76.47%
```

```
## R/alink.R: 79.49%
```

```
## R/showRepo.R: 80.00%
```

```
## R/deleteRepo.R: 81.25%
```

```
## R/copyToRepo.R: 83.72%
```

```
## R/aread.R: 84.00%
```

```
## R/createEmptyRepo.R: 88.24%
```

```
## R/searchInRepo.R: 88.31%
```

```
## R/magrittr.R: 88.71%
```

```
## R/saveToRepo.R: 88.89%
```

```
## R/setRepo.R: 94.12%
```

```
## R/archivistOptions.R: 100.00%
```

```
## R/createMDGallery.R: 100.00%
```

## Więcej informacji

* Interesujący przegląd możliwości RStudio dotyczących debudowania http://www.win-vector.com/blog/2016/04/free-data-science-video-lecture-debugging-in-r/


