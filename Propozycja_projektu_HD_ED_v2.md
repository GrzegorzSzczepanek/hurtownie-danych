# Propozycja projektu – Hurtownie i eksploracja danych (wersja 2)

Iuliia Kapustinskaia, Amelia Sroczyńska, Grzegorz Szczepanek · 05.11.2026

**Zmiany względem wersji z 30.10.2026:** zamiast generatora (Synthea) i zagregowanych wskaźników (CDC PLACES) źródłami są dwa istniejące systemy szpitalne (MIMIC-IV i eICU-CRD). Część ED używa klasyfikacji trójklasowej. Uzasadnienie na końcu.

---

## Część 1: Hurtownia danych

**Temat:** Hurtownia danych o pacjentach szpitalnych z cukrzycą i nadciśnieniem, zintegrowana z dwóch niezależnych systemów szpitalnych USA.

### Systemy źródłowe

| | MIMIC-IV (demo 2.2) | eICU-CRD (demo 2.0.1) |
|---|---|---|
| Co to jest | baza elektronicznej dokumentacji medycznej szpitala Beth Israel Deaconess Medical Center | wielośrodkowa baza oddziałów intensywnej terapii (telemedycyna eICU) |
| Model | relacyjny, 26 tabel (moduły `hosp` i `icu`) | relacyjny, tabele zdarzeń (`patient`, `diagnosis`, `lab`, `hospital`, …) |
| Skala demo | 100 pacjentów, 275 przyjęć, ok. 107 tys. wyników badań | 2520 pobytów OIOM, 1841 pacjentów, 186 szpitali |
| Identyfikatory | `subject_id`, `hadm_id` | `uniquepid`, `patientunitstayid`, `hospitalid` |
| Kodowanie diagnoz | ICD-9 i ICD-10 (ok. 2200 i 2300 wierszy) | ICD-9 z ICD-10 w jednym polu tekstowym + hierarchia tekstowa `diagnosisstring` |
| Czas | daty przesunięte; rok tylko jako przedział (`anchor_year_group`: 2011–13, 2014–16) | offsety w minutach od przyjęcia; rok wypisu 2014 lub 2015; godzina doby |
| Lokalizacja | jeden szpital | `region` (Midwest, South, West, Northeast), `teachingstatus`, `numbedscategory` |
| Licencja | otwarty dostęp (PhysioNet) | ODbL v1.0 |

Wszystkie wymienione liczby policzyliśmy na pobranych plikach.

**Heterogeniczność źródeł** (to ona uzasadnia ETL): różne identyfikatory, różne wersje ICD, różne modele czasu, różne nazwy pól dla tych samych pojęć (zgon: `hospital_expire_flag` vs `hospitaldischargestatus`), brak HbA1c w eICU. Dane z obu systemów nie dotyczą tych samych pacjentów, więc integracja polega na uzgodnieniu wymiarów, a nie na łączeniu rekordów.

Opcjonalny moduł izby przyjęć MIMIC-IV-ED (osobny pakiet) rozważymy, jeśli starczy czasu. Nie sprawdzaliśmy jego zawartości.

### Pytania analityczne

Pytania 1–4 wymagają danych z obu systemów jednocześnie. Porównanie jest ograniczone do pobytów na OIOM (w MIMIC: tabela `icustays`), bo eICU zawiera tylko OIOM.

1. **Czy udział pacjentów z cukrzycą wśród pobytów na OIOM zależy od regionu USA?**
   MIMIC: diagnozy ICD (E10–E14, 250) w pobytach z `icustays` · eICU: diagnozy po regionie szpitala
2. **Jak często cukrzyca współwystępuje z nadciśnieniem i czy jest to podobne w obu systemach?**
   MIMIC: ICD-10 I10–I16, ICD-9 401–405 · eICU: te same grupy po ujednoliceniu kodów
3. **Czy śmiertelność szpitalna pacjentów z cukrzycą różni się od pozostałych, w podziale na system i region?**
   MIMIC: `hospital_expire_flag` · eICU: `hospitaldischargestatus`
4. **Czy pobyt na OIOM jest dłuższy dla pacjentów z cukrzycą i nadciśnieniem niż dla pozostałych?**
   MIMIC: długość pobytu w `icustays` · eICU: `unitdischargeoffset`
5. **(tylko MIMIC, ograniczone liczebnościowo)** Jaki odsetek pacjentów z cukrzycą ma HbA1c poniżej 7%?
   MIMIC: tabela `labevents` (`itemid` 50852 – „% Hemoglobin A1c”). W demo jest 20 pacjentów z cukrzycą i wynikiem HbA1c, więc traktujemy to jako przykład działania miary, nie jako wniosek statystyczny.

### Model wielowymiarowy

Dla każdego źródła powstaje osobny schemat gwiazdy. Gwiazdy dzielą wymiary uzgodnione (conformed), co daje konstelację faktów.

- **Gwiazda 1: Przyjęcie MIMIC** (ziarno: jedno przyjęcie do szpitala, `hadm_id`)
- **Gwiazda 2: Pobyt OIOM eICU** (ziarno: jeden pobyt na oddziale, `patientunitstayid`)
- **Gwiazda 3: Pomiar HbA1c MIMIC** (ziarno: jeden wynik badania)

| Pojęcie | Gwiazda 1 | Gwiazda 2 | Gwiazda 3 |
|---|---|---|---|
| **Fakt** | Przyjęcie | Pobyt OIOM | Pomiar HbA1c |
| **Miary** | liczba przyjęć, czas pobytu (dni), zgon (0/1), liczba rozpoznań | liczba pobytów, czas pobytu na OIOM (dni), zgon (0/1) | wartość HbA1c (%), „w normie” (0/1, < 7%), liczba pomiarów |
| **Wymiary** | Czas, Pacjent, Diagnoza, Placówka | Czas, Pacjent, Diagnoza, Placówka | Czas, Pacjent, Placówka |

**Wymiary uzgodnione i ich atrybuty**

| Wymiar | Atrybuty | Hierarchia |
|---|---|---|
| **Czas** | grupa lat / rok wypisu, dzień pobytu (względny), godzina doby, pora doby | okres (2011–13, 2014–16, 2014, 2015) → rok → dzień pobytu → godzina |
| **Pacjent** | płeć, wiek, grupa wiekowa, rasa/pochodzenie (jeśli dostępne w obu) | pacjent → grupa wiekowa |
| **Diagnoza** | kod, wersja klasyfikacji (ICD-9/ICD-10), nazwa, grupa (cukrzyca / nadciśnienie / inne) | kod → grupa → rozdział ICD |
| **Placówka** | system źródłowy, region, status szpitala uczącego, kategoria wielkości (liczba łóżek), szpital | system → region → szpital |

Uwagi do modelu:
- **Czas jest względny**, bo źródła nie podają prawdziwych dat kalendarzowych. Hierarchia dzień → miesiąc → rok nie jest możliwa. Opisujemy to jako cechę źródeł, nie błąd modelu.
- **Diagnoza** jest w relacji wiele do wielu z faktem (jedno przyjęcie ma wiele rozpoznań). Rozwiązujemy to tabelą mostkową lub flagami cukrzyca/nadciśnienie w wymiarze Pobyt.
- **Placówka** w eICU ma 18 szpitali bez regionu i część bez liczby łóżek. Przypisujemy im wartość „nieznany”.
- Poziom „szpital” jest zachowany w modelu, ale **nie służy do porównań**: w eICU demo przypada 10–40 pobytów na szpital (maks. 16 z cukrzycą), więc porównujemy regiony.

### Architektura (zgodnie z wykładem)

MIMIC-IV + eICU-CRD (warstwa źródłowa OLTP) → ETL w Pythonie z obszarem przejściowym (ekstrakcja, ujednolicenie ICD-9/ICD-10, mapowanie nazw, wyliczenie miar) → hurtownia relacyjna w PostgreSQL (ROLAP, schemat gwiazdy, konstelacja) → wizualizacja w Excelu (tabele przestawne).

---

## Część 2: Eksploracja danych

**Temat:** Przewidywanie ponownej hospitalizacji pacjentów z cukrzycą.

**Zbiór danych:** UCI Machine Learning Repository, „Diabetes 130-US Hospitals for Years 1999–2008”: 101 766 hospitalizacji, 47 cech, licencja CC BY 4.0.

**Zadanie:** klasyfikacja nadzorowana **trójklasowa**. Zmienna celu `readmitted` ma trzy wartości:
- `<30` – powrót do szpitala w ciągu 30 dni,
- `>30` – powrót po więcej niż 30 dniach,
- `NO` – brak powrotu.

Wariant dwuklasowy (`<30` vs reszta) traktujemy jako dodatkowe porównanie, nie jako zadanie podstawowe.

### Plan eksperymentu

| Element | Decyzja | Uzasadnienie |
|---|---|---|
| Metryki | macro-F1, F1 dla klasy `<30`, macierz pomyłek, porównanie z klasyfikatorem większościowym | accuracy jest mylące dla danych niezbalansowanych (`NO` 53,9%, `>30` 34,9%, `<30` 11,2%; klasyfikator większościowy ma accuracy ok. 54%) |
| Walidacja | walidacja krzyżowa grupowana po `patient_nbr` | ten sam pacjent ma w zbiorze wiele pobytów, zwykła walidacja powodowałaby wyciek |
| Czyszczenie | usunięcie pobytów zakończonych zgonem lub przekazaniem do hospicjum | takie przypadki nie mogą się zakończyć ponowną hospitalizacją (kody `discharge_disposition_id` 11, 13, 14, 19, 20, 21; usuwa to 2423 z 101 766 rekordów) |
| Modele | regresja logistyczna (punkt odniesienia), drzewo decyzyjne, las losowy, gradient boosting | od prostych do złożonych, zgodnie z wykładem |
| Hiperparametry | random search w uzasadnionych przedziałach (np. głębokość drzewa, liczba drzew, współczynnik uczenia) | duża liczba kombinacji, grid search byłby zbyt kosztowny |
| Niezbalansowanie | wagi klas; porównanie z oversamplingiem tylko na zbiorze uczącym | oversampling na całości zbioru powoduje wyciek |
| „Definition of done” | ustalamy z góry minimalną poprawę macro-F1 względem punktu odniesienia, którą uznajemy za istotną | zgodnie z wykładem o planowaniu eksperymentu |
| Wnioski | opcjonalnie test statystyczny różnic między modelami | |

---

## Uzasadnienie decyzji

| Decyzja | Dlaczego |
|---|---|
| Rezygnacja z Synthea | to generator, a nie istniejący system informatyczny |
| Rezygnacja z CDC PLACES | dane są silnie zagregowane (jedna wartość na hrabstwo), bez danych transakcyjnych. Hurtownia ma stać na danych niezagregowanych. Dodatkowo żadne źródło szpitalne nie podaje hrabstwa pacjenta (dane są zanonimizowane), więc nie ma klucza łączenia |
| MIMIC-IV + eICU-CRD | rzeczywiste, niezależne systemy szpitalne z własnymi schematami, różnymi identyfikatorami i kodowaniem, co uzasadnia ETL i integrację |
| Zostajemy przy cukrzycy i nadciśnieniu | zachowujemy temat zgłoszony 30.10. Zmienia się tło porównawcze: zamiast wskaźników hrabstw jest porównanie systemów i regionów |
| Klasyfikacja trójklasowa | `>30` i `NO` to różne zdarzenia kliniczne, scalenie ich ukrywałoby informację |

## Ograniczenia i punkty do potwierdzenia

1. **Czy zrzut bazy badawczej (MIMIC-IV, eICU) jest akceptowany jako „istniejący system”?** To są bazy zbudowane z systemów szpitalnych, a nie same systemy. Chcemy to potwierdzić z prowadzącym.
2. **Liczebność:** MIMIC demo ma 35 pacjentów z cukrzycą (20 z wynikiem HbA1c). eICU ma 346 pobytów z cukrzycą, ale brak wyników HbA1c. Porównania regionów obejmują Northeast tylko 14 pobytów z cukrzycą.
3. **Ciśnienie krwi:** w MIMIC jest w tabeli `omr` (zapisy ambulatoryjne), a w eICU w `vitalAperiodic` (pomiar nieinwazyjny na OIOM, 2328 z 2520 pobytów) i w `vitalPeriodic` (inwazyjny, 430 pobytów). Kontekst pomiaru jest inny, a dane zawierają wartości 0 do usunięcia, więc nadciśnienie definiujemy rozpoznaniem, a pomiar traktujemy jako opcjonalną miarę.
4. **Brak kalendarza:** wymiar Czas jest względny, więc analizy trendów w czasie nie są możliwe.
5. Dane z MIMIC-IV i eICU opisują pacjentów OIOM i szpitalnych, więc wnioski nie przenoszą się na populację ogólną.
