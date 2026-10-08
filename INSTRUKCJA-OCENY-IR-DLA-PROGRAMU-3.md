# Instrukcja dla sesji LLM: ocena eksportów PowerCenter i PL/SQL

Wersja 1.0 • 8 października 2026 • dokument samodzielny do przekazania drugiej sesji.

## 1. Twoje zadanie i granice

Masz dostęp do wyników parsera Informatica PowerCenter i parsera Oracle PL/SQL oraz ewentualnie do ich repozytoriów. Sprawdź, czy rzeczywiste eksporty wystarczą do importu do programu nr 3, który połączy przepływy i udostępni ich analizę ludziom oraz LLM. Przedstaw dowody dla wniosków. Nie projektuj rozbudowy parserów, nie zmieniaj ich dokumentacji ani implementacji i nie implementuj programu nr 3. Jeśli czegoś brakuje, opisz konkretną lukę oraz jej wpływ. Nie traktuj odmiennego formatu jako braku danych.

Zacznij od artefaktów, schematów i udokumentowanych interfejsów. Nie analizuj całych kodów obu programów. Wąskie fragmenty kodu czytaj wyłącznie, gdy wynik lub kontrakt nie pozwala rozstrzygnąć konkretnego pytania. Nie uruchamiaj ponownie pełnej analizy, jeżeli gotowe artefakty wystarczają.

Rozróżniaj: wymaganie dokumentacji, zaobserwowany wynik, deklarację implementacji i niepotwierdzoną hipotezę. Istnienie modelu lub testu nie dowodzi, że odpowiadające mu dane znajdują się w eksporcie.

## 2. Cel i podział programów

- A: PowerCenter XML → pełny deterministyczny IR tej technologii i lokalne zależności.
- B: PL/SQL + dostarczone definicje Oracle → pełny deterministyczny IR, źródła i dowody, manifest, diagnostyka oraz ExchangeIR. Dokumentacja B przewiduje także lokalny opis biznesowy LLM; ten zakres pozostaje bez zmian.
- C: import wyników A/B → uzgodnienie tożsamości obiektów → wspólny lineage oraz API nad bazą Neo4j. Raporty dla człowieka, zewnętrzna wizualizacja i rozmowa z LLM mają korzystać z tej warstwy.

Nie zakładaj kolejności PowerCenter → PL/SQL. Możliwy jest kierunek odwrotny, wiele producentów/odbiorców oraz przepływ wyłącznie w jednej technologii. Tabela może być jednocześnie odczytywana i zapisywana. Powiązanie przez wspólną tabelę nie dowodzi kolejności uruchomień ani zgodności zbiorów wierszy.

Baza C ma zapewnić jeden punkt dostępu do wszystkich informacji potrzebnych do wyjaśnienia przepływu: zależności, formuł, warunków, typów, kodu i dowodów oraz luk analizy. W tej ocenie sprawdź dostępność tych informacji; nie narzucaj jeszcze fizycznego modelu Neo4j ani endpointów C.

## 3. Przykład biznesowy, który eksporty mają umożliwić

Oracle zawiera tabelę KLIENT z atrybutami. Dane pochodzą z CRMX, z pliku ładowanego przez pakiet PL/SQL LOADER, przechodzą przez tabele stage, a część kolumn zmienia nazwę. Ładowanie ma logikę Data Vault i porównuje dane z poprzednim ładowaniem. NAZWISKO przechodzi przez kilka funkcji przed zapisem do tabeli docelowej.

KLIENT, NAZWISKO, CRMX i LOADER są nazwami ilustracyjnymi, a nie potwierdzonymi identyfikatorami w dostarczonych wynikach. Wybierz rzeczywistą kolumnę z artefaktów. Jeżeli nie ma przykładu spełniającego wszystkie cechy, sprawdź osobne przypadki i oznacz, czego nie można potwierdzić na tym materiale. Nie wymyślaj brakującej ścieżki.

Pytania, na które C i LLM powinny móc odpowiedzieć:

1. Z jakiego pola pliku i systemu pochodzi wskazana kolumna?
2. Przez jakie tabele, kolumny i operacje przechodzi? Gdzie zmienia nazwę?
3. Jakie funkcje i wyrażenia są stosowane, w jakiej kolejności i z jakimi argumentami?
4. Z jaką tabelą i kolumnami porównywane są dane, po jakich kluczach?
5. Jaki dokładnie warunek wybiera poprzedni stan lub datę?
6. Co decyduje o INSERT/UPDATE lub pominięciu rekordu? Jak wpływają hash, NULL, defaulty i konwersje, jeśli występują w kodzie?
7. Jaki kod potwierdza każdy wniosek? Gdzie analiza pozostaje niepełna?
8. Czy da się zebrać analogiczne informacje dla całej tabeli, z jednym wspólnym kontekstem ładowania i regułami per kolumna?

Nie utożsamiaj poprzedniego ładowania z datą minus jeden dzień. Nie zakładaj konkretnej odmiany Data Vault, zastosowania hashdiff ani sposobu obsługi historii bez dowodów.

## 4. Co wiadomo o miejscach wyników i API

Poniższe ścieżki B i komendy są wymaganiami dokumentacji projektowej, a nie potwierdzeniem bieżącej implementacji. Rzeczywisty katalog może być wybrany parametrem CLI. Korzystaj z manifestu i konfiguracji znalezionej w danym repozytorium.

| Program | Punkt startowy | Status wiedzy |
|---|---|---|
| PowerCenter | `graph.json` w katalogu wynikowym, według użytkownika w `graphs`; obok inne artefakty | Istnienie zgłoszone przez użytkownika; pełna ścieżka i schemat niezweryfikowane |
| PowerCenter | Funkcje `upstream`, `downstream`, `resolve` | Nazwy zgłoszone przez użytkownika; składnia, kierunek krawędzi i rodzaj interfejsu niezweryfikowane |
| PL/SQL | `data/artifacts/<analysis_id>/` | Domyślna lokalizacja z dokumentacji; artefakt zawiera raw/IR/indeksy/manifest |
| PL/SQL | `reports/<run_id>/preflight.json`, `provisional-inventory.json` | Wyniki zablokowanej analizy; nie zastępują finalnego IR |
| PL/SQL | `data/business/<description_id>/` | Oddzielne opisy LLM, jeżeli etap jest wdrożony |
| PL/SQL | ExchangeIR w katalogu wskazanym przez `export-exchange --output` | Brak gwarantowanej nazwy `graph.json` lub `exchange.json` |
| Program C | Neo4j i API | Docelowy cel; nie podano istniejącego adresu, endpointów ani modelu bazy |

### PL/SQL: interfejs CLI określony dokumentacją

Najpierw sprawdź lokalne `plir --help` i pomoc konkretnej podkomendy. Jeżeli CLI nie jest dostępne, nie instaluj i nie uruchamiaj całego programu tylko dla odczytu JSON.

```text
plir version
plir verify --artifact <KATALOG_ARTEFAKTU>
plir export-exchange --artifact <KATALOG_ARTEFAKTU> --output <NOWY_KATALOG>
plir inspect --input <KATALOG_LUB_ZIP_WEJSCIA> --output <NOWY_KATALOG>
plir preflight --input <KATALOG_LUB_ZIP_WEJSCIA> --output <NOWY_KATALOG>
plir analyze --input <KATALOG_LUB_ZIP_WEJSCIA> --output <NOWY_KATALOG>
plir describe --artifact <KATALOG_ARTEFAKTU> --routine-id <ID> --output <NOWY_KATALOG>
```

Do tej oceny preferuj odczyt oraz `verify`. Eksport ExchangeIR wykonuj tylko jeśli potrzebny i dostępny, do nowego katalogu. `inspect`, `preflight`, `analyze` i `describe` pokazano dla orientacji; nie są obowiązkowym przebiegiem oceny. Nie uruchamiaj LLM/providerów ani analizy Oracle w ramach domyślnego sprawdzania eksportów.

Dokumentacja rozróżnia kod wyjścia 0 również dla PARTIAL, 2 dla błędnego wejścia/schematu, 3 dla braków definicji/discovery, 4 dla awarii parsera/storage i 5 dla błędu warstwy biznesowej. Dlatego sam exit 0 nie dowodzi kompletnej semantyki.

### PL/SQL: pliki kontraktów, które warto przeczytać

W katalogu dokumentacji projektu B:

- `00-CONTEXT.md`: zakres i ograniczenia.
- `01-ARCHITECTURE.md`: mapa repozytorium i porty.
- `contracts/IR.md`: pełny IR i minimalny CLI.
- `contracts/EXCHANGE.md`: kontrakt A/B → C; przeczytaj całość, także końcowe doprecyzowania typów i locatorów.
- `contracts/BUSINESS.md`: tylko jeśli oceniasz dostępne opisy LLM.
- `stages/09a.md`: zapis, integralność i kompletność artefaktów.
- `stages/10a.md`, `stages/10b.md`: eksport i testy wymiany.
- `docs/schemas/` względem root repo B: wygenerowane schematy, w tym przewidziany `exchange-v1.json`; sprawdź ich faktyczne istnienie.

Katalog dokumentacji może być nazwany `docs`, `plsql-execution-docs` lub umieszczony inaczej. Nie zakładaj, że ścieżki z poprzedniej sesji są dostępne tutaj.

### Ograniczony odczyt kodu, gdy naprawdę potrzebny

Ścieżki przewidziane dla B, względem root repo:

- `src/plsql_ir/cli.py`: składnia komend i konfiguracja lokalizacji.
- `src/plsql_ir/adapters/storage/json_repository.py`, `application/artifact_service.py` w tym samym pakiecie: organizacja zapisu i odczytu.
- `src/plsql_ir/domain/exchange_models.py`, `application/exchange_service.py`: rzeczywisty eksport wymiany.
- `src/plsql_ir/domain/{schema_models,program_models,sql_models,diagnostics}.py`: tylko wybrany model/pole, którego semantyki nie rozstrzygają schematy.

Porty B z dokumentacji: `ArtifactRepository.read(analysis_id)->Artifact`, `ExchangeExporter.export(Artifact)->ExchangeIR`. To wewnętrzne porty aplikacyjne, nie endpointy HTTP. Nie przedstawiaj ich jako działającego REST API.

Dla A nie ma w tej instrukcji potwierdzonej mapy kodu. Ustal ją z README, konfiguracji entrypoints i schematów. Szukaj definicji `upstream`, `downstream`, `resolve` oraz serializacji `graph.json` wyłącznie wtedy, gdy wynik i pomoc nie wyjaśniają kontraktu. Nie skanuj całego repozytorium pod kątem każdej transformacji.

## 5. Procedura wykonania z małym kontekstem

### Krok 1 — inwentaryzacja

Zidentyfikuj osobno root repo A/B i root artefaktów A/B. Jeśli jest wiele uruchomień, wybierz jawnie analysis/run ID i wersję; nie mieszaj plików z różnych analiz. Zacznij od `rg --files` na wskazanych katalogach wynikowych; ogranicz wyszukiwanie do wyników i dokumentacji. Jeśli nie znasz root, przeczytaj README/config lub lokalną pomoc, zamiast od razu czytać kod parserów.

Zapisz mapę: producent, wersja, analysis ID, ścieżka, rola pliku, format, schema version, status. Nazwy plików i analiz podawaj rzeczywiste. Nie odczytuj całego dużego JSON do kontekstu LLM: użyj lokalnego skryptu/jq do wypisania kluczy, liczności, pól rekordów i kilku identyfikatorów.

### Krok 2 — manifest, schemat i dostępność źródeł

Odczytaj manifest i diagnostykę; sprawdź hashes/refs istniejącym verifierem, jeśli jest dostępny. Sprawdź jednocześnie zachowanie źródeł i rozpoznanie semantyki. Zweryfikuj dostępność raw, offsetów lub innych wskazań źródła. Brak manifestu w A nie dowodzi automatycznie braku przydatności, ale wymaga jawnego ustalenia integralności i wersji inną dostępną metodą.

W B artefakt z brakującymi wymaganymi definicjami Oracle jest BLOCKED; provisional inventory nie wolno importować jako pełnego semantycznego wyniku. PARTIAL z zachowanymi lukami to odrębny przypadek od BLOCKED. Nierozpoznany kod utrudniający discovery może również blokować preflight.

### Krok 3 — jedna rzeczywista kolumna

Wybierz istniejącą kolumnę docelową i znajdź operacje, które ją zapisują. Zbuduj mały pakiet danych: pełna tożsamość obiektu, write/read bindings, wartości, wyrażenia, warunki, wywołania, typy, stany i dowody. Podążaj po ID zamiast wyszukiwać podobne nazwy w całym kodzie.

Rozwiń tylko zależności tej kolumny oraz kontekst konieczny do jej interpretacji: filtr, JOIN, gałąź MERGE, funkcja, parametr, źródło daty, porównanie historyczne. Przy cyklach użyj visited i zachowaj informację o cyklu. Przy limicie rozmiaru oznacz nieprzejrzany zakres; nie ogłaszaj kompletnej ścieżki po jej obcięciu.

Dla A użyj istniejących `upstream/downstream/resolve`, jeżeli pomoc potwierdza ich kontrakt. Zweryfikuj kierunek krawędzi i czy wyniki dotyczą wartości, obiektów czy instancji transformacji. Samo `upstream` nie musi zawierać funkcji i warunków; dociągnij je z powiązanych rekordów IR.

### Krok 4 — połączenie technologii

Znajdź rzeczywisty wspólny obiekt, jeśli jest w dostarczonych wynikach. Porównaj environment, database/connection identity, owner/schema, nazwę obiektu, kolumnę, reguły cytowania/case i edition, jeśli dotyczy. Nazwa połączenia Informatiki nie musi być identyfikatorem tej samej bazy co Oracle database_key. Jeśli potrzebne mapowanie konfiguracji nie jest dostarczone, wynik to unresolved, a nie automatyczne dopasowanie.

Nie łącz tabel tylko po nazwie. Nie scalaj lokalnych ID różnych producentów. Zachowaj analysis IDs i lokalne wystąpienia wartości/operacji. Kolejne UPDATE tej samej kolumny nie są jednym zapisem. Ustal osobno dowód tożsamości fizycznego obiektu i dowód konkretnego przepływu/czasu/partycji danych. Nie twórz fikcyjnej bezwarunkowej ścieżki między każdym producentem i każdym odbiorcą tabeli.

Jeżeli brak wspólnego przykładu, oceniaj kompatybilność strukturalną; oznacz połączenie end-to-end jako NIEZWERYFIKOWANE.

### Krok 5 — ocena pakietu dla LLM

Bez wywoływania modelu przygotuj z rzeczywistych rekordów mały pakiet, który pozwala wyjaśnić wybraną kolumnę: ścieżka, wyrażenia, argumenty funkcji, kontekst porównania, tryb zapisu, dowody i luki. Zachowaj kolejność oraz alternatywne gałęzie. Wspólne funkcje identyfikuj raz i odwołuj się do ich definicji; nie kopiuj całego pakietu PL/SQL dla każdej kolumny.

Oceń, czy dane można wybierać po ID i zakresach, bez ponownego parsowania XML/PLSQL. Odczyt zachowanego raw dla opaque jest prawidłowy; konieczność ponownego parsowania całego źródła, by odtworzyć podstawowe rozpoznane zależności, wymaga zaznaczenia. Nie obiecuj określonego kosztu tokenów bez pomiaru.

## 6. Co zawiera kontrakt ExchangeIR B

ExchangeIR jest neutralnym eksportem faktów, a nie substytutem pełnego IR ani gotowym globalnym grafem. Zgodnie z dokumentacją obejmuje:

- producer i wersję schematu, assets z tożsamością i parent kolumny;
- operations z odczytami/zapisami, kolejnością dzieci, wyrażeniami, warunkami i skutkami;
- values jako wystąpienia odczytu/wyliczenia/zapisu/zmiennej/rowset, typy i expressions;
- dependencies rozróżniające value, filter, join, group, order, control, call, state i rowset;
- control_edges, calls, references oraz type_descriptors;
- evidence, diagnostics i coverage;
- source_ir_locator i source element locators do danych niezdublowanych w wymianie.

Pewność krawędzi: exact/inferred/unknown. Nie wolno podnosić inferred/unknown do exact na podstawie płynnego opisu LLM. Locator powinien pozwalać znaleźć manifest oraz konkretny rekord źródłowego IR; zweryfikuj nie tylko obecność pola, ale rzeczywistą możliwość jego rozwiązania.

A może mieć inny własny format. Dokumentacja B wprost wymaga osobnego adaptera PowerCenter do kontraktu wymiany; nie zakłada automatycznej zgodności istniejącego A. Sprawdź, czy dane A pozwalają na projekcję do tych kategorii. Adapter/importer formatu to inny problem niż brak faktów w eksporcie.

## 7. Macierz oceny i wymagane dowody

Dla każdego wiersza nadaj status: OBECNE, CZĘŚCIOWE, BRAK, NIEZWERYFIKOWANE albo NIE DOTYCZY. Status NIE DOTYCZY wymaga uzasadnienia na podstawie badanego przykładu; nie dowodzi obsługi całej technologii. Wskaż plik i JSON Pointer/ID rekordu albo udokumentowany wynik komendy. Rozdziel A i B.

| Obszar | Co należy sprawdzić |
|---|---|
| Tożsamość | Baza/środowisko/schema, tabela/plik, kolumna, quoted identifiers, połączenia i parametry |
| Pochodzenie | Pole pliku, source qualifier/SQL/loader, stage, zmiany nazw, konkretne powiązania kolumn |
| Formuły i funkcje | Wyrażenia, literal/NULL, argumenty i kolejność, definicje funkcji lub jawny brak |
| Typy | Deklaracje i wynikowe typy, precision/scale/length/char semantics, konwersje i unknown |
| Warunki | WHERE/ON/HAVING, gałęzie, filtry, powiązanie warunku z właściwą operacją |
| Historia i daty | Tabela porównawcza, klucze, predykat poprzedniego stanu, parametry i ich pochodzenie |
| Zapis i stan | INSERT/UPDATE/MERGE/DELETE, defaulty, old/new values, kolejność, alternatywy i row overlap unknown |
| Sterowanie i skutki | Calls, exceptions/transactions/triggers tam, gdzie występują i wpływają na badaną ścieżkę |
| Dowody | Zachowany kod/XML/DDL, rozwiązane locatory, poprawne zakresy źródłowe |
| Niepełność | Opaque/raw, diagnostyka i zakres luki, propagacja niepewności do zależności |
| Import | Wersje, stabilne ID, foreign keys, dostęp do pełnego IR, wymagane mapowania |
| Pobieranie dla LLM | Możliwość zebrania ograniczonego, kompletnego kontekstu wskazanej kolumny |

## 8. Nierozpoznany SQL i opisy LLM

Parser zachowuje fakty, nierozpoznany fragment i wpływającą lukę. Program C może użyć LLM do wyjaśnienia tego fragmentu z otaczającym kontekstem. Interpretacja musi wskazywać źródła i niepewność oraz być oddzielna od deterministycznych faktów. Nie twórz brakujących zależności jako pewnych tylko dlatego, że LLM zasugerował logicznie prawdopodobne wyjaśnienie.

Jeżeli B dostarcza BusinessDescriptionIR, zachowaj classification (code_fact/interpretation/hypothesis), validation/reviewer status, evidence, limitations, questions i wersje/model/context hashes. Opis biznesowy nie zastępuje źródłowego IR. Brak opisu LLM nie oznacza braku semantyki parsera.

Istniejące pełne źródła mogą umożliwić późniejsze wyjaśnienie luki, lecz nie dowodzą już rozpoznanego lineage. Zaznacz różnicę między zachowaniem danych a analizą semantyczną.

## 9. Wymagany wynik tej sesji

Utwórz `OCENA-EKSPORTOW-IR.md` i mały `PRZYKLAD-KOLUMNY.json` z rzeczywistymi ID/locatorami. Pakiet JSON jest pomocniczym materiałem oceny, nie nowym publicznym standardem eksportu. Nie zmieniaj istniejących artefaktów.

Raport ma zawierać, w tej kolejności:

1. Werdykt osobno dla A, B i możliwości połączenia: wystarczające na badanym materiale / warunkowo wystarczające / niewystarczające / niezweryfikowane. Podaj zakres dowodu i brakujące dane.
2. Mapę rzeczywistych lokalizacji artefaktów, wersji, analysis IDs i dostępnych komend/API. Dla REST podaj endpointy wyłącznie, jeśli znalazłeś ich definicję lub specyfikację; nie zgaduj URL.
3. Macierz z sekcji 7 z odwołaniami do rekordów.
4. Jedną odtworzoną ścieżkę kolumny z funkcjami, warunkami i porównaniem historycznym, jeśli te cechy występują. Braki oznacz w odpowiednich miejscach.
5. Wynik uzgadniania wspólnego obiektu albo brak dowodu end-to-end.
6. Listę plików, które C musi importować lub mieć dostępne, oraz dlaczego sam graph.json wystarcza lub nie wystarcza w tym konkretnym eksporcie.
7. Zwięzłe mapowanie rzeczywistych pól A/B na kategorie C/ExchangeIR; oddziel różnicę formatu, brak mapowania konfiguracji i faktyczny brak informacji.
8. Listę luk z wpływem: blokuje połączenie, ogranicza wyjaśnienie, ogranicza kompletność lub tylko prezentację. Bez planu wdrożenia i bez zmian kodu.
9. Wykonane odczyty/weryfikacje i niewykonane sprawdzenia. Brak wyniku B podczas implementacji nie jest FAIL — oznacz NIEZWERYFIKOWANE.

Odbiór tej analizy: każdy wniosek ma dowód albo oznaczoną niepewność; nie pomylono dokumentacji z implementacją; nie zgadywano schematów/endpointów; nie połączono obiektów wyłącznie po nazwach; zachowano wpływ warunków i luk; wskazano dokładnie, co wykorzystać w C, bez analizy całego kodu parserów.

Nie uznawaj tej oceny za dowód równoważności wykonania przyszłego dbt. Dokumentacja biznesowa może powstać z opisanej bazy; generator dbt wymaga odrębnej weryfikacji semantyki Oracle, historii, transakcji i sposobu wykonywania przepływu.

## 10. Polecenie startowe dla drugiej sesji

> Wykonaj ocenę eksportów zgodnie z tym dokumentem. Masz dostęp do wyników parserów. Zacznij od manifestów, schematów i małej ścieżki rzeczywistej kolumny. Odkryj rzeczywiste lokalizacje i interfejsy zamiast je zakładać. Czytaj kod tylko punktowo, aby rozstrzygnąć konkretną niejasność kontraktu. Przygotuj raport i pakiet przykładu; nie rozbudowuj parserów ani nie implementuj programu nr 3. Gdy materiał nie pozwala czegoś sprawdzić, oznacz to jawnie i kontynuuj ocenę pozostałych obszarów.
