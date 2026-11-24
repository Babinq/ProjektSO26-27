# Projekt z przedmiotu Systemy Operacyjne (2026/2027)
## Temat 3: Muzeum

* **Autor:** Przemysław Babiński
* **Nr albumu:** 158118
* **Grupa laboratoryjna:** GL01
* **Repozytorium GitHub:** https://github.com/Babinq/ProjektSO26-27
* **Język implementacji:** C++ (kompilacja na serwerze torus)

---

## 1. Opis ogólny i zasady działania symulacji

Celem projektu jest wieloprocesowa symulacja funkcjonowania muzeum działającego w oparciu o mechanizmy komunikacji międzyprocesowej (IPC) systemu Linux:

* **Dni i godziny otwarcia:**
  * Muzeum jest zamknięte w każdy **wtorek** oraz w wybrane dni świąteczne (**1 stycznia, 1 maja, 25 grudnia**).
  * W **poniedziałki, czwartki, soboty i niedziele** muzeum jest czynne w godzinach **9:00 – 18:00**.
  * W **środy i piątki** obowiązują wydłużone godziny otwarcia (nokturny) do **21:00**.
  * Opuszczanie sal wystawowych rozpoczyna się na **30 minut przed zamknięciem**.
  * Każde wyjście z budynku jest ostateczne (bilet nie uprawnia do ponownego wejścia w tym samym dniu).
* **Pojemność obiektu i bilety:**
  * Wewnątrz muzeum może przebywać jednocześnie maksymalnie **$N$ osób**.
  * Zwiedzający w wieku od **1 do 100 lat** przychodzą w losowych momentach czasu.
  * **Bilet standardowy:** 22 PLN dla obywateli/mieszkańców EOG oraz 32 PLN dla osób spoza EOG.
  * **Darmowy wstęp:** przysługuje dzieciom i młodzieży do 18. roku życia oraz młodzieży do 26. roku życia będącej obywatelami lub rezydentami UE/EOG — wymagana jest wcześniejsza rezerwacja darmowej wejściówki na konkretną godzinę online (ok. 1% wszystkich odwiedzających posiada taką rezerwację).
  * **Darmowe wieczory:** w każdy pierwszy piątek miesiąca w godzinach **18:00 – 21:45** (z wyłączeniem lipca i sierpnia) wstęp jest bezpłatny, również po wcześniejszej rezerwacji online.
  * Kasa rejestruje liczbę sprzedanych i zarezerwowanych biletów w danym dniu.
* **Kontrola bezpieczeństwa i zasady porządkowe:**
  * Do muzeum można wnieść bagaż podręczny o maksymalnych wymiarach **55 × 35 × 20 cm**. Osoby z większym bagażem nie są wpuszczane (brak przechowalni).
  * Osoby, które kupiły bilet w kasie, ale nie zostały wpuszczone wyłącznie z powodu zbyt dużego bagażu, mogą wrócić do kasy i otrzymać zwrot pieniędzy (zwrócony bilet wraca do puli sprzedaży).
  * Kontrola bezpieczeństwa (skanery, bramki) odbywa się równolegle na **2 stanowiskach**, po maksymalnie **3 osoby jednocześnie** na każdym stanowisku.
  * **Reguła płci:** jeżeli na stanowisku kontrolowana jest więcej niż 1 osoba jednocześnie, muszą to być osoby tej samej płci (zasada nie dotyczy dzieci).
  * **Zapobieganie frustracji w kolejce:** zwiedzający oczekujący przed kontrolą może przepuścić maksymalnie **5 innych osób**.
  * **Opieka nad dziećmi:** dzieci poniżej **10. roku życia** wchodzą do muzeum wyłącznie pod opieką osoby dorosłej.
  * Kontrola bezpieczeństwa rejestruje wchodzących, niewpuszczonych oraz wychodzących i na bieżąco wymienia informacje z kasą w celu kontroli limitu $N$ osób w budynku.

---

## 2. Architektura procesów i mechanizmy systemowe

Symulacja zostanie zrealizowana w architekturze rozproszonej (bez procesu centralnie sterującego logiką) z wykorzystaniem wywołań `fork()` oraz `exec()`:

1. **`main` (Proces inicjujący / zegar symulacji):** waliduje parametry wejściowe, inicjalizuje zasoby IPC (z minimalnymi prawami dostępu `0600`), uruchamia procesy obsługi oraz generuje procesy zwiedzających, a po zakończeniu lub przerwaniu (`SIGINT`) usuwa wszystkie struktury IPC.
2. **`dyrektor`:** nadzoruje pracę obiektu i wysyła polecenia za pomocą sygnałów:
   * `sygnał 1` (`SIGUSR1`) – wstrzymanie wpuszczania nowych zwiedzających na stanowiskach kontroli bezpieczeństwa,
   * `sygnał 2` (`SIGUSR2`) – wznowienie wpuszczania zwiedzających,
   * `sygnał 3` (`SIGTERM`) – natychmiastowa ewakuacja wszystkich zwiedzających z budynku; po opuszczeniu obiektu przez ostatnią osobę pracownik techniczny wysyła potwierdzenie do dyrektora.
3. **`kasjer`:** obsługuje sprzedaż biletów, weryfikację rezerwacji online, zwroty biletów za nadwymiarowy bagaż oraz zlicza osoby przebywające w muzeum.
4. **`pracownik_tech` (Kontrola bezpieczeństwa i nadzór techniczny):** obsługuje 2 bramki kontroli (limity miejsc, reguła płci, limit 5 przepuszczeń, kontrola wymiarów bagażu), rejestruje ruch zwiedzających oraz raportuje zakończenie ewakuacji dyrektorowi.
5. **`zwiedzajacy`:** reprezentuje pojedynczego odwiedzającego (lub dorosłego z dzieckiem < 10 lat) przechodzącego przez kasę, kontrolę bezpieczeństwa, zwiedzanie oraz wyjście.

**Planowane mechanizmy IPC i logowanie:**
* **Pamięć dzielona + Semafory System V:** stan muzeum (licznik osób $\le N$, status otwarcia/wstrzymania/ewakuacji) oraz stan 2 stanowisk kontroli bezpieczeństwa (zajętość, płeć w bramce, licznik przepuszczeń).
* **Kolejki komunikatów System V:** komunikacja `zwiedzajacy <-> kasjer` (zakup/zwrot biletu) oraz `pracownik_tech -> kasjer` (raportowanie wejść i wyjść).
* **Plik raportu (logi):** pełny zapis zdarzeń symulacji zapisywany do pliku tekstowego `raport_muzeum.log`.

---

## 3. Planowane testy funkcjonalne

### Test 1: Weryfikacja limitu pojemności muzeum ($N$) i zwalniania miejsc
* **Cel:** Sprawdzenie, czy liczba osób przebywających wewnątrz muzeum nigdy nie przekracza zadanego limitu $N$ oraz czy wyjście zwiedzającego poprawnie odblokowuje wejście dla kolejnej osoby (brak zakleszczeń).
* **Przebieg:** Uruchomienie symulacji z małym limitem $N = 5$ przy jednoczesnym wygenerowaniu 20 zwiedzających z ważnymi biletami i poprawnym bagażem.
* **Oczekiwany rezultat:** W logach symulacji stan licznika osób wewnątrz przyjmuje wartości z przedziału $[0, 5]$. Osoby 6–20 oczekują przed wejściem i wchodzą kolejno po zakończeniu zwiedzania przez poprzedników.

### Test 2: Kontrola bezpieczeństwa – reguła płci oraz limit 5 przepuszczeń
* **Cel:** Sprawdzenie poprawnej synchronizacji na stanowisku kontroli przy jednoczesnym wymogu tej samej płci (max 3 osoby) oraz ochronie przed zagłodzeniem (max 5 przepuszczonych osób).
* **Przebieg:** Skierowanie do jednego stanowiska kontroli 1 osoby płci M, a bezpośrednio po niej strumienia 8 osób płci K.
* **Oczekiwany rezultat:** Bramka obsługuje maksymalnie 3 osoby płci K jednocześnie, a po przepuszczeniu łącznie 5 osób płci K blokuje kolejne osoby K, opróżnia stanowisko i wpuszcza oczekującą osobę płci M.

### Test 3: Odmowa wejścia z powodu nadwymiarowego bagażu i zwrot biletu w kasie
* **Cel:** Weryfikacja ścieżki obsługi zwiedzającego posiadającego opłacony bilet, ale wnoszącego bagaż przekraczający 55 × 35 × 20 cm.
* **Przebieg:** Wygenerowanie zwiedzających kupujących bilet w kasie, z których część losuje bagaż o wymiarach np. 70 × 40 × 30 cm.
* **Oczekiwany rezultat:** Kontrola bezpieczeństwa nie wpuszcza tych osób do muzeum i nie zwiększa licznika osób wewnątrz. Zwiedzający wracają do kasy, otrzymują zwrot opłaty, a zwrócony bilet wraca do puli dostępnych biletów.

### Test 4: Reakcja na sygnały Dyrektora (wstrzymanie wejść, wznowienie, ewakuacja) oraz czyszczenie zasobów IPC
* **Cel:** Sprawdzenie asynchronicznej obsługi sygnałów (`sygnał 1`, `sygnał 2`, `sygnał 3`) oraz poprawnego zwalniania pamięci dzielonej, semaforów i kolejek komunikatów.
* **Przebieg:** Podczas pełnego obciążenia muzeum wysłanie kolejno: `sygnału 1` (wstrzymanie wejść), `sygnału 2` (wznowienie wejść), a następnie `sygnału 3` (ewakuacja). Po zakończeniu sprawdzenie poleceniem `ipcs` stanu zasobów systemowych.
* **Oczekiwany rezultat:** Po `sygnale 1` nikt nie wchodzi, ale osoby ze środka mogą wyjść; po `sygnale 2` ruch wraca do normy; po `sygnale 3` wszyscy zwiedzający opuszczają budynek, pracownik techniczny wysyła potwierdzenie do dyrektora, a polecenie `ipcs` nie wykazuje żadnych pozostałych segmentów pamięci, semaforów ani kolejek.
