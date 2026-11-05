# Ściągawka: pytania i odpowiedzi – HD i ED

Iuliia Kapustinskaia, Amelia Sroczyńska, Grzegorz Szczepanek · 05.11.2026

Odpowiedzi są oparte na liczbach sprawdzonych w danych (MIMIC-IV demo 2.2, eICU-CRD demo 2.0.1, UCI Diabetes 130). Szczegóły i alternatywy: `Rejestr_decyzji`.

---

## A. Pojęcia modelu wielowymiarowego

**P: Co to jest fakt?**  
Zdarzenie, które liczymy. Jedno zdarzenie to jeden wiersz tabeli faktów. U nas: jedno przyjęcie do szpitala (MIMIC), jeden pobyt na OIOM (eICU), jeden wynik HbA1c.

**P: Co to jest miara?**  
Liczba w fakcie, którą sumujemy albo uśredniamy. U nas: czas pobytu w dniach, zgon (0/1), wartość HbA1c, liczba pobytów. Test: czy da się z tego policzyć sumę lub średnią.

**P: Co to jest wymiar?**  
Przekrój, według którego grupujemy fakty („według czego?”). U nas: Czas, Pacjent, Diagnoza, Placówka.

**P: Czym różni się atrybut od miary?**  
Miara jest w tabeli faktów i się ją liczy (czas pobytu). Atrybut jest w tabeli wymiaru i się według niego grupuje (płeć, region). Przykład: „średni czas pobytu (miara) według regionu (atrybut)”.

**P: Co to jest hierarchia?**  
Poziomy w wymiarze od szczegółu do ogółu, które pozwalają zwijać i rozwijać wyniki. U nas: kod diagnozy → grupa (cukrzyca/nadciśnienie) → rozdział ICD oraz system → region → szpital.

**P: Co to jest ziarno?**  
Znaczenie jednego wiersza tabeli faktów. U nas np.: jedno przyjęcie (`hadm_id`) lub jeden pobyt na oddziale (`patientunitstayid`).

**P: Co to jest schemat gwiazdy?**  
Tabela faktów w środku i tabele wymiarów dookoła. Mamy trzy gwiazdy: Przyjęcie MIMIC, Pobyt OIOM eICU, Pomiar HbA1c.

**P: Co to jest konstelacja faktów i wymiar uzgodniony?**  
Konstelacja to kilka tabel faktów dzielących te same wymiary. Wymiar uzgodniony (conformed) to taki wspólny wymiar (u nas Diagnoza i Placówka). Dzięki niemu da się porównać MIMIC z eICU.

**P: Po co tabela mostkowa?**  
Gdy jeden fakt ma wiele wartości wymiaru. Jedno przyjęcie ma wiele rozpoznań (relacja wiele do wielu), więc potrzebna jest tabela łącząca przyjęcia z diagnozami lub flagi w wymiarze.

## B. Hurtownia i architektura

**P: Czym różni się hurtownia od zwykłej bazy (OLTP)?**  
OLTP obsługuje codzienną pracę: dużo małych zapisów, aktualne dane, jedno źródło. Hurtownia służy do analiz: dane historyczne, zintegrowane z wielu źródeł, oczyszczone, mało zagregowane, głównie odczyt.

**P: Co to jest ETL?**  
Extract: pobranie z systemów źródłowych. Transform: oczyszczenie i ujednolicenie (u nas m.in. ICD-9 i ICD-10, nazwy pól, miary). Load: załadowanie do hurtowni. Między etapami są zbiory przejściowe (staging).

**P: ROLAP czy MOLAP?**  
ROLAP: model wielowymiarowy w tabelach relacyjnych (nasz PostgreSQL), bez ograniczeń liczby wymiarów, często do centralnej hurtowni. MOLAP: osobne struktury wielowymiarowe, szybsze, ale trudniej je modyfikować, często do data martów.

**P: Hurtownia a data mart (składnica)?**  
Hurtownia obejmuje całą organizację i trzyma dane mało zagregowane. Data mart jest dla jednego działu i trzyma dane silnie zagregowane, zwykle z jednego źródła (hurtowni).

**P: Jaka jest architektura waszego projektu?**  
MIMIC-IV i eICU (źródła OLTP) → ETL w Pythonie ze zbiorami przejściowymi → hurtownia w PostgreSQL (ROLAP, gwiazdy, konstelacja) → tabele przestawne w Excelu.

## C. Nasz projekt HD i uzasadnienia

**P: Skąd macie dane i czemu akurat te?**  
MIMIC-IV (szpital Beth Israel Deaconess, 100 pacjentów, 275 przyjęć) i eICU-CRD (2520 pobytów OIOM, 186 szpitali). To niezależne systemy szpitalne z własnymi schematami, identyfikatorami i kodowaniem, co uzasadnia ETL.

**P: Dlaczego nie Synthea?**  
To generator, a nie istniejący system informatyczny, a prowadzący wymaga istniejącego systemu.

**P: Dlaczego odrzuciliście CDC PLACES?**  
Dane są silnie zagregowane (jedna wartość na hrabstwo), bez transakcji, a hurtownia ma stać na danych mało zagregowanych. Poza tym żadne źródło szpitalne nie podaje hrabstwa pacjenta, więc nie ma klucza łączenia.

**P: Czy to nie są po prostu gotowe zbiory danych?**  
To najgroźniejsze pytanie. Odpowiedź: to relacyjne bazy szpitalne z wieloma tabelami (MIMIC ma 26), a nie jedna płaska tabela. Uczciwie: są to bazy badawcze zbudowane z systemów szpitalnych. Zapytajcie prowadzącego, czy to akceptuje.

**P: Gdzie tu integracja wielu źródeł, skoro pacjenci się nie pokrywają?**  
Integracja to uzgodnienie wymiarów: inne identyfikatory (`subject_id` vs `uniquepid`), mieszane ICD-9 i ICD-10 (MIMIC: 2313 kodów ICD-10 i 2193 ICD-9), inne modele czasu, inne nazwy pól dla zgonu (`hospital_expire_flag` vs `hospitaldischargestatus`).

**P: Czemu zmieniliście źródła i temat?**  
Temat zostaje (cukrzyca i nadciśnienie). Zmieniły się źródła, bo uwagi prowadzącego wykazały, że generator i dane zagregowane nie spełniają wymogu. Tło porównawcze to teraz porównanie systemów i regionów.

**P: Czemu osobna gwiazda dla każdego źródła?**  
Źródła mają różne ziarno i miary: przyjęcie do szpitala kontra pobyt na oddziale, a HbA1c jest tylko w MIMIC. Wspólne wymiary tworzą konstelację. Słabość: jedna wspólna tabela byłaby prostsza dla zapytań między systemami.

**P: Czemu porównujecie tylko pobyty na OIOM?**  
eICU zawiera wyłącznie OIOM. Porównanie z całymi przyjęciami MIMIC byłoby nieuczciwe (inna populacja), więc w MIMIC bierzemy `icustays`.

**P: Dlaczego wymiar Czas jest względny?**  
Źródła nie mają prawdziwych dat. MIMIC przesuwa daty (lata 2110–2201) i podaje tylko przedział lat (2011–13, 2014–16). eICU ma offsety w minutach od przyjęcia oraz rok wypisu (2014, 2015). Hierarchia dzień → miesiąc → rok nie jest możliwa. Trendów w czasie nie analizujemy.

**P: Czemu region, a nie szpital?**  
W eICU demo jest 186 szpitali po 10–40 pobytów, w żadnym więcej niż 16 z cukrzycą. Region ma od 14 (Northeast) do 125 (South) pobytów z cukrzycą. Szpital zostaje w modelu jako poziom hierarchii, ale nie służy do porównań.

**P: Jak definiujecie cukrzycę i nadciśnienie?**  
Kodami ICD (cukrzyca: E10–E14 lub 250, nadciśnienie: I10–I16 lub 401–405) z mapowaniem ICD-9 i ICD-10. Nie HbA1c, bo nie ma go w eICU. Nie pomiarem ciśnienia, bo w MIMIC jest ambulatoryjne (`omr`), a w eICU z monitorowania OIOM, więc znaczy co innego.

**P: Ilu pacjentów z cukrzycą macie?**  
MIMIC: 35 (111 przyjęć), z nadciśnieniem 68, z oboma 30. W eICU: 346 pobytów z cukrzycą i 327 z nadciśnieniem.

**P: Czemu pytanie o HbA1c jest tylko w MIMIC?**  
eICU nie ma HbA1c (0 wierszy). W MIMIC jest 117 pomiarów u 44 pacjentów, a z cukrzycą i HbA1c tylko 20. To pokaz działania miary, a nie wniosek statystyczny.

**P: Jakie są ograniczenia?**  
Mała liczebność (MIMIC demo 100 pacjentów, Northeast 14 pobytów z cukrzycą), brak kalendarza, brak HbA1c w eICU, dane tylko z OIOM i szpitali (nie populacja ogólna).

## D. Eksploracja danych

**P: Co przewidujecie i na jakich danych?**  
Ponowną hospitalizację pacjentów z cukrzycą: zmienna `readmitted` ze zbioru UCI Diabetes 130 (101 766 rekordów, 50 kolumn). Klasyfikacja nadzorowana, trzy klasy: `<30`, `>30`, `NO`.

**P: Dlaczego trzy klasy, a nie dwie?**  
`>30` i `NO` to różne zdarzenia kliniczne (powrót po ponad 30 dniach i brak powrotu), a scalenie ukrywa informację. Wariant dwuklasowy (`<30` vs reszta) zostaje jako dodatkowe porównanie.

**P: Jaki jest rozkład klas?**  
`NO` 53,91%, `>30` 34,93%, `<30` 11,16%.

**P: Dlaczego nie accuracy?**  
Klasyfikator „zawsze `NO`” ma ok. 54% accuracy bez żadnej wiedzy. Dlatego macro-F1 (średnia F1 po klasach), F1 dla `<30` i macierz pomyłek. Wykład też mówi, że błąd całkowity nie nadaje się do danych niezbalansowanych.

**P: Co to jest F1 i macro-F1?**  
F1 łączy precyzję (ile przewidzianych było trafnych) i czułość (ile trafnych znaleziono). Macro-F1 to średnia F1 po trzech klasach, więc mała klasa `<30` liczy się tak samo jak duża.

**P: Po co walidacja grupowana po `patient_nbr`?**  
W zbiorze jest 71 518 pacjentów, z czego 16 773 ma więcej niż jeden pobyt (do 40), co daje 47 021 rekordów (46%). Przy zwykłej walidacji ten sam pacjent trafiałby do uczenia i testu (wyciek danych).

**P: Czemu usuwacie zgony i hospicjum?**  
Pacjent, który zmarł lub trafił do hospicjum (kody wypisu 11, 13, 14, 19, 20, 21), nie może wrócić do szpitala. Usuwa to 2423 rekordy (zostaje 99 343), z czego 2337 miało `NO`. Anomalia: 43 + 43 z nich ma `<30` i `>30`, czego nie umiemy wyjaśnić.

**P: Jakie modele i dlaczego?**  
Regresja logistyczna jako punkt odniesienia, potem drzewo, las losowy i gradient boosting. Wykład: zacząć od prostych, porównać wiele modeli. „No free lunch”: żaden algorytm nie jest uniwersalny.

**P: Czym różni się parametr od hiperparametru?**  
Parametr model dostraja sam w trakcie uczenia (np. wagi). Hiperparametr ustalamy przed uczeniem (np. głębokość drzewa, liczba drzew).

**P: Grid search, random search czy bayesowska?**  
Random search. Grid search nadaje się do małej liczby przeszukiwań, a my mamy wiele hiperparametrów. Bayesowska jest bardziej złożona i nie jest tu potrzebna. Słabość: wynik zależy od dobrze dobranych przedziałów.

**P: Jak radzicie sobie z niezbalansowaniem?**  
Wagi klas, a oversampling tylko na zbiorze uczącym. Oversampling przed podziałem powoduje wyciek.

**P: Co to jest „definition of done”?**  
Z góry ustalony próg metryki (poprawa macro-F1 względem punktu odniesienia), od którego uznajemy model za wystarczający. Próg wybieramy po policzeniu punktu odniesienia.

## E. Podchwytliwe pytania

**P: Czemu ED jest na innych danych niż HD?**  
Slajdy podają UCI MLR jako przykład zbioru do ED, a MIMIC demo ma 100 pacjentów, co jest za mało na uczenie modelu. Obie części dotyczą cukrzycy.

**P: Co byście zrobili, gdyby MIMIC i eICU zostały odrzucone?**  
Rezerwą jest OpenMRS demo (5000 pacjentów). Wg wyszukiwania dane są prawdziwe, ale zdeanonimizowane z przemieszanym wiekiem i płcią, więc wymiar Pacjent byłby słaby. Nie pobieraliśmy go ani nie sprawdzaliśmy pod kątem cukrzycy.

**P: Co jest największą słabością projektu?**  
Czy zrzut bazy badawczej liczy się jako „istniejący system”, mała liczebność w niektórych komórkach i brak kalendarza. Wszystkie trzy znamy i opisujemy w ograniczeniach.

**P: Co jest „nie do wykonania” w waszym modelu?**  
Analiza trendów w czasie, porównanie pojedynczych szpitali, kontrola HbA1c w eICU i porównanie wartości ciśnienia między systemami.
