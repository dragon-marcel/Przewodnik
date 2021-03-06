# Jak pracować z bazami danych?
 
Zdecydowana większość funkcji w programie R wymaga by dane na których pracujemy były w pamięci RAM.
A jeżeli chcemy pracować z dużymi zbiorami danych, które zajmują dużo miejsca, to mamy dwie możliwości:

* pracować na ,,tłustych’’ komputerach z dużą ilością RAM (jednak na dzień dzisiejszy dzisiaj trudno wyjść poza 1 TB RAM, chyba że ma się baaardzo duży budżet),
* trzymać dane poza R, możliwie dużo przetwarzania wykonywać poza R, a do R wczytywać tylko takie dane, które są naprawdę niezbędne. Poza R, oznacza tutaj w bazie danych.

Nawet jeżeli danych nie jest bardzo dużo, to bazy dane mogą być używane by zapewnić jednolity sposób dostępu do danych z różnych narzędzi, by efektywnie zarządzać danymi, by zapewnić skalowalność operacji na danych.

Poniżej przedstawiamy dwa przykłady komunikacji R z bazami danych.
Pierwszy przykład będzie dotyczył prostej bazy danych `SQLite` bazującej na jednym pliku. Są to najczęściej zabawkowe przykłady pozwalające na przećwiczenie podstawowych operacji. Drugi przykład będzie dotyczył pracy z popularną bazą PostgreSQL. Sposób pracy z nią jest bardzo podobny do większości popularnie stosowanych baz relacyjnych.

## Jak pracować z bazą danych SQLite?

SQLite to lekka baza danych oparta o jeden plik. Ma ona dosyć ograniczone funkcjonalności, ale tak prosto ją zainstalować, że wręcz trudno powiedzieć w którym momencie się to robi. Łatwo też tę bazę skopiować czy komuś wysłać. Dlatego pomimo ograniczonych możliwości ma ona sporo zastosowań w których liczby się prostota nad skalowalnością.

Aby korzystać z tej bazy danych potrzebny jest pakiet `RSQLite`.
Korzystanie z tej bazy składa się z następujących kroków

* Należy wczytać sterownik do łączenia się z bazą danych, najczęściej funkcją `dbDriver()`.
* Należy nawiązać połączenie z bazą danych, najczęściej funkcją `dbConnect()`. W przypadku bazy SQLite wystarczy wskazać ścieżkę do pliku z bazą danych.
* Operacje na bazie danych wykonuje się poprzez nawiązane połączenie.
* Po zakończeniu pracy z bazą danych należy zwolnić połączenie (funkcja `dbDisconnect()`) i sterownik.

Przykładowa sesja z bazą danych jest następująca.

Ładujemy sterownik do bazy danych i inicjujemy połączenie z serwerem bazodanowym. Jeżeli wskazany plik nie istnieje, to zostanie stworzony z pustą bazą danych.


```r
library("RSQLite")
sterownik <- dbDriver("SQLite")
polaczenie <- dbConnect(sterownik, "zabawka.db")
```

Wyświetlamy tabele widoczne w bazie danych pod wskazanym połączeniem a następnie wyświetlamy kolumny w określonej tabeli.


```r
dbListTables(polaczenie)
```

```
## [1] "auta2012"     "sqlite_stat1" "wynik"
```

```r
dbListFields(polaczenie, "auta2012")
```

```
##  [1] "Cena"                         "Waluta"                      
##  [3] "Cena.w.PLN"                   "Brutto.netto"                
##  [5] "KM"                           "kW"                          
##  [7] "Marka"                        "Model"                       
##  [9] "Wersja"                       "Liczba.drzwi"                
## [11] "Pojemnosc.skokowa"            "Przebieg.w.km"               
## [13] "Rodzaj.paliwa"                "Rok.produkcji"               
## [15] "Kolor"                        "Kraj.aktualnej.rejestracji"  
## [17] "Kraj.pochodzenia"             "Pojazd.uszkodzony"           
## [19] "Skrzynia.biegow"              "Status.pojazdu.sprowadzonego"
## [21] "Wyposazenie.dodatkowe"
```

Używając funkcji `dbGetQuery()` możemy wykonać na bazie zapytanie SQL.


```r
pierwsze5 <- dbGetQuery(polaczenie, 
           "select Cena, Waluta, Marka, Model from auta2012 limit 5")
pierwsze5
```

```
##    Cena Waluta         Marka     Model
## 1 49900    PLN           Kia    Carens
## 2 88000    PLN    Mitsubishi Outlander
## 3 86000    PLN     Chevrolet   Captiva
## 4 25900    PLN         Volvo       S80
## 5 55900    PLN Mercedes-Benz  Sprinter
```

```r
agregat <- dbGetQuery(polaczenie, 
           "select count(*) as liczba, avg(`Cena.w.PLN`) as cena, Marka from auta2012 group by Marka limit 10")
agregat
```

```
##    liczba      cena       Marka
## 1      24  32735.33            
## 2      59  68140.39       Acura
## 3      50  27037.04       Aixam
## 4    2142  21403.58   AlfaRomeo
## 5       4  15424.42         Aro
## 6      35 505359.61 AstonMartin
## 7   12851  64608.61        Audi
## 8      17  46396.60      Austin
## 9   10126  72385.68         BMW
## 10     39 478483.82     Bentley
```

Używając funkcji `dbDisconnect()` możemy się z bazą danych rozłączyć. Ważne jest by po sobie sprzątać na wypadek gdyby dane z pamięci nie zostały zapisane do pliku.


```r
dbDisconnect(polaczenie)
```

```
## [1] TRUE
```

Funkcja `dbGetQuery()` tworzy zapytanie, wykonuje je i pobiera jego wyniki. Tę operację można rozbić na części. Funkcja `dbSendQuery()` jedynie tworzy i wysyła zapytanie SQL do bazy, a funkcja `fetch()` pobiera kolejne porcje danych.

Funkcja `dbWriteTable()` zapisuje wskazany obiekt `data.table` jako tabelę w bazie danych.


## Jak pracować z relacyjnymi bazami danych?

SQLite to baza zabawka. Do przechowywania większych danych w produkcyjnych rozwiązaniach wykorzystywać można otwarte rozwiązania typu PostgreSQL czy MySQL lub komercyjnie rozwijane silniki bazodanowe takie jak Oracle, RedShift, Teradata, Netezza i inne.

O ile te bazy różnią się funkcjonalnością, skalowalnością i prędkością, to z perspektywy użytkownika R korzystanie z nich jest dosyć podobne. Poniżej pokażemy jak korzystać z PostgreSQL.

Jako przykład, wykorzystamy bazę PostgreSQL dostępną na serwerze `services.mini.pw.edu.pl`. PostgreSQL pozwala na zarządzanie wieloma użytkownikami i wieloma bazami danych, tutaj wykorzystamy bazę `sejmrp` przechowującą dane z Sejmu RP 7 i 8 kadencji (głosowania i stenogramy). Dane te są uzupełniane za pomocą pakietu  [sejmRP](https://github.com/mi2-warsaw/sejmRP).

Aby korzystać z bazy danych potrzebny jest użytkownik i hasło. Poniżej przedstawimy przykład dla użytkownika `reader` i hasła `qux94874`. Ten użytkownik ma wyłącznie uprawnienia do czytania.

Aby połączyć się z bazą PostgreSQL potrzebujemy sterownika, który jest dostępny w pakiecie `RPostgreSQL`. Wczytajmy ten pakiet i nawiążmy połączenie z bazą danych.


```r
library(RPostgreSQL)
dbname = "sejmrp"
user = "reader"
password = "qux94874"
host = "services.mini.pw.edu.pl"

sterownik <- dbDriver("PostgreSQL")
polaczenie <- dbConnect(sterownik, dbname = dbname, user = user, password = password, host = host)
```

Możemy teraz zadawać dowolne zapytania SQL, pobierać i wysyłać całe tabele z danymi.


```r
gadki <- dbGetQuery(polaczenie, "SELECT * FROM statements ORDER BY nr_term_of_office, id_statement limit 1")
gadki
```

```
##   id_statement nr_term_of_office                    surname_name
## 1    100.1.001                 7 Sekretarz Poseł Marek Poznański
##   date_statement titles_order_points
## 1     2015-09-16                    
##                                                                                          statement
## 1 Informuję, że w dniu dzisiejszym o godz. 16 odbędzie się posiedzenie Komisji  Zdrowia. Dziękuję.
```

```r
glosy <- dbGetQuery(polaczenie, "SELECT club, vote, count(*) FROM votes GROUP BY club, vote limit 12")
glosy  
```

```
##       club          vote  count
## 1       ZP       Przeciw   8448
## 2       PO       Przeciw 827472
## 3     KPSP       Przeciw   5659
## 4    niez.     Nieobecny  12418
## 5       PO Wstrzymał się   3291
## 6       ZP Wstrzymał się   3472
## 7  Kukiz15            Za   4495
## 8    niez.            Za  27274
## 9     KPSP Wstrzymał się   2699
## 10 Kukiz15     Nieobecny    893
## 11     PSL       Przeciw 120436
## 12     PiS Wstrzymał się 111698
```

Na koniec pracy należy rozłączyć się z bazą danych i zwolnić połączenie. 
  

```r
dbDisconnect(polaczenie)
```

```
## [1] TRUE
```


## Jak używać pakietu dplyr w pracy z bazami danych?

Standardów i implementacji SQLa jest tak wiele, że zastanawiające jest dlaczego nazywane są standardami. Praktycznie każda baza danych różni się listą zaimplementowanych funkcjonalności czy agregatów. Jeżeli pracujemy z jedną bazą danych to ta różnorodność nie będzie nam doskwierać. Ale jeżeli przyjdzie nam jednocześnie korzystać z baz MSSQL, MySQL, RedShift i Postgres? Lub gdy okaże się, że dane zostały zmigrowane na nową bazę danych?

Wielu problemów można sobie oszczędzić używając pośrednika do komunikacji z bazą danych. Takim pośrednikiem może być pakiet `dplyr` omówiony w poprzednim rozdziale.
Pozwala on do pewnego stopnia na pracę z danymi bez zastanawiania się gdzie te dane aktualnie są i jak nazywa się w tym systemie bazodanowym potrzebna nam funkcja.

W pakiecie `dplyr` połączenia ze źródłem danych (tabelą z liczbami lub bazą danych) tworzy się funkcjami `src_*`. 
Poniżej skorzystamy z funkcji `src_sqlite()` i `src_postgres()`.


Zainicjujmy połączenie z bazą SQLite i pobierzmy kilka pierwszych wierszy z tabeli `auta2012`.


```r
 library(dplyr)
polaczenie <- src_sqlite(path = 'zabawka.db')
auta1 <- tbl(polaczenie, "auta2012")
```

Mając taki obiekt, reszta operacji wygląda tak jak w zwykłym `dplyr`.


```r
auta1 %>% 
  head(2)
```

```
##    Cena Waluta Cena.w.PLN Brutto.netto  KM  kW      Marka     Model Wersja
## 1 49900    PLN      49900       brutto 140 103        Kia    Carens       
## 2 88000    PLN      88000       brutto 156 115 Mitsubishi Outlander       
##   Liczba.drzwi Pojemnosc.skokowa Przebieg.w.km          Rodzaj.paliwa
## 1          4/5              1991         41000 olej napedowy (diesel)
## 2          4/5              2179         46500 olej napedowy (diesel)
##   Rok.produkcji Kolor Kraj.aktualnej.rejestracji Kraj.pochodzenia
## 1          2008                           Polska                 
## 2          2008                           Polska                 
##   Pojazd.uszkodzony Skrzynia.biegow Status.pojazdu.sprowadzonego
## 1                          manualna                             
## 2                          manualna                             
##                                                                                                                                                                                Wyposazenie.dodatkowe
## 1                         ABS, el. lusterka, klimatyzacja, alufelgi, centralny zamek, autoalarm, poduszka powietrzna, radio / CD, wspomaganie kierownicy, immobiliser, komputer, przyciemniane szyby
## 2 ABS, 4x4, el. lusterka, klimatyzacja, skorzana tapicerka, alufelgi, centralny zamek, poduszka powietrzna, radio / CD, wspomaganie kierownicy, immobiliser, komputer, tempomat, przyciemniane szyby
```

A teraz pokażmy przykład dla bazy PostgreSQL. Zainicjujmy połączenie do tabeli `votes`.


```r
polaczenie <- src_postgres(dbname = dbname, 
                        host = host, user = user, password = password)
src_tbls(polaczenie)
```

```
## [1] "votes"            "counter"          "statements"      
## [4] "deputies"         "votings"          "votes_copy_27_08"
## [7] "db"               "test_statements"
```

```r
glosy <- tbl(polaczenie, "votes")
```

Zdefiniowawszy połączenie możemy na tej tabeli robić dosyć zaawansowane rzeczy używając już znanych funkcji z pakietu `dplyr`.


```r
glosy %>% 
  group_by(club, vote) %>% 
  summarise(liczba = n()) ->
  liczba_glosow

class(liczba_glosow)
```

```
## [1] "tbl_postgres" "tbl_sql"      "tbl"
```

Wynikiem tych operacji jest obiekt klasy `tbl_sql`. Nie przechowuje on jednak danych, ale instrukcje pozwalające na dostęp do danych (zapytanie SQL). Można to zapytanie i plan zapytania wyłuskać


```r
liczba_glosow$query
```

```
## <Query> SELECT "club", "vote", "liczba"
## FROM (SELECT "club", "vote", count(*) AS "liczba"
## FROM "votes"
## GROUP BY "club", "vote") AS "zzz3"
## <PostgreSQLConnection:(42999,5)>
```

```r
explain(liczba_glosow)
```

```
## <SQL>
## SELECT "club", "vote", "liczba"
## FROM (SELECT "club", "vote", count(*) AS "liczba"
## FROM "votes"
## GROUP BY "club", "vote") AS "zzz3"
```

```
## 
```

```
## <PLAN>
## HashAggregate  (cost=74577.20..74577.72 rows=52 width=9)
##   ->  Seq Scan on votes  (cost=0.00..51780.54 rows=3039554 width=9)
```


Leniwość ale nie lenistwo. Operacje na tych obiektach nie są materializowane o ile nie muszą być materializowane (użytkownik wprost tego zażąda). Gdy już jest jasne co użytkownik chce zrobić i gdy jawnie zażąda wyniku, wszystkie operacje są wykonywane w możliwie małej liczbie (=jeden) kroków.

Materializować wyniki można na dwa sposoby

* `collect()` - wyznacza wynik oraz pobiera do R,
* `compute()` - wyznacza wynik i zapisuje w tymczasowej tabeli w bazie danych.

Poniższa instrukcja pobierze wyliczone agregaty ze zbioru danych.


```r
collect(liczba_glosow)
```

```
## Source: local data frame [60 x 3]
## Groups: club [15]
## 
##       club          vote liczba
##      (chr)         (chr)  (dbl)
## 1       ZP       Przeciw   8448
## 2       PO       Przeciw 827472
## 3     KPSP       Przeciw   5659
## 4    niez.     Nieobecny  12418
## 5       PO Wstrzymał się   3291
## 6       ZP Wstrzymał się   3472
## 7  Kukiz15            Za   4495
## 8    niez.            Za  27274
## 9     KPSP Wstrzymał się   2699
## 10 Kukiz15     Nieobecny    893
## ..     ...           ...    ...
```

Więcej informacji o funkcjach z pakietu `dplyr`, które można stosowac do baz danych znaleźć można na stronie http://cran.rstudio.com/web/packages/dplyr/vignettes/databases.html.

## Do ćwiczeń

Czego potrzebuje użytkownik `mi2user`?


```r
digest::digest("All you need is R")
```

```
## [1] "bdb54d9c58b91c382f58423bec2bf5f0"
```

I funkcji `copy_to`, która zapisuje lokalną tabelę do bazy danych.
