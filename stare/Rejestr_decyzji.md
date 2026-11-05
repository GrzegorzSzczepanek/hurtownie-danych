# Rejestr decyzji projektowych – HD i ED

Iuliia Kapustinskaia, Amelia Sroczyńska, Grzegorz Szczepanek · 05.11.2026

Do każdej decyzji: **co wybraliśmy**, **jakie alternatywy odrzuciliśmy i dlaczego**, **jaka jest słabość** (czego prowadzący może się czepić). Liczby pochodzą z pobranych plików demo MIMIC-IV 2.2 i eICU-CRD 2.0.1. Rzeczy, których nie sprawdziliśmy, są oznaczone jako **niezweryfikowane**.

Zasada prowadzącego: *jeśli robimy A, a nie B, musimy umieć udowodnić dlaczego.* Poniżej jest to „dlaczego”.

---

## Część HD: źródła danych

| # | Decyzja | Rozważone i odrzucone | Dlaczego tak | Słabość / obrona |
|---|---|---|---|---|
| 1 | **Źródła: MIMIC-IV (demo) + eICU-CRD (demo)** | **Synthea** (generator sztucznych pacjentów) | Prowadzący: ma nie być generatora, ma być istniejący system informatyczny. Dane syntetyczne nie mają własnego schematu operacyjnego | |
| | | **CDC PLACES** jako główne źródło | Dane silnie zagregowane (jedna wartość na hrabstwo), bez transakcji. Wykład: hurtownia przechowuje dane mało zagregowane, silnie zagregowane są cechą składnicy. Do tego żadne źródło szpitalne nie podaje hrabstwa pacjenta, więc nie ma klucza łączenia | Zmienia to zgłoszony temat (patrz #3) |
| | | **Diabetes 130 (UCI)** jako źródło HD | To jedna płaska tabela z jednego zbioru, a prowadzący wymaga wielu baz. Zostaje jako zbiór do części ED | |
| | | **OpenMRS demo** (5000 pacjentów) | Jest bliżej „prawdziwego systemu” (dump bazy działającego oprogramowania). Wg wyników wyszukiwania oryginalne dane demo pochodzą z prawdziwych danych, mocno zdeanonimizowanych: **imiona, wiek i płeć są przemieszane**, a tekst swobodny usunięty. Osobno aplikacja referencyjna potrafi generować pacjentów demo. Przemieszany wiek i płeć psują wymiar Pacjent (grupa wiekowa, płeć) | Rezerwa, jeśli prowadzący odrzuci MIMIC/eICU. **Częściowo zweryfikowane:** strona dokumentacji nie załadowała się w całości, a samego dumpu nie pobieraliśmy. Nie wiemy, czy zawiera cukrzycę, nadciśnienie ani HbA1c |
| | | **Pełny MIMIC-IV** | Wymaga szkolenia i umowy na PhysioNet, więc niedostępny na czas | Przez to demo ma tylko 100 pacjentów |
| | | **MIMIC-IV-ED** (izba przyjęć) jako trzecie źródło | To moduł tej samej bazy (łączy się po `subject_id`), więc nie jest niezależnym źródłem. Nie sprawdzaliśmy jego zawartości | Opcjonalnie, jeśli starczy czasu |
| 2 | **Dwa niezależne systemy** | jedno źródło | Wymóg: odpowiedź ma wymagać kilku źródeł. Pacjenci się nie pokrywają, więc integracja = uzgodnienie wymiarów, nie łączenie rekordów | Prowadzący może zapytać, gdzie „prawdziwa” integracja. Odpowiedź: różne ID, ICD-9/ICD-10, różne modele czasu, różne nazwy pól. **Główne ryzyko:** MIMIC i eICU to bazy badawcze zbudowane z systemów szpitalnych, a nie same systemy, co trzeba potwierdzić u prowadzącego |
| 3 | **Zostajemy przy temacie cukrzyca + nadciśnienie** | zmiana tematu | Temat był już zgłoszony, a zmiana jest ryzykowna. Domena szpitalna pasuje do obu części projektu | Znika „tło regionalnych wskaźników z CDC”. Zastępuje je porównanie systemów i regionów. Trzeba to powiedzieć wprost |
| 4 | **Porównujemy tylko pobyty na OIOM** | wszystkie przyjęcia MIMIC | eICU zawiera tylko OIOM. Porównanie całych przyjęć MIMIC z OIOM eICU byłoby nieuczciwe (inna populacja). W MIMIC bierzemy `icustays` | Zmniejsza liczebność MIMIC (jeszcze nie policzona) |

## Część HD: model

| # | Decyzja | Rozważone i odrzucone | Dlaczego tak | Słabość / obrona |
|---|---|---|---|---|
| 5 | **Osobna gwiazda dla każdego źródła + wymiary uzgodnione (konstelacja)** | jedna wspólna tabela faktów „Pobyt” z kolumną źródła | Prowadzący pytał o osobne gwiazdy dla każdego zbioru. Źródła mają różne ziarno i miary (przyjęcie do szpitala vs pobyt na oddziale, HbA1c tylko w MIMIC) | Wspólna tabela byłaby prostsza dla zapytań międzysystemowych. Ratują nas wymiary uzgodnione |
| 6 | **Wymiar Czas jest względny** (grupa lat / rok wypisu, dzień pobytu, godzina doby) | wymiar kalendarzowy dzień → miesiąc → rok | **Sprawdzone:** MIMIC przesuwa daty (lata 2110–2201) i podaje tylko przedział `anchor_year_group` (2011–13, 2014–16). eICU ma offsety w minutach i rok wypisu (2014, 2015). Kalendarza z tego nie zbudujemy | Brak analizy trendów w czasie. To cecha źródeł, nie błąd modelu |
| 7 | **Placówka: porównujemy regiony, nie szpitale** | porównanie szpital vs szpital | **Sprawdzone:** eICU demo ma 186 szpitali po 10–40 pobytów (maks. 16 z cukrzycą w jednym szpitalu). Statystycznie bez sensu. Region ma 14–125 pobytów z cukrzycą | Northeast to tylko 14 pobytów z cukrzycą. 18 szpitali nie ma regionu (wartość „nieznany”, 210 pobytów) |
| 8 | **Diagnozy: tabela mostkowa lub flagi cukrzyca/nadciśnienie** | jedna diagnoza na fakt | Jedno przyjęcie ma wiele rozpoznań (relacja wiele do wielu) | Tabela mostkowa komplikuje zapytania, flagi tracą szczegóły. Wybór zostawiamy do implementacji |
| 9 | **Diagnoza definiowana kodami ICD (z mapowaniem ICD-9 i ICD-10)** | definicja po wyniku HbA1c | HbA1c nie ma w eICU (0 wierszy). Kody są w obu źródłach. MIMIC ma mieszankę: 2313 kodów ICD-10 i 2193 kodów ICD-9 | Kod rozpoznania nie równa się rzeczywistej chorobie. Mapowanie ICD-9 → ICD-10 jest przybliżone |
| 10 | **Nadciśnienie definiowane rozpoznaniem** (pomiar ciśnienia tylko jako opcjonalna miara) | definicja po pomiarze ciśnienia | **Sprawdzone:** pomiary są w obu systemach, ale znaczą co innego. MIMIC: tabela `omr` (866 pomiarów, 70 pacjentów, zapisy ambulatoryjne). eICU: `vitalAperiodic` (nieinwazyjne ciśnienie, 2328 z 2520 pobytów) i `vitalPeriodic` (inwazyjne, tylko 430 pobytów). To monitorowanie na OIOM, nie kontrola ciśnienia. Dane zawierają wartości 0 (szum) | Porównanie wartości ciśnienia między systemami byłoby mylące (inny kontekst pomiaru), więc kontrolę ciśnienia tylko opcjonalnie, z czyszczeniem zer |
| 11 | **Pytanie o HbA1c tylko w MIMIC (pytanie 5)** | pytanie o HbA1c w obu systemach | eICU nie ma HbA1c. W MIMIC jest 20 pacjentów z cukrzycą i wynikiem | Zbyt mało na wniosek statystyczny. Traktujemy jako pokaz działania miary |
| 12 | **PostgreSQL (ROLAP), ETL w Pythonie, Excel (tabele przestawne)** | MOLAP | Slajdy: ROLAP to relacyjne odwzorowanie modelu wielowymiarowego, bez ograniczeń liczby wymiarów, często do centralnej hurtowni. Excel jest wymieniony w wymaganiach prowadzącego | |

## Część ED

| # | Decyzja | Rozważone i odrzucone | Dlaczego tak | Słabość / obrona |
|---|---|---|---|---|
| 13 | **Zbiór: UCI Diabetes 130** (101 766 hospitalizacji) | eksploracja na MIMIC / eICU | Slajdy podają UCI MLR jako przykład. MIMIC demo ma 100 pacjentów, co jest za mało na uczenie modelu | Dwie części projektu opierają się na różnych danych. Obie dotyczą cukrzycy |
| 14 | **Trzy klasy: `<30`, `>30`, `NO`** | dwie klasy (`<30` vs reszta) | Prowadzący chce 3 klas. `>30` i `NO` to różne zdarzenia kliniczne, scalenie ukrywa informację. Wariant dwuklasowy zostaje jako dodatkowe porównanie | Trudniejsze zadanie, niższe wyniki. **Sprawdzone** (101 766 rekordów): `NO` 53,91%, `>30` 34,93%, `<30` 11,16%. Klasa `<30` jest mała, więc F1 dla niej będzie niski |
| 15 | **Metryki: macro-F1, F1 dla `<30`, macierz pomyłek, punkt odniesienia** | accuracy | Wykład: błąd całkowity nie jest właściwą metryką dla danych niezbalansowanych | |
| 16 | **Walidacja krzyżowa grupowana po `patient_nbr`** | zwykła walidacja krzyżowa | **Sprawdzone:** 71 518 pacjentów, 16 773 z więcej niż jednym pobytem (do 40), co daje 47 021 rekordów (46%). Przy zwykłym podziale ten sam pacjent byłby w uczeniu i teście (wyciek) | Zmniejsza liczbę efektywnych próbek |
| 17 | **Usuwamy pobyty zakończone zgonem lub hospicjum** | zostawienie wszystkich | Taki pacjent nie może wrócić do szpitala, więc to szum w klasie `NO` | **Sprawdzone** w `IDS_mapping.csv`: kody 11 (Expired), 13, 14 (Hospice), 19, 20, 21 (Expired, hospicjum). Usuwa to 2423 rekordów (zostaje 99 343), z czego 2337 miało `NO`. 43+43 z nich ma jednak `<30`/`>30` (anomalia w danych, nie umiemy jej wyjaśnić) |
| 18 | **Modele: regresja logistyczna → drzewo → las losowy → gradient boosting** | tylko jeden złożony model | Wykład: zacząć od prostych, porównać wiele modeli. Twierdzenie „no free lunch”: żaden algorytm nie jest uniwersalny | |
| 19 | **Hiperparametry: random search** | grid search; optymalizacja bayesowska | Wykład: grid search tylko przy małej liczbie przeszukiwań. Random search ogranicza przestrzeń. Bayesowska jest zaawansowana i nie jest potrzebna na tym etapie | Wynik zależy od dobrze dobranych przedziałów |
| 20 | **Niezbalansowanie: wagi klas, oversampling tylko na zbiorze uczącym** | oversampling na całości | Oversampling przed podziałem powoduje wyciek | |
| 21 | **„Definition of done” ustalona z góry** | brak progu | Wykład: trzeba ustalić, jaka wartość metryki uznaje model za wystarczający | Próg do wybrania po policzeniu punktu odniesienia |

---

## Najsłabsze miejsca, na które trzeba się przygotować

1. **„To bazy badawcze, nie systemy”** (#1, #2). Najgroźniejsze. Dobrze zapytać prowadzącego, zanim zbudujemy ETL.
2. **Zmiana tematu** (#3). Powiedzieć wprost i uzasadnić uwagami o zagregowanych danych.
3. **Mała liczebność** (#7, #11): Northeast 14 pobytów, HbA1c 20 pacjentów. Zbiór UCI: klasa `<30` to 11,16%.
4. **Czas względny** (#6). Zadbać, żeby nie obiecywać analiz trendów.
5. **Częściowo zweryfikowane**: OpenMRS (#1), dump nie był pobierany ani sprawdzany pod kątem cukrzycy i nadciśnienia.
