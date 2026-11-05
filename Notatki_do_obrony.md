# Notatki do obrony projektu

Dokument pomocniczy, nie do oddania. Wyjaśnia pojęcia użyte w propozycji i zbiera decyzje razem z alternatywami, które odrzuciliśmy. Liczby pochodzą z pobranych plików MIMIC-IV demo 2.2, eICU-CRD demo 2.0.1 i zbioru UCI Diabetes 130.

## 1. Pojęcia modelu wielowymiarowego

Najłatwiej zrozumieć je na jednym zdaniu: „średni czas pobytu pacjentów z cukrzycą w regionie South”.

Fakt to zdarzenie, które zapisujemy w hurtowni. Jedno zdarzenie to jeden wiersz tabeli faktów. U nas są to przyjęcie do szpitala (MIMIC) i pobyt na OIOM (eICU). W zdaniu powyżej faktem jest pobyt na OIOM.

Miara to liczba zapisana przy fakcie, którą da się sumować albo uśredniać. Tutaj jest to czas pobytu w dniach. Inna nasza miara to zgon (0 lub 1). Jeśli wartość można sensownie zsumować lub uśrednić, jest miarą.

Wymiar to punkt widzenia, według którego grupujemy fakty: kto, co, gdzie, kiedy. Mamy cztery wymiary: Pacjent, Diagnoza, Placówka i Czas. W zdaniu przykładowym „z cukrzycą” dotyczy wymiaru Diagnoza, a „w regionie South” wymiaru Placówka.

Atrybut to cecha opisująca wymiar, na przykład płeć i wiek pacjenta albo region i status szpitala uczącego. Różnica względem miary jest taka, że atrybut leży w tabeli wymiaru i służy do grupowania, a miara leży w tabeli faktów i służy do liczenia.

Hierarchia to uporządkowane poziomy w obrębie wymiaru, od najbardziej szczegółowego do najogólniejszego. Wymiar Placówka ma poziomy szpital, region, system. Dzięki hierarchii można wynik „zwijać” do regionu albo „rozwijać” do szpitala.

Ziarno to odpowiedź na pytanie, co dokładnie reprezentuje jeden wiersz tabeli faktów. W gwieździe eICU jest to jeden pobyt na oddziale, a w gwieździe MIMIC jedno przyjęcie do szpitala. Prowadzący może zapytać o ziarno przy każdej gwieździe.

Schemat gwiazdy to tabela faktów otoczona tabelami wymiarów, z którymi łączy ją klucz obcy. Mamy dwie gwiazdy: Przyjęcie (MIMIC) i Pobyt OIOM (eICU). Gwiazdy mają wspólne wymiary, na przykład Diagnozę i Placówkę. Taki układ nazywa się konstelacją faktów, a wspólne wymiary wymiarami uzgodnionymi. Dzięki nim da się zestawić dane z MIMIC i eICU w jednym zapytaniu.

Tabela mostkowa jest potrzebna, gdy jeden fakt ma wiele wartości wymiaru. Jedno przyjęcie ma zazwyczaj kilka rozpoznań, więc zamiast jednej kolumny „diagnoza” potrzebna jest tabela łącząca przyjęcia z diagnozami albo osobne flagi (cukrzyca, nadciśnienie).

## 2. Hurtownia i ETL

System transakcyjny (OLTP) obsługuje codzienną pracę placówki: zapisuje przyjęcia, wyniki, wypisy. Jest zoptymalizowany pod szybkie dopisywanie i zmianę pojedynczych rekordów i zwykle pokazuje stan bieżący. Hurtownia służy do analiz. Zbiera dane historyczne z wielu źródeł, jest oczyszczona i uporządkowana pod duże zapytania, a dane ładuje się do niej okresowo, bez bieżących zmian.

ETL oznacza ekstrakcję, transformację i ładowanie. Ekstrakcja to pobranie danych ze źródeł. Transformacja to oczyszczenie i ujednolicenie, u nas między innymi sprowadzenie kodów ICD-9 i ICD-10 do jednej klasyfikacji, wyrównanie nazw pól (zgon jest raz w `hospital_expire_flag`, raz w `hospitaldischargestatus`) i wyliczenie miar. Ładowanie to zapis wyniku do hurtowni. Między etapami dane leżą w obszarze przejściowym.

Hurtownia, którą budujemy, jest relacyjna (ROLAP), czyli model wielowymiarowy zapisujemy zwykłymi tabelami w PostgreSQL. Alternatywą jest MOLAP, czyli osobne struktury wielowymiarowe. Slajdy wskazują ROLAP jako typowy wybór dla centralnej hurtowni, bo nie ogranicza liczby wymiarów i łatwiej go modyfikować.

Hurtownia różni się od składnicy danych (data martu). Hurtownia obejmuje całą organizację i trzyma dane mało zagregowane, a składnica jest przeznaczona dla jednego działu i zawiera dane silnie zagregowane. To jeden z powodów, dla których dane CDC nie nadają się na źródło.

## 3. Decyzje dotyczące źródeł danych

Pierwotnie planowaliśmy Synthea i CDC PLACES. Prowadzący zwrócił uwagę, że źródłem ma być istniejący system informatyczny, a nie generator ani gotowy zbiór.

Synthea odpada, bo jest programem, który wymyśla pacjentów. Nie ma więc własnego schematu operacyjnego, a dane pochodzą z naszej konfiguracji, a nie z działającego systemu.

CDC PLACES odpada jako źródło główne z dwóch powodów. Po pierwsze, podaje jedną wartość wskaźnika dla całego hrabstwa, a hurtownia według wykładu ma przechowywać dane mało zagregowane. Po drugie, bazy szpitalne są zanonimizowane i nie zawierają hrabstwa pacjenta, więc nie ma klucza, po którym dałoby się je połączyć z CDC. Zrezygnowanie z CDC zmienia zgłoszony temat, bo znika tło regionalnych wskaźników. Zastępujemy je porównaniem systemów i regionów, a sam temat (cukrzyca i nadciśnienie) zostaje.

Wybraliśmy MIMIC-IV i eICU-CRD, ponieważ są to rzeczywiste, niezależne bazy szpitalne z własnymi schematami, identyfikatorami i kodowaniem. Pacjenci się nie pokrywają, więc integracja polega na uzgodnieniu wymiarów, a nie na łączeniu rekordów jednej osoby. Dobrze to uzasadnia ETL, bo diagnozy są zapisane w dwóch wersjach klasyfikacji (w MIMIC 2313 wierszy ICD-10 i 2193 ICD-9), czas jest zamaskowany inaczej w każdej bazie, a nazwy pól różnią się nawet dla tych samych pojęć.

Rozważaliśmy też inne opcje. Zbiór UCI Diabetes 130 jako źródło części pierwszej odpada, bo to jedna płaska tabela z jednego źródła, a prowadzący chce wielu baz. Zostaje za to w części drugiej. OpenMRS (demo z 5000 pacjentów) to dump działającego oprogramowania, więc jest bliżej „prawdziwego systemu”, ale według wyników wyszukiwania dane demo mają przemieszany wiek i płeć, co psuje wymiar Pacjent. Nie pobieraliśmy go i nie sprawdzaliśmy, czy zawiera cukrzycę. Zostaje rezerwą na wypadek odrzucenia MIMIC i eICU. Pełny MIMIC-IV wymaga szkolenia i umowy z PhysioNet, więc jest niedostępny w naszym terminie. Moduł MIMIC-IV-ED to część tej samej bazy (łączy się przez `subject_id`), więc nie byłby niezależnym źródłem, a jego zawartości nie sprawdzaliśmy.

Mamy dwa źródła w części pierwszej, bo tyle dało się znaleźć niezależnych, publicznych i niezmieszanych z generatorem. To minimum zgodne z wymaganiem „wielu źródeł”, ale obie bazy pochodzą od tego samego dostawcy (PhysioNet) i mają podobny format plików, co osłabia argument o różnorodności. Rozważaliśmy trzecie źródło. Moduł MIMIC-IV-ED jest częścią tej samej bazy, więc niczego nie wnosi. Zewnętrzny słownik ICD jest tabelą referencyjną, a nie systemem, więc nie liczymy go jako źródła, tylko jako uzupełnienie hierarchii diagnozy (patrz niżej).

Najsłabsze miejsce tej decyzji jest takie, że MIMIC-IV i eICU są bazami badawczymi zbudowanymi z systemów szpitalnych, a nie samymi systemami. Nie wiemy, czy prowadzący to zaakceptuje, i warto go o to zapytać, zanim zaczniemy budować ETL.

## 4. Decyzje dotyczące modelu hurtowni

Porównujemy tylko pobyty na OIOM. eICU zawiera wyłącznie OIOM, a porównanie z wszystkimi przyjęciami MIMIC zestawiałoby różne populacje, więc w MIMIC bierzemy pobyty z tabeli `icustays`.

Mamy osobną gwiazdę dla każdego źródła, ze wspólnymi wymiarami. Tak prosił prowadzący, a źródła rzeczywiście mają różne ziarno i miary: przyjęcie do szpitala nie jest tym samym co pobyt na oddziale, a w MIMIC są dodatkowe dane o przyjęciach, których nie ma w eICU. Alternatywą była jedna wspólna tabela faktów z kolumną „źródło”. Byłaby prostsza w zapytaniach między systemami, ale zacierałaby różnice w ziarnie.

Wymiar Czas jest względny. W MIMIC daty są przesunięte (lata 2110–2201), a prawdziwy rok znamy tylko jako przedział 2011–2013 albo 2014–2016. W eICU jest przesunięcie w minutach od przyjęcia i rok wypisu (2014 lub 2015). Hierarchia dzień–miesiąc–rok jest więc niemożliwa i nie robimy analizy trendów.

Porównujemy regiony, a nie szpitale. W demo eICU jest 186 szpitali, a na jeden przypada od 10 do 40 pobytów i nie więcej niż 16 z cukrzycą, co nie wystarcza do porównań. Regiony mają od 14 pobytów z cukrzycą (Northeast) do 125 (South). Osiemnaście szpitali nie ma regionu i dostają wartość „nieznany”.

Cukrzycę i nadciśnienie definiujemy kodami ICD, bo kody są w obu bazach. Definicja przez HbA1c odpada, bo eICU w ogóle nie ma tego badania (zero wierszy). Definicja nadciśnienia przez pomiar ciśnienia też odpada: pomiary istnieją w obu bazach, ale w MIMIC to zapisy ambulatoryjne (tabela `omr`, 866 pomiarów), a w eICU monitorowanie na OIOM (`vitalAperiodic` i `vitalPeriodic`), więc ich wartości nie są porównywalne. Dane eICU zawierają dodatkowo wartości zerowe, które trzeba usunąć.

Rozdziały ICD pochodzą z własnej tabeli `slowniki/rozdzialy_icd.csv`. Słownik w MIMIC (`d_icd_diagnoses`) ma tylko kod, wersję i pełną nazwę, bez rozdziałów, a w eICU słownika nie ma. Rozważaliśmy pobranie gotowego pliku (CMS dla ICD-10-CM i ICD-9-CM, WHO w formacie ClaML), ale nie sprawdziliśmy, czy pliki CMS zawierają nagłówki rozdziałów (z opisu wynika, że to listy kodów z opisami), a pobranie z WHO wymaga rejestracji i akceptacji licencji. Główny powód własnej tabeli jest jednak inny: podział na rozdziały w obu wersjach klasyfikacji jest różny, więc i tak trzeba było ustalić wspólną listę. Zrobiliśmy dwadzieścia rozdziałów. Choroby oczu i uszu z ICD-10 łączymy z układem nerwowym, tak jak w ICD-9, a zaburzenia odporności (kod 279 w ICD-9) przypisujemy do rozdziału o krwi, tak jak w ICD-10. Dodaliśmy też rozdział przyczyn zewnętrznych, który w ICD-10 ma kody V00–Y99, a w ICD-9 E800–E999 i E000–E030, oraz rozdział kodów specjalnych (U00–U85).

Tabelę napisaliśmy na podstawie struktury klasyfikacji, a nie przepisaliśmy z pobranego pliku, więc jej poprawność sprawdziliśmy tylko pośrednio. Każdy kod ze słownika MIMIC (109 775) i każdy kod z eICU (42 135) trafia do jakiegoś rozdziału. Sprawdzenie to wykryło, że początkowo brakowało zakresu E000–E030 w ICD-9 (111 kodów o aktywnościach, jak skakanie na bungee), który dopisaliśmy. Nie sprawdzaliśmy natomiast, czy każdy kod trafił do właściwego rozdziału.

W eICU pole `icd9code` zawiera od jednego do siedmiu kodów, najczęściej parę „ICD-9, ICD-10”. Gdy jest jeden kod, może to być każda z wersji, więc wersję rozpoznajemy po kształcie kodu. Kod zaczynający się cyfrą to ICD-9, kod z literą to ICD-10. Wyjątkiem są kody zaczynające się od V i E. Kody V01–V91 traktujemy jako ICD-9 (w ICD-10 litera V oznacza rzadkie przyczyny zewnętrzne), a o kodach z E decyduje położenie kropki: `E870.0` jest ICD-9, a `E87.0` ICD-10. To założenie, które może się mylić w rzadkich przypadkach.

Wcześniej mieliśmy piąte pytanie, o odsetek pacjentów z cukrzycą z wynikiem HbA1c poniżej 7%, i osobną gwiazdę z pomiarami HbA1c. Usunęliśmy je po uwadze prowadzącego, że każde pytanie ma korzystać z kilku źródeł, a to dotyczyło tylko MIMIC (eICU nie ma HbA1c w ogóle). Nie było to też pytanie analityczne, tylko opis jednego wskaźnika, a w MIMIC mieliśmy do niego zaledwie 20 pacjentów z cukrzycą. Gwiazda HbA1c nie miała po tej zmianie pytania, któremu by służyła.

## 5. Decyzje dotyczące eksploracji danych

Część drugą opieramy na zbiorze UCI Diabetes 130 (101 766 hospitalizacji, 71 518 pacjentów, 130 szpitali w USA). Powody są cztery. Po pierwsze, dotyczy cukrzycy, więc cały projekt ma jeden temat. Po drugie, zmienna `readmitted` ma w danych naturalnie trzy wartości, a prowadzący chce klasyfikacji trójklasowej. Po trzecie, zbiór jest wystarczająco duży i bogaty w cechy (wiek, leki, wyniki badań, liczba wcześniejszych wizyt), żeby porównywać kilka modeli. Najmniejsza klasa `<30` ma 11 357 przykładów. Po czwarte, slajdy do przedmiotu wymieniają UCI jako typowe miejsce wyboru danych.

Rozważaliśmy inne opcje. MIMIC demo nie nadaje się do uczenia, bo ma tylko 100 pacjentów. Mniejsze zbiory o cukrzycy z UCI (na przykład zbiór Pima Indians, który według naszej wiedzy ma kilkaset rekordów i dotyczy samego rozpoznania cukrzycy, a nie ponownego przyjęcia) nie pasują do tematu przewidywania hospitalizacji. Tych zbiorów nie sprawdzaliśmy w danych. Wadą wybranego zbioru jest to, że pochodzi z lat 1999–2008, więc praktyki leczenia są sprzed ponad piętnastu lat, i że ma kilka znanych problemów (wielu pacjentów z kilkoma pobytami, anomalie w kodach wypisu), które opisujemy niżej.

Zadanie ma trzy klasy zamiast dwóch. Prowadzący tego chciał, a poza tym ponowne przyjęcie po ponad 30 dniach i brak ponownego przyjęcia to różne zdarzenia kliniczne, których scalenie ukrywałoby informację. Rozkład klas to `NO` 53,9%, `>30` 34,9% i `<30` 11,2%. Wariant dwuklasowy zostaje jako dodatkowe porównanie.

Główną miarą jest macro-F1, a nie trafność (accuracy). Klasyfikator, który zawsze odpowiada `NO`, ma około 54% trafności bez żadnej wiedzy, więc ta miara myli. F1 łączy precyzję (jaka część przewidzianych przypadków klasy była trafna) z czułością (jaką część rzeczywistych przypadków klasy znaleziono). Macro-F1 jest średnią F1 po trzech klasach, więc mała klasa `<30` liczy się tak samo jak duża. Dodajemy też macierz pomyłek, żeby zobaczyć, co z czym model myli.

Walidację krzyżową dzielimy po pacjentach (`patient_nbr`). Większość pacjentów ma jeden pobyt, ale 16 773 ma więcej niż jeden (do 40), a ich pobyty to 46% rekordów. Przy zwykłym podziale ten sam pacjent trafiałby jednocześnie do uczenia i testu, a model mógłby „rozpoznać” go zamiast nauczyć się wzorców. Nazywa się to wyciekiem danych.

Usuwamy pobyty zakończone zgonem lub hospicjum, czyli kody wypisu 11, 13, 14, 19, 20 i 21. To 2423 rekordy, z czego 2337 miało klasę `NO`, bo taki pacjent nie może wrócić do szpitala. W tej grupie jest jednak po 43 rekordy z klasami `<30` i `>30`, co wygląda na anomalię w danych. Nie umiemy jej wyjaśnić i warto o tym powiedzieć wprost.

Modele porównujemy od najprostszego: regresja logistyczna jako punkt odniesienia, potem drzewo decyzyjne, las losowy i gradient boosting. Tak radzi wykład, który mówi też, że żaden algorytm nie jest najlepszy dla każdego problemu.

Hiperparametry dobieramy losowo (random search). Hiperparametr to ustawienie, które wybieramy przed uczeniem (na przykład głębokość drzewa), w odróżnieniu od parametrów, które model dostraja sam. Grid search sprawdzałby wszystkie kombinacje i jest sensowny tylko przy kilku wartościach, a optymalizacja bayesowska jest bardziej złożona, niż tu potrzeba. Słabością losowego przeszukiwania jest to, że wynik zależy od przedziałów, które sami wybierzemy.

Przy niezbalansowaniu klas stosujemy wagi klas. Oversampling dopuszczamy tylko na zbiorze uczącym, bo zastosowany przed podziałem przeniósłby kopie rekordów do testu. Z góry ustalamy też, jaka poprawa macro-F1 względem punktu odniesienia jest dla nas istotna.

## 6. Czego projekt nie potrafi

Nie pokaże trendów w czasie, bo nie mamy prawdziwych dat. Nie porówna pojedynczych szpitali i nie oceni kontroli HbA1c w eICU, bo tego badania tam nie ma. Nie porówna wartości ciśnienia między bazami. Region Northeast ma tylko 14 pobytów z cukrzycą, więc wnioski o nim są ostrożne, a wyniki dotyczą pacjentów szpitalnych i OIOM w USA, a nie ogółu populacji.

## 7. Pytania, które mogą paść

Dlaczego nie Synthea? Bo to generator, a mamy użyć istniejącego systemu informatycznego.

Dlaczego nie CDC? Dane są zagregowane do poziomu hrabstwa, a hurtownia ma przechowywać dane o niskim poziomie agregacji. Dodatkowo nie mamy klucza, po którym połączylibyśmy je z bazami szpitalnymi.

Czy MIMIC i eICU to nie gotowe zbiory? To relacyjne bazy szpitalne z wieloma tabelami, a nie pojedyncza płaska tabela, ale rzeczywiście są to wersje przygotowane do badań. Warto zapytać prowadzącego, czy to akceptuje.

Gdzie jest integracja, skoro pacjenci się nie pokrywają? W uzgodnieniu wymiarów: inne identyfikatory, dwie wersje klasyfikacji chorób, różne modele czasu, różne nazwy pól dla tych samych pojęć.

Dlaczego zmieniliście źródła, ale nie temat? Temat był już zgłoszony, a uwagi prowadzącego dotyczyły źródeł. Tło porównawcze to teraz systemy i regiony zamiast wskaźników hrabstw.

Dlaczego osobne gwiazdy? Źródła mają różne ziarno i miary, a prowadzący o to prosił. Wspólne wymiary pozwalają je zestawiać.

Dlaczego macro-F1? Bo trafność przy takim rozkładzie klas myli: stała odpowiedź `NO` daje 54%.

Dlaczego walidacja po pacjentach? Bo 46% rekordów należy do pacjentów z więcej niż jednym pobytem i bez tego byłby wyciek danych.

Dlaczego trzy klasy? Bo `>30` i `NO` oznaczają różne zdarzenia, a scalenie ich traciłoby informację.
