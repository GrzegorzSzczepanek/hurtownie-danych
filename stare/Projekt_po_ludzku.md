# Projekt po ludzku

Wyjaśnienie bez żargonu. Czytaj od góry do dołu, każda część opiera się na poprzedniej.

---

## 1. O co w ogóle chodzi

Projekt ma dwie niezależne części.

**Część pierwsza (hurtownia danych):** zbieramy dane o pacjentach z dwóch szpitali w jednym uporządkowanym miejscu, żeby móc łatwo odpowiadać na pytania, np. „czy pacjenci z cukrzycą leżą dłużej niż inni?”.

**Część druga (eksploracja danych):** uczymy komputer zgadywać, którzy pacjenci z cukrzycą wrócą do szpitala po wypisie.

Obie części dotyczą cukrzycy, ale używają **innych danych** i nie zależą od siebie.

## 2. Co się zmieniło po uwagach prowadzącego

| Było | Jest | Dlaczego |
|---|---|---|
| dane ze Syntheą (program, który wymyśla pacjentów) | dane z prawdziwych baz szpitalnych | prowadzący chce istniejącego systemu, a nie generatora |
| dane z CDC (jedna liczba na całe hrabstwo) | dane o pojedynczych pacjentach | hurtownia ma liczyć z drobnych danych, a nie z gotowych sum |
| przewidujemy: wróci albo nie wróci (2 możliwości) | przewidujemy: wróci szybko, wróci później, nie wróci (3 możliwości) | w danych są trzy odpowiedzi, a my dwie sklejaliśmy w jedną |

Temat zostaje ten sam: cukrzyca i nadciśnienie.

---

## 3. Część pierwsza: hurtownia, krok po kroku

### Po co ona jest

Szpital prowadzi swoją bazę do codziennej pracy: przyjęcie pacjenta, wynik badania, wypis. Taka baza jest szybka w zapisywaniu, ale kiepska w odpowiadaniu na pytania typu „jak to wygląda ogółem”.

**Hurtownia** to osobna kopia danych, poukładana tak, żeby łatwo było liczyć i porównywać. Przenoszenie danych ze szpitala do hurtowni (z oczyszczaniem po drodze) nazywa się **ETL**.

Obraz do zapamiętania: szpital to zeszyt, do którego wpisuje się wszystko na bieżąco. Hurtownia to segregator, w którym to samo jest przepisane tak, żeby dało się policzyć.

### Skąd bierzemy dane

Dwie otwarte bazy szpitalne z USA:

- **MIMIC-IV:** jeden szpital w Bostonie. W naszej wersji demo jest tylko 100 pacjentów.
- **eICU:** wiele szpitali z całych Stanów, ale tylko oddziały intensywnej terapii (OIOM). W demo jest 2520 pobytów.

Pacjenci w obu bazach to **różni ludzie**. Nie łączymy więc „tego samego pacjenta z dwóch miejsc”. Łączymy dane tak, żeby dało się je porównać.

### Dlaczego to nie jest takie proste

Obie bazy opisują to samo inaczej. Trzy przykłady:

1. **Choroby są zapisane kodami w dwóch wersjach** (starszej i nowszej). Trzeba je sprowadzić do jednego sposobu, żeby „cukrzyca” znaczyła to samo w obu.
2. **Daty są ukryte** (dla ochrony prywatności). MIMIC podaje tylko przedział lat, np. 2011–2013. eICU podaje, ile minut po przyjęciu coś się stało. Nie mamy więc prawdziwego kalendarza.
3. **Badania cukrzycy (HbA1c) są tylko w MIMIC.** W eICU ich nie ma w ogóle.

To wszystko jest właśnie **pracą ETL** i dlatego projekt ma sens.

### Najważniejsze pojęcia na jednym przykładzie

Wyobraź sobie paragon ze sklepu. Jeden wiersz na paragonie to jedna rzecz kupiona.

U nas jeden wiersz to **jeden pobyt pacjenta na OIOM**.

| Pojęcie | Co to znaczy | U nas |
|---|---|---|
| **Fakt** | jedno zdarzenie, które zapisujemy (wiersz na paragonie) | jeden pobyt na OIOM |
| **Miara** | liczba przy tym zdarzeniu, którą można dodawać lub uśredniać (cena na paragonie) | ile dni trwał pobyt, czy pacjent zmarł (0 lub 1) |
| **Wymiar** | „według czego” chcemy to oglądać (kto, gdzie, kiedy) | pacjent, choroba, miejsce, czas |
| **Atrybut** | szczegół opisujący wymiar | pacjent: płeć, wiek. Miejsce: region |
| **Hierarchia** | drabinka od szczegółu do ogółu | szpital → region → cały system |

Jak to rozpoznać: jeśli możesz to **zsumować lub uśrednić**, to miara. Jeśli według tego **grupujesz**, to wymiar lub jego atrybut.

Jedno zdanie, które używa wszystkiego naraz:

> „Średni czas pobytu (miara) pacjentów z cukrzycą (atrybut wymiaru *choroba*) w regionie South (poziom hierarchii *miejsce*).”

### Gwiazda

Tabela z faktami jest w środku, a wokół niej wymiary. Tak to wygląda na rysunku, stąd nazwa **schemat gwiazdy**:

```
              Pacjent
                 |
   Czas ——— [ POBYT NA OIOM ] ——— Choroba
                 |
              Miejsce
```

U nas są **trzy gwiazdy**, bo mamy trzy rodzaje zdarzeń:

1. przyjęcie do szpitala (MIMIC),
2. pobyt na OIOM (eICU),
3. wynik badania HbA1c (MIMIC).

Wymiary **Choroba** i **Miejsce** są wspólne dla gwiazd. Dzięki temu można porównywać dane z MIMIC i eICU. Kilka gwiazd ze wspólnymi wymiarami to **konstelacja**. Prowadzący prosił właśnie o „osobne gwiazdy dla każdego zbioru”.

Dwa drobiazgi, które mogą paść w pytaniach:
- **Ziarno** to odpowiedź na pytanie „co dokładnie znaczy jeden wiersz?”. U nas: jedno przyjęcie albo jeden pobyt.
- Jedno przyjęcie ma często kilka chorób naraz, więc potrzebna jest dodatkowa tabela łącząca pobyty z chorobami (tabela mostkowa).

### Na jakie pytania odpowiadamy

Pytania muszą wymagać **obu** baz naraz:

1. Czy cukrzyca jest częstsza w jednych regionach USA niż w innych?
2. Jak często cukrzyca idzie w parze z nadciśnieniem, i czy w obu bazach tak samo?
3. Czy pacjenci z cukrzycą częściej umierają w szpitalu niż pozostali?
4. Czy leżą dłużej na OIOM?
5. (tylko MIMIC) Ilu pacjentów z cukrzycą ma dobrze wyrównany cukier? Tutaj danych jest mało (20 osób), więc to tylko pokaz, że miara działa.

### Czego nie da się zrobić (i trzeba to uczciwie powiedzieć)

- **Nie zrobimy trendów w czasie**, bo nie mamy prawdziwych dat.
- **Nie porównamy pojedynczych szpitali**, bo na jeden szpital w eICU przypada tylko 10–40 pobytów. Porównujemy regiony.
- **Nie porównamy wartości ciśnienia** między bazami, bo w jednej to pomiary z wizyt, a w drugiej z monitora na OIOM. Dlatego nadciśnienie definiujemy rozpoznaniem lekarza.

---

## 4. Część druga: zgadywanie, kto wróci do szpitala

### Co robimy

Mamy gotowy zbiór: **ponad 100 tysięcy pobytów w szpitalach z cukrzycą**. Przy każdym wiemy, co się potem stało. Są trzy możliwości:

- `<30`: wrócił w ciągu 30 dni,
- `>30`: wrócił później,
- `NO`: nie wrócił.

Komputer uczy się na części pobytów, a potem sprawdzamy, jak zgaduje na pobytach, których nie widział.

Rozkład odpowiedzi: nie wrócił 54%, wrócił później 35%, wrócił szybko tylko 11%.

### Dlaczego „procent trafnych odpowiedzi” oszukuje

Jeśli komputer **zawsze** odpowie „nie wróci”, będzie miał rację w 54% przypadków, nie umiejąc nic. To wygląda przyzwoicie, a jest bezwartościowe.

Dlatego liczymy miarę, która bierze pod uwagę **każdą z trzech odpowiedzi osobno** (to się nazywa macro-F1) i patrzymy, co z czym się myli (macierz pomyłek).

### Dlaczego jeden pacjent nie może być w nauce i w sprawdzianie naraz

To jak egzamin z zadań, które uczeń już widział. W zbiorze jest 71 tysięcy pacjentów, a prawie połowa pobytów (47 tysięcy) należy do osób, które były w szpitalu **więcej niż raz**. Gdyby ta sama osoba trafiła i do uczenia, i do testu, komputer „podejrzałby” odpowiedź. Dlatego dzielimy dane **po pacjentach, a nie po pobytach**.

### Inne decyzje po ludzku

- **Usuwamy pobyty zakończone śmiercią lub hospicjum** (2423 z ponad 100 tysięcy), bo taki pacjent nie ma jak wrócić do szpitala.
- **Zaczynamy od prostego modelu** (regresja logistyczna), żeby mieć punkt odniesienia. Potem próbujemy trudniejszych (drzewo, las, boosting) i sprawdzamy, czy naprawdę są lepsze.
- **Ustawienia modelu** (np. jak głębokie ma być drzewo) to „pokrętła”, które ustawiamy sami przed nauką. Nie testujemy wszystkich kombinacji, tylko losowe, bo to szybsze (random search).
- **Z góry ustalamy, co uznamy za „wystarczająco dobry wynik”**, żeby nie szukać w nieskończoność.

---

## 5. Co powiedzieć, gdy prowadzący zapyta „dlaczego”

Zasada prowadzącego: *jeśli robicie A, a nie B, musicie umieć powiedzieć dlaczego.* Zestaw gotowców:

- **Dlaczego nie Synthea?** Bo to program, który wymyśla pacjentów, a my mamy użyć istniejącego systemu.
- **Dlaczego nie CDC?** Bo to gotowe sumy dla całych hrabstw, a hurtownia ma liczyć z drobnych danych. Poza tym w bazach szpitalnych nie ma adresu pacjenta, więc nie ma jak ich połączyć.
- **Dlaczego trzy klasy?** Bo „wrócił po miesiącu” i „nie wrócił” to różne sytuacje. Sklejenie ich ukrywałoby informację.
- **Dlaczego region, a nie szpital?** Bo na jeden szpital przypada za mało pobytów, żeby coś porównywać.
- **Dlaczego nie ma trendów w czasie?** Bo dane celowo ukrywają prawdziwe daty.

**Najsłabsze miejsce projektu** (lepiej powiedzieć to pierwsze, niż żeby prowadzący to znalazł): MIMIC i eICU to bazy przygotowane do badań z prawdziwych systemów szpitalnych. Trzeba zapytać prowadzącego, czy uzna je za „istniejący system”.

---

## 6. Co dalej

1. **Zapytać prowadzącego** o to jedno pytanie z końca punktu 5. Od odpowiedzi zależy, czy budujemy na MIMIC i eICU.
2. **Przepisać propozycję projektu** w prostym języku, na bazie tego dokumentu.
3. **Zbudować hurtownię** i ETL.
4. **Zrobić eksperymenty** w części drugiej.

Dokładne liczby, alternatywy i uzasadnienia są w pozostałych plikach:
- `Rejestr_decyzji`: dlaczego tak i co odrzuciliśmy,
- `Sciagawka_QA`: pytania i odpowiedzi,
- `Propozycja_projektu_HD_ED_v2`: wersja do oddania (gęsta).
