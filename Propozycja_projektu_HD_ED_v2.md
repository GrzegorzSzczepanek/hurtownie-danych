# Propozycja projektu – Hurtownie i eksploracja danych

Iuliia Kapustinskaia, Amelia Sroczyńska, Grzegorz Szczepanek · 5.11.2026

Ta wersja zastępuje propozycję z 30 października. W części pierwszej zmieniliśmy źródła danych, ponieważ generator Synthea i zagregowane wskaźniki CDC nie spełniają wymagania, by źródłem był istniejący system informatyczny. W części drugiej zadanie jest teraz klasyfikacją na trzy klasy, a nie na dwie.

## Część 1: Hurtownia danych

**Temat.** Hurtownia danych o pacjentach szpitalnych z cukrzycą i nadciśnieniem, zbudowana z dwóch niezależnych systemów szpitalnych z USA.

### Źródła danych

Korzystamy z dwóch publicznie dostępnych baz szpitalnych w wersjach demonstracyjnych. MIMIC-IV pochodzi ze szpitala Beth Israel Deaconess Medical Center. Jest to relacyjna baza z 26 tabelami, a wersja demo obejmuje 100 pacjentów i 275 przyjęć. eICU-CRD zawiera dane z oddziałów intensywnej terapii (OIOM) w wielu szpitalach. W wersji demo jest to 2520 pobytów 1841 pacjentów w 186 szpitalach. MIMIC-IV udostępnia PhysioNet, a eICU-CRD działa na licencji ODbL 1.0.

Systemy różnią się w sposób, który wymaga uzgadniania danych podczas ETL:

| | MIMIC-IV | eICU-CRD |
|---|---|---|
| Identyfikatory | `subject_id`, `hadm_id` | `uniquepid`, `patientunitstayid`, `hospitalid` |
| Diagnozy | kody ICD-9 (2193 wiersze) i ICD-10 (2313) | kody ICD-9 i ICD-10 w jednym polu tekstowym oraz tekstowa hierarchia `diagnosisstring` |
| Czas | daty przesunięte, rok znany tylko jako przedział (2011–2013 lub 2014–2016) | przesunięcia w minutach od przyjęcia, rok wypisu (2014 lub 2015) |
| Lokalizacja | jeden szpital | region, status szpitala uczącego i kategoria wielkości |
| Zgon | `hospital_expire_flag` | `hospitaldischargestatus` |

Pacjenci obu baz nie pokrywają się, więc integracja nie polega na łączeniu rekordów tej samej osoby. Polega na sprowadzeniu obu źródeł do wspólnych wymiarów: jednej klasyfikacji chorób, wspólnej definicji pobytu i wspólnego opisu miejsca. Liczby w tej propozycji policzyliśmy na pobranych plikach.

### Pytania analityczne

Każde pytanie wymaga danych z obu systemów jednocześnie, bo korzysta z miar i wymiarów, które powstają dopiero po ich połączeniu w hurtowni. Ograniczamy się do pobytów na OIOM (w MIMIC wybieramy je z tabeli `icustays`), bo eICU zawiera tylko takie pobyty i inaczej porównanie byłoby nierzetelne.

1. Czy odsetek pacjentów z cukrzycą wśród pobytów na OIOM w szpitalu z MIMIC (Boston) mieści się w zakresie odsetków obserwowanych w regionach USA według eICU? Źródła: diagnozy i pobyty z MIMIC oraz diagnozy i region szpitala z eICU.
2. Jak często cukrzyca współwystępuje z nadciśnieniem i czy w obu systemach jest to podobnie często? Źródła: kody ICD z MIMIC i z eICU po sprowadzeniu do wspólnych grup chorób.
3. Czy śmiertelność szpitalna pacjentów z cukrzycą różni się od śmiertelności pozostałych, osobno dla każdego systemu? Źródła: `hospital_expire_flag` z MIMIC i `hospitaldischargestatus` z eICU, a w eICU dodatkowo w podziale na regiony.
4. Czy pacjenci z cukrzycą i nadciśnieniem leżą na OIOM dłużej niż pozostali? Źródła: długość pobytu z `icustays` (MIMIC) i z `unitdischargeoffset` (eICU).

### Model wielowymiarowy

Dla każdego źródła budujemy osobny schemat gwiazdy, a gwiazdy łączymy wspólnymi wymiarami w konstelację faktów.

| Gwiazda | Fakt i ziarno | Miary | Wymiary |
|---|---|---|---|
| Przyjęcie (MIMIC) | jedno przyjęcie do szpitala | liczba przyjęć, czas pobytu w dniach, zgon (0/1), liczba rozpoznań | Czas, Pacjent, Diagnoza, Placówka |
| Pobyt OIOM (eICU) | jeden pobyt na oddziale | liczba pobytów, czas pobytu w dniach, zgon (0/1) | Czas, Pacjent, Diagnoza, Placówka |

Atrybuty i hierarchie wymiarów wspólnych:

| Wymiar | Atrybuty | Hierarchia |
|---|---|---|
| Czas | rok lub przedział lat, dzień pobytu, godzina doby | przedział lat, rok, dzień pobytu, godzina |
| Pacjent | płeć, wiek, grupa wiekowa | pacjent, grupa wiekowa |
| Diagnoza | kod, wersja klasyfikacji (ICD-9 lub ICD-10), nazwa, grupa (cukrzyca, nadciśnienie, inne) | kod, grupa, rozdział ICD |
| Placówka | system źródłowy, region, status szpitala uczącego, kategoria wielkości | system, region, szpital |

Kilka uwag do modelu. Wymiar Czas jest względny, ponieważ żadne ze źródeł nie podaje prawdziwych dat, więc hierarchia dzień–miesiąc–rok nie jest możliwa i nie analizujemy trendów w czasie. Jedno przyjęcie ma zwykle wiele rozpoznań, dlatego diagnozy podłączamy do faktu tabelą mostkową albo flagami (cukrzyca, nadciśnienie). W eICU 18 szpitali nie ma przypisanego regionu, więc dostają wartość „nieznany”. Poziom szpitala zostaje w hierarchii, ale nie służy do porównań, bo w demo eICU na jeden szpital przypada od 10 do 40 pobytów. Porównujemy regiony.

Cukrzycę i nadciśnienie definiujemy kodami ICD z mapowaniem między wersjami 9 i 10. Pomiary ciśnienia występują w obu bazach, ale w MIMIC są to zapisy ambulatoryjne (tabela `omr`), a w eICU monitorowanie na OIOM (`vitalAperiodic`, `vitalPeriodic`, z wartościami zerowymi do usunięcia). Ich wartości nie są porównywalne, więc traktujemy je co najwyżej jako dodatkową miarę.

Najwyższy poziom hierarchii diagnozy, czyli rozdział, bierzemy z własnej tabeli `slowniki/rozdzialy_icd.csv`. Słownik dołączony do MIMIC zawiera kody i ich opisy, ale nie ma rozdziałów, a eICU nie ma słownika wcale. Podział na rozdziały w ICD-9 i ICD-10 jest różny (na przykład choroby oczu i uszu są w ICD-10 osobnymi rozdziałami, a w ICD-9 częścią rozdziału o układzie nerwowym), więc zbudowaliśmy wspólną listę dwudziestu rozdziałów, do której przypisujemy zakresy kodów z obu wersji. Tabela opiera się na strukturze klasyfikacji ICD-9-CM i ICD-10-CM i obejmuje wszystkie kody, które występują w naszych danych: 109 775 kodów słownika MIMIC i 42 135 kodów w eICU.

### Architektura

Bazy MIMIC-IV i eICU stanowią warstwę źródłową. Dane przechodzą przez obszar przejściowy, w którym w Pythonie ujednolicamy kody chorób, nazwy pól i miary, a następnie trafiają do hurtowni relacyjnej w PostgreSQL (ROLAP). Wyniki prezentujemy w tabelach przestawnych w Excelu.

## Część 2: Eksploracja danych

**Temat.** Przewidywanie ponownej hospitalizacji pacjentów z cukrzycą.

**Zbiór.** UCI Machine Learning Repository, „Diabetes 130-US Hospitals for Years 1999–2008”: 101 766 hospitalizacji, 50 kolumn, licencja CC BY 4.0. Wybraliśmy go z kilku powodów. Dotyczy tej samej choroby co część pierwsza, więc cały projekt ma jeden temat. Zmienna `readmitted` ma od początku trzy wartości, tak jak chce prowadzący, i opisuje rzeczywisty problem kliniczny, czyli ponowną hospitalizację. Zbiór jest wystarczająco duży (101 766 rekordów, 71 518 pacjentów, a najmniejsza klasa `<30` ma ponad 11 tysięcy przykładów) i zawiera wiele cech (wiek, leki, wyniki badań, liczba wcześniejszych wizyt), więc można sensownie porównać kilka modeli. Jest publiczny, a slajdy do przedmiotu wymieniają UCI jako typowe miejsce wyboru danych. Nie użyliśmy w tym celu MIMIC, bo wersja demo ma tylko 100 pacjentów, co nie wystarcza do uczenia modelu.

**Zadanie.** Klasyfikacja nadzorowana na trzy klasy zmiennej `readmitted`: ponowne przyjęcie w ciągu 30 dni (`<30`, 11,2% rekordów), po więcej niż 30 dniach (`>30`, 34,9%) i brak ponownego przyjęcia (`NO`, 53,9%). Wariant dwuklasowy (`<30` kontra pozostałe) przeprowadzimy dodatkowo, żeby porównać wyniki.

**Plan eksperymentu.** Ponieważ klasy są niezbalansowane, a klasyfikator zawsze wskazujący `NO` osiąga ok. 54% trafności, główną miarą jest macro-F1, uzupełniona o F1 dla klasy `<30` i macierz pomyłek. Z danych usuwamy pobyty zakończone zgonem lub przekazaniem do hospicjum (kody wypisu 11, 13, 14, 19, 20 i 21; razem 2423 rekordy), ponieważ taki pacjent nie może wrócić do szpitala.

Jeden pacjent ma w zbiorze zwykle kilka pobytów: 16 773 spośród 71 518 pacjentów ma więcej niż jeden, a ich pobyty stanowią 46% rekordów. Dlatego walidację krzyżową dzielimy według `patient_nbr`, tak aby ten sam pacjent nie trafiał jednocześnie do zbioru uczącego i testowego. Porównujemy regresję logistyczną (punkt odniesienia), drzewo decyzyjne, las losowy i gradient boosting. Hiperparametry dobieramy losowym przeszukiwaniem (random search) w przedziałach określonych dla każdego modelu, bo siatka wszystkich kombinacji byłaby zbyt kosztowna. Niezbalansowanie klas uwzględniamy wagami klas. Oversampling stosujemy tylko na zbiorze uczącym, by uniknąć wycieku danych. Przed eksperymentami ustalimy, jaka poprawa macro-F1 względem punktu odniesienia jest dla nas istotna.

## Uzasadnienie wyboru źródeł

Synthea jest generatorem, więc nie spełnia wymogu, by źródłem był istniejący system. CDC PLACES podaje jedną wartość wskaźnika dla całego hrabstwa, czyli dane silnie zagregowane, podczas gdy hurtownia ma przechowywać dane o niskim poziomie agregacji. Poza tym bazy szpitalne są zanonimizowane i nie podają hrabstwa pacjenta, więc nie ma klucza, po którym dałoby się je połączyć z CDC. MIMIC-IV i eICU-CRD są natomiast rzeczywistymi, niezależnymi bazami szpitalnymi z własnymi schematami, identyfikatorami i kodowaniem. Temat (cukrzyca i nadciśnienie) pozostaje taki sam jak w pierwotnej propozycji. Zmienia się punkt odniesienia: zamiast regionalnych wskaźników CDC porównujemy systemy i regiony USA.

## Ograniczenia

Nie wiemy jeszcze, czy zrzuty badawcze MIMIC-IV i eICU-CRD zostaną uznane za „istniejący system informatyczny”, bo powstały z systemów szpitalnych, ale nie są ich bezpośrednią kopią. Liczebność jest niewielka: MIMIC demo ma 35 pacjentów z cukrzycą, a w eICU region Northeast ma tylko 14 pobytów z cukrzycą. Brak prawdziwych dat wyklucza analizę trendów, a dane pochodzą z OIOM i szpitali w USA, więc wnioski nie dotyczą populacji ogólnej.
