# Automatyczna klasyfikacja faktur KSeF przed importem (enova365)

> Element większej całości: **[Obieg faktur zakupu z KSeF w enova365 — mapa rozwiązania](https://github.com/trynityeu/enova365-obieg-faktur-ksef)**

Zestaw **cech algorytmicznych** w systemie ERP **enova365** (Soneta
sp. z o.o.), które nadają każdej fakturze pobranej z **Krajowego Systemu
e-Faktur (KSeF)** rodzaj dokumentu — czyli rozstrzygają, **którą ścieżką
importu** faktura pojedzie i w jakiej ewidencji wyląduje — oraz pokazują
operatorowi na liście wszystko, czego potrzebuje, żeby tę klasyfikację
zweryfikować bez otwierania pojedynczych plików.

To nie jest dodatek (DLL) — to **konfiguracja bazy danych**: cechy
wyliczane, które enova365 wywołuje sama, w standardowym miejscu procesu.

To repozytorium zawiera wyłącznie **opis funkcjonalny** — bez kodu
cech, bez danych klienta, bez konfiguracji wdrożeniowej.

Odbiorcy tej klasyfikacji:

- [Import faktur zakupu materiałowego (ZME) z dopasowaniem do zamówień](https://github.com/trynityeu/enova365-import-faktur-ksef-dopasowanie)
- [Import faktur kosztowych (ZKE) i samochodowych (ZSE)](https://github.com/trynityeu/enova365-import-faktur-ksef-koszty-pojazdy)

## Problem, który rozwiązuje

Faktury spływają z KSeF jedną wspólną listą, a rozchodzą się na trzy
zupełnie różne procesy księgowe: **zakup materiałowy** (wchodzi na
magazyn, wymaga uzgodnienia z zamówieniem), **zakup samochodowy**
(pojazdy — odrębny reżim odliczeń VAT) i **zakup kosztowy** (cała
reszta). Każdy z nich to inna ewidencja, inny dokument i inny przycisk
importu.

Rozstrzygnięcie „czym jest ta faktura" musi więc zapaść **przed**
importem. Robienie tego ręcznie przy kilkuset fakturach miesięcznie jest
nierealne, a pomyłka jest kosztowna nieproporcjonalnie do swojej wagi:
zaimportowany dokument idzie dalej do księgowości i do osób rozliczających
koszty, więc błędny rodzaj oznacza fakturę w niewłaściwym rejestrze i
pracę do cofnięcia u kilku osób naraz.

## Jak działa klasyfikacja

enova365 wywołuje cechę **raz, w momencie pobrania pliku z KSeF** — to
standardowy punkt rozszerzenia systemu, uruchamiany zanim ktokolwiek
zobaczy fakturę na liście. Konfiguracja wskazuje tryb „wg algorytmu"
i cechę wyliczaną, która ma dostarczyć wartość.

Wdrożona reguła to łańcuch odwołań:

```
NIP sprzedawcy z faktury  →  kontrahent w kartotece  →  jego typ  →  rodzaj dokumentu
```

| Typ kontrahenta | Rodzaj dokumentu | Ścieżka importu |
|---|---|---|
| dostawca materiałów | zakup materiałowy | dopasowanie do zamówień zakupu, dokument magazynowy |
| pojazdy | zakup samochodowy | import kosztowy, odrębna definicja dokumentu |
| **oba naraz** | **brak rodzaju** — do rozstrzygnięcia przez człowieka | żadna, dopóki operator nie wskaże rodzaju |
| pozostałe typy **oraz brak kontrahenta w kartotece** | zakup kosztowy | import kosztowy |

Cztery właściwości tej konstrukcji są celowe:

- **Klasyfikacja opiera się na dostawcy, nie na treści faktury.** Nie
  próbujemy zgadywać z nazw pozycji — stacja paliw wystawia faktury za
  paliwo, hurtownia stali za materiał. Wyjątki (dostawca sprzedał coś
  spoza swojego profilu) poprawia się ręcznie, ale są rzadkie.
- **Typ kontrahenta jest wielowartościowy.** Jeden podmiot bywa
  jednocześnie klientem i dostawcą; dodanie nowego typu nie kasuje
  poprzednich.
- **Sprzeczne wskazanie nie jest rozstrzygane po cichu.** Dostawca
  oznaczony jednocześnie jako materiałowy i pojazdowy dostaje **pusty
  rodzaj**, nie arbitralnie wybrany jeden z dwóch. Faktura zatrzymuje się
  wtedy na liście jako wymagająca decyzji — lepsze niż zaksięgowanie jej
  w rejestrze wybranym rzutem monety.
- **Nieznany dostawca ląduje w kosztach.** Bezpieczny wybór domyślny:
  dokument powstaje i jest widoczny w ewidencji, zamiast zatrzymywać się
  z błędem. Cena tego wyboru — faktura za materiały od nowego dostawcy
  trafi w złą ścieżkę, dopóki nie założy się jego karty — jest świadomie
  zaakceptowana i obsłużona przeglądem przed importem (niżej).

## Dwie cechy pomocnicze: przegląd bez otwierania faktur

Sama klasyfikacja to za mało — operator musi umieć ją **sprawdzić**,
a otwieranie kilkuset plików po kolei przeczyłoby całej oszczędności.
Dlatego na tej samej liście stoją dwie cechy wyliczane, które wyciągają
na wierzch dokładnie te informacje, które są potrzebne do decyzji:

- **Nazwy pozycji faktury**, sklejone w jedno pole — widać *co* jest na
  fakturze bez otwierania jej podglądu. To główne narzędzie oceny, czy
  rodzaj się zgadza: pozycje typu „profile, śruby, obróbka" przy rodzaju
  kosztowym od razu rzucają się w oczy.
- **Cechy kontrahenta** (typ, centrum kosztów, projekt, pracownik,
  lokalizacja), również sklejone w jedno pole — pokazują z góry, **jak
  koszt zostanie opisany** po imporcie, i czy dostawca ma w ogóle
  uzupełnioną dekretację. Pusto oznacza albo brak cech, albo brak
  kontrahenta w bazie.

Dzięki nim przegląd przed importem sprowadza się do przejrzenia trzech
kolumn na jednym ekranie, z filtrowaniem po rodzaju.

## Korekta klasyfikacji — dwie różne czynności

Rodzaj wylicza się **raz, przy pobraniu**, i to rozróżnienie jest istotne
w codziennej pracy:

| Sytuacja | Co zrobić | Zasięg |
|---|---|---|
| Pojedyncza faktura wyłamuje się z profilu dostawcy | zmiana rodzaju na formularzu pliku (na liście pole jest tylko do odczytu) albo akcją grupową | **ta jedna faktura** |
| Dostawca ma zły typ na karcie albo nie ma go w kartotece | poprawka karty kontrahenta | **kolejne** faktury; już pobrane trzeba poprawić osobno |

Zmiana rodzaju jest możliwa **wyłącznie do momentu importu** — po
utworzeniu dokumentu pole jest zablokowane, bo rodzaj przestaje być
klasyfikacją, a staje się faktem księgowym.

## Co dzieje się wcześniej

- Zapytanie do KSeF o faktury z danego okresu i pobranie paczki do
  enova365 (mechanizm wbudowany w system).
- Kartoteka kontrahentów z uzupełnionym typem — to ona zasila całą
  regułę. Kontrahenci zakładani są po NIP z faktury, więc uzupełnianie
  kartoteki jest naturalną częścią pierwszego przeglądu.

## Co dzieje się później

- Faktury rodzaju **materiałowego** importuje dodatek dopasowujący je do
  zamówień zakupu i budujący opis analityczny wg podziału na projekty.
  → [Import faktur zakupu materiałowego (ZME) z dopasowaniem do zamówień](https://github.com/trynityeu/enova365-import-faktur-ksef-dopasowanie)
- Faktury **kosztowe i samochodowe** importuje dodatek rozpoznający
  dodatkowo wariant faktury (zwykła, korekta, zaliczkowa, rozliczeniowa)
  i budujący dekretację z karty kontrahenta.
  → [Import faktur kosztowych (ZKE) i samochodowych (ZSE)](https://github.com/trynityeu/enova365-import-faktur-ksef-koszty-pojazdy)
- Oba dodatki **pomijają pliki cudzego rodzaju** zamiast je przetwarzać —
  błąd klasyfikacji kończy się czytelnym komunikatem, nie dokumentem
  zaksięgowanym według niewłaściwych reguł.

## Co jest do tego potrzebne

- enova365 z aktywną integracją KSeF oraz zdefiniowanymi rodzajami
  dokumentów KSeF i odpowiadającymi im definicjami dokumentów ewidencji.
- Konfiguracja pobierania ustawiona na wyliczanie rodzaju **wg algorytmu**
  ze wskazaniem cechy klasyfikującej. Tryb dopuszcza kilka sposobów
  ustalania rodzaju (m.in. wg identyfikatora wewnętrznego z faktury albo
  wg przypisania rodzaju do kontrahenta) stosowanych w zadanej kolejności —
  tu wykorzystany jest wariant oparty o cechę wyliczaną, bo pozwala
  zapisać dowolną regułę zamiast korzystać z gotowego przypisania.
- Cecha klasyfikująca musi być typu referencyjnego (zwraca rodzaj
  dokumentu) i mieć algorytm odczytu — inne typy cech system w tym polu
  nie przyjmie.
- Słownik typów kontrahenta uzupełniony o wartości sterujące
  klasyfikacją, dostępny do wielokrotnego wyboru na karcie kontrahenta.

## Zgodność

| | |
|---|---|
| System | enova365 (Soneta sp. z o.o.), wersja referencyjna **2512.9.10** |
| Postać | cechy wyliczane (algorytmiczne) w konfiguracji bazy, nie dodatek DLL |
| Punkt wywołania | pobranie pliku z KSeF (mechanizm systemowy) |
| Rodzaje dokumentów | zakup materiałowy, zakup kosztowy, zakup samochodowy |

## Czego tu nie ma

Treść cech, słowniki wartości, dane handlowe i wszelka konfiguracja
specyficzna dla wdrożenia pozostają poza tym repozytorium. To repo służy
wyłącznie jako publiczny opis mechanizmu.
