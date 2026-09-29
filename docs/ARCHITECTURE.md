# Architektura pc-checker

Ten dokument tłumaczy, **jak** projekt jest zbudowany i **dlaczego** właśnie tak.
Czytaj go przed rozpoczęciem każdej fazy planu – będziesz do niego wracać.

**Platforma docelowa: Windows 11.** Wszystkie decyzje poniżej zakładają Windows.

---

## 1. Główna idea w jednym zdaniu

> Program składa się z wielu **małych, niezależnych modułów**. Każdy z nich robi jedną rzecz
> i zwraca wynik w **tym samym formacie**. Na końcu jeden moduł zbiera wszystkie wyniki i robi z nich raport.

Wyobraź sobie warsztat samochodowy: każdy mechanik sprawdza inną część (hamulce, silnik, opony)
i wpisuje wynik na **ten sam formularz**. Kierownik zbiera formularze i wystawia raport dla klienta.
Gdy chcesz sprawdzać coś nowego – zatrudniasz nowego mechanika, nie przebudowujesz całego warsztatu.

To jest cała tajemnica skalowalności: **wspólny format wyniku + jedna lista modułów**.

---

## 2. Dwa rodzaje modułów

| Rodzaj | Nazwa w projekcie | Co robi | Przykład | Czas |
|--------|-------------------|---------|----------|------|
| **Kolektor** | `collectors/` | Tylko **odczytuje** informacje. Niczego nie obciąża. | „Ile jest RAM-u i ile wolnego?” | ułamek sekundy |
| **Test** | `diagnostics/` | Aktywnie **coś robi** i mierzy efekt. | „Obciąż CPU na 60 s i patrz na temperaturę” | sekundy–minuty |

Rozdzielamy je, bo kolektory można uruchamiać zawsze i wszystkie naraz, a testy – tylko na życzenie
(obciążają komputer, trwają długo, czasem wymagają internetu).

---

## 3. Skąd na Windows bierzemy dane? (4 źródła)

To najważniejsza rzecz do zrozumienia przy Windows. Nie ma jednego miejsca ze wszystkimi informacjami –
korzystamy z czterech „kranów”, od najprostszego do najtrudniejszego:

| # | Źródło | Co to jest | Co z niego bierzemy | Jak w Pythonie |
|---|--------|------------|---------------------|----------------|
| 1 | **psutil** | biblioteka Pythona | CPU %, RAM, partycje, sieć, procesy, bateria (podstawowo) | `import psutil` |
| 2 | **WMI / CIM** | wbudowana w Windows „baza danych” o sprzęcie i systemie; pytamy ją zapytaniami podobnymi do SQL | model płyty, BIOS, moduły RAM, dyski fizyczne, GPU, zdrowie baterii, stan dysków | biblioteka `wmi` |
| 3 | **PowerShell / narzędzia systemowe** | komendy Windows uruchamiane z Pythona | dziennik zdarzeń (`Get-WinEvent`), liczniki niezawodności dysków, Wi-Fi (`netsh`), raport baterii (`powercfg`) | `subprocess.run(["powershell", ...])` |
| 4 | **Programy zewnętrzne (opcjonalne)** | darmowe narzędzia, które trzeba doinstalować | dokładne temperatury i wentylatory (**LibreHardwareMonitor**), pełne dane SMART (**smartctl**) | `wmi` / `subprocess` |

**Dlaczego aż cztery?** Bo np. `psutil` na Windows **nie odczytuje temperatur**, a Windows sam z siebie
pokazuje je bardzo słabo. Każde źródło uzupełnia luki poprzedniego.

**Zasada:** każdy kolektor używa najprostszego źródła, które działa. Jeśli źródła brak (np. nie zainstalowano
LibreHardwareMonitor) → status `SKIPPED` z podpowiedzią, co doinstalować. Program **nigdy** się przez to nie wywala.

Żeby nie powtarzać w każdym kolektorze tego samego kodu do WMI i PowerShell, tworzymy moduł pomocniczy
**`winapi.py`** (sekcja 4) z kilkoma prostymi funkcjami, np. „wykonaj zapytanie WMI” i „uruchom komendę
PowerShell i zwróć wynik jako JSON”.

---

## 4. Struktura katalogów

```
pc-checker/
├── README.md                 ← opis projektu (pierwsza rzecz, którą ktoś czyta)
├── pyproject.toml            ← „dowód osobisty” projektu: nazwa, wersja, zależności
├── docs/                     ← dokumentacja (to, co teraz czytasz)
│
├── src/
│   └── pc_checker/           ← PAKIET – cały kod programu
│       ├── __init__.py       ← oznacza, że katalog jest pakietem (może być pusty)
│       ├── __main__.py       ← pozwala uruchomić: python -m pc_checker
│       ├── cli.py            ← obsługa wiersza poleceń (argumenty, flagi)
│       ├── runner.py         ← „kierownik”: uruchamia moduły, zbiera wyniki
│       ├── registry.py       ← LISTA wszystkich kolektorów i testów
│       ├── models.py         ← wspólny „formularz” wyniku (CheckResult, Status)
│       ├── thresholds.py     ← progi: kiedy WARN, kiedy FAIL (np. temp > 90°C)
│       ├── utils.py          ← drobne pomocnicze funkcje (np. bajty → „4.2 GB”)
│       ├── winapi.py         ← JEDYNE miejsce, które dotyka Windows: WMI, PowerShell, is_admin()
│       │
│       ├── collectors/       ← KOLEKTORY – każdy plik = jeden obszar
│       │   ├── __init__.py
│       │   ├── system.py     ← wersja Windows, aktywacja, uptime, producent, BIOS, Secure Boot/TPM
│       │   ├── cpu.py        ← model, rdzenie, taktowanie, obciążenie
│       │   ├── memory.py     ← RAM, plik stronicowania, moduły RAM
│       │   ├── disk.py       ← partycje, zajętość, dyski fizyczne
│       │   ├── smart.py      ← zdrowie dysków (SMART / liczniki niezawodności)
│       │   ├── network.py    ← karty sieciowe, IP, Wi-Fi, błędy pakietów
│       │   ├── sensors.py    ← temperatury, wentylatory (LibreHardwareMonitor)
│       │   ├── battery.py    ← bateria (laptopy): zużycie, cykle
│       │   ├── gpu.py        ← karta graficzna, sterownik
│       │   ├── processes.py  ← procesy zjadające najwięcej zasobów, autostart
│       │   ├── drivers.py    ← urządzenia z błędami w Menedżerze urządzeń
│       │   └── eventlog.py   ← błędy krytyczne z Dziennika zdarzeń (BSOD, WHEA, dyski)
│       │
│       ├── diagnostics/      ← TESTY – każdy plik = jeden test
│       │   ├── __init__.py
│       │   ├── network_test.py   ← brama, ping, DNS, HTTP, opóźnienie
│       │   ├── cpu_stress.py     ← stress test CPU
│       │   ├── disk_speed.py     ← prędkość zapisu/odczytu
│       │   └── memory_test.py    ← prosty test RAM
│       │
│       └── reports/          ← RAPORTY – różne sposoby pokazania wyników
│           ├── __init__.py
│           ├── console.py    ← kolorowa tabela w terminalu
│           ├── json_report.py← zapis do pliku JSON
│           └── html_report.py← (później) ładny raport HTML
│
└── tests/                    ← TESTY JEDNOSTKOWE naszego kodu (pytest) – działają na Linuxie
    ├── fixtures/             ← prawdziwe próbki danych nagrane na Windows (--dump-raw)
    ├── test_models.py
    ├── test_utils.py
    └── ...
```

> ⚠️ Uwaga na słowo „test”: `diagnostics/` to testy **komputera** (sprzętu),
> a `tests/` to testy **naszego kodu** (czy program działa poprawnie). To dwie różne rzeczy.

**Dlaczego `src/`?** To standardowy układ w Pythonie. Chroni przed błędem, w którym Python
importuje kod „z boku” zamiast zainstalowanej wersji. Na początku nie musisz tego rozumieć – po prostu tak robimy.

---

## 5. Serce projektu: wspólny format wyniku (`models.py`)

Każdy kolektor i każdy test zwraca obiekt **`CheckResult`**. Wygląda (koncepcyjnie) tak:

| Pole | Typ | Znaczenie | Przykład |
|------|-----|-----------|----------|
| `name` | tekst | nazwa modułu | `"memory"` |
| `status` | `Status` | ogólna ocena | `Status.WARN` |
| `summary` | tekst | jedno zdanie dla człowieka | `"RAM zajęty w 91%"` |
| `metrics` | słownik | surowe dane (klucz → wartość) | `{"total_gb": 16, "used_percent": 91}` |
| `issues` | lista tekstów | wykryte problemy | `["Mało wolnej pamięci"]` |
| `duration_s` | liczba | ile trwało sprawdzenie | `0.02` |
| `error` | tekst lub brak | jeśli moduł się wysypał | `None` |

**`Status`** to *enum* (zamknięta lista wartości):

| Status | Znaczenie |
|--------|-----------|
| `OK` | wszystko w porządku |
| `WARN` | coś niepokojącego, warto się przyjrzeć |
| `FAIL` | prawdopodobna usterka |
| `SKIPPED` | nie dało się sprawdzić (brak czujnika, brak uprawnień admina, brak baterii w PC, brak LibreHardwareMonitor) |
| `ERROR` | nasz kod rzucił wyjątek – błąd programu, nie komputera |

W Pythonie zrobimy to za pomocą `@dataclass` (dla `CheckResult`) i `Enum` (dla `Status`) – obie rzeczy
są wbudowane w Pythona, nie trzeba nic instalować.

**Dlaczego to takie ważne?** Bo raport, runner i CLI nie muszą nic wiedzieć o CPU czy dyskach.
Znają tylko `CheckResult`. Dzięki temu dodanie 20. kolektora nie wymaga zmian w raporcie.

---

## 6. Kontrakt modułu („umowa”)

Każdy moduł to zwykły plik `.py` z **jedną główną funkcją**:

- kolektor: funkcja `collect()` → zwraca `CheckResult`
- test: funkcja `run(options)` → zwraca `CheckResult` (`options` to np. czas trwania, adres do pingowania)

Świadomie **nie** używamy klas ani dziedziczenia na start – funkcje są prostsze dla początkującego.
Gdy projekt urośnie, można to zmienić bez przepisywania wszystkiego (patrz sekcja 12).

Wewnątrz modułu schemat jest zawsze ten sam:

```
1. Odczytaj dane (psutil → WMI → PowerShell → program zewnętrzny)
2. Włóż je do słownika metrics
3. Porównaj z progami z thresholds.py → ustal status i listę issues
4. Zwróć CheckResult
```

---

## 7. Rejestr (`registry.py`) – jedna lista, żeby wszystkim rządzić

Rejestr to po prostu **dwa słowniki**: nazwa → funkcja.

```
COLLECTORS = { "cpu": cpu.collect, "memory": memory.collect, ... }
DIAGNOSTICS = { "network": network_test.run, "cpu_stress": cpu_stress.run, ... }
```

**Dodanie nowego modułu = nowy plik + jedna linijka w rejestrze.** Nic więcej.

---

## 8. Runner – „kierownik” (`runner.py`)

Runner:
1. dostaje listę nazw do uruchomienia (od CLI),
2. na starcie sprawdza, czy program działa **jako administrator**, i ostrzega, jeśli nie,
3. bierze funkcje z rejestru,
4. uruchamia każdą **w bloku `try/except`** – jeśli jeden moduł się wysypie, program działa dalej,
   a ten moduł dostaje status `ERROR`,
5. mierzy czas,
6. zwraca listę `CheckResult`.

To najważniejsza zasada niezawodności: **awaria jednego modułu nigdy nie zatrzymuje całej diagnostyki.**
Diagnozujemy przecież komputery, które bywają zepsute – czujniki kłamią, sterowników brakuje, WMI bywa uszkodzone.

---

## 9. Przepływ danych (od komendy do raportu)

```
  użytkownik wpisuje: pc-checker --only cpu,memory --json out.json
            │
            ▼
      ┌──────────┐   parsuje argumenty
      │  cli.py  │──────────────────────┐
      └──────────┘                      │ lista nazw: ["cpu","memory"]
                                        ▼
                                 ┌─────────────┐   pyta o funkcje   ┌──────────────┐
                                 │  runner.py  │ ─────────────────▶ │ registry.py  │
                                 └─────────────┘                    └──────────────┘
                                        │ wywołuje
                     ┌──────────────────┼──────────────────┐
                     ▼                                     ▼
              collectors/cpu.py                    collectors/memory.py
              (psutil, winapi,                     (psutil, winapi,
               thresholds)                          thresholds)
                     │ CheckResult                         │ CheckResult
                     └──────────────────┬──────────────────┘
                                        ▼
                             lista [CheckResult, ...]
                                        │
                     ┌──────────────────┴──────────────────┐
                     ▼                                     ▼
            reports/console.py                    reports/json_report.py
            (tabela w terminalu)                  (plik out.json)
```

Zasada: **dane płyną w jedną stronę**. Kolektory nic nie wiedzą o raportach, raporty nic nie wiedzą o psutil czy WMI.

---

## 10. Biblioteki (zależności)

| Biblioteka | Po co | Obowiązkowa? |
|------------|-------|--------------|
| `psutil` | CPU, RAM, dyski, sieć, bateria, procesy | tak |
| `wmi` (+ instalowany automatycznie `pywin32`) | zapytania WMI – sprzęt, BIOS, dyski, GPU, bateria | tak |
| `rich` | kolorowe tabele i paski postępu w terminalu | tak |
| `py-cpuinfo` | dokładna nazwa modelu CPU, flagi (AVX itd.) | tak |
| `pytest` | testowanie naszego kodu | tylko do developmentu |
| `nvidia-ml-py` | szczegółowe dane z kart NVIDIA (temp, użycie, VRAM) | opcjonalna |
| `pyinstaller` | spakowanie programu do jednego `.exe` | tylko do budowania |
| `jinja2` | szablon raportu HTML | opcjonalna (faza późna) |

**Narzędzia Windows** (są w każdym Windows 11, wywołujemy je przez `subprocess`):
`powershell` (m.in. `Get-WinEvent`, `Get-PhysicalDisk`, `Get-StorageReliabilityCounter`, `Get-PnpDevice`),
`netsh wlan`, `powercfg`, `ping`, `ipconfig`.

**Programy opcjonalne** (trzeba doinstalować – bez nich odpowiednie moduły dają `SKIPPED`):

| Program | Po co | Uwagi |
|---------|-------|-------|
| **LibreHardwareMonitor** | temperatury CPU/GPU/płyty, obroty wentylatorów, napięcia | musi być uruchomiony (najlepiej jako admin); udostępnia dane przez WMI (`root\LibreHardwareMonitor`) |
| **smartctl** (smartmontools) | pełne atrybuty SMART | bez niego korzystamy z wbudowanych liczników Windows (mniej danych, ale zwykle wystarcza) |

---

## 11. Specyfika Windows, o której trzeba pamiętać

1. **Uprawnienia administratora.** Część danych (SMART, niektóre logi, LibreHardwareMonitor) wymaga uruchomienia
   PowerShell „jako administrator”. `winapi.py` ma funkcję `is_admin()` – runner informuje, jeśli jej wynik to `False`.
2. **`multiprocessing` na Windows** działa inaczej niż na Linuxie – kod uruchamiający procesy **musi** być wywoływany
   spod `if __name__ == "__main__":`, a w `.exe` dodatkowo potrzebne jest `multiprocessing.freeze_support()`.
   Bez tego stress test CPU zawiesi się lub uruchomi program w pętli.
3. **Polskie znaki i kodowanie.** Wyjście z `netsh`/`ipconfig` bywa w kodowaniu `cp852`, a nie UTF-8,
   i jest zależne od języka systemu. Dlatego, gdzie się da, używamy **PowerShell z `ConvertTo-Json`**
   (zwraca dane w stałym formacie, niezależnie od języka Windows) zamiast parsowania tekstu.
4. **PowerShell jest wolny na starcie** (~0.3–1 s na każde wywołanie). Łączymy kilka zapytań w jedno wywołanie,
   gdy to możliwe, i dajemy każdemu **timeout**.
5. **Antywirus / SmartScreen** może zgłaszać `.exe` z PyInstallera jako podejrzany. To normalne dla niepodpisanych programów.

### Gdzie pisać i testować kod? – model „Linux pisze, Windows uruchamia”

Projekt **mieszka na serwerze Linux** (tam jest repozytorium, tam piszesz kod, testujesz i robisz commity).
Program **działa naprawdę** tylko na Windows (WMI i PowerShell nie istnieją na Linuxie). Dlatego mamy dwie role:

| Maszyna | Rola | Co tam robisz |
|---------|------|---------------|
| **Serwer Linux** | warsztat | edycja kodu, `git`, `pytest` (testy z udawanymi danymi), `ruff`, dokumentacja |
| **Komputer z Windows 11** | poligon | `git pull` → uruchomienie programu na prawdziwym sprzęcie, nagranie próbek danych, budowa `.exe` |

```
   ┌──────────── serwer Linux ────────────┐                 ┌──────── Windows 11 ────────┐
   │ VS Code (Remote-SSH) / vim           │                 │                            │
   │ kod → pytest (mocki) → git commit    │── git push ───▶ │ git pull                   │
   │                                      │   (GitHub)      │ pc-checker (prawdziwe dane)│
   │ tests/fixtures/*.json  ◀─────────────┼─────────────────│ --dump-raw → próbki JSON   │
   └──────────────────────────────────────┘                 └────────────────────────────┘
```

**Jak połączyć maszyny (zalecane):**
- **Edycja:** VS Code na dowolnym komputerze + rozszerzenie **Remote - SSH** → pracujesz bezpośrednio na plikach serwera,
  a terminal w VS Code to terminal Linuxa. Świetna okazja do ćwiczenia Linuxa.
- **Przesyłanie kodu na Windows:** repozytorium na **GitHubie** jako `origin`. Linux robi `git push`, Windows robi `git pull`.
  Dodatkowy plus: kopia zapasowa kodu poza serwerem.

**Trzy zasady, które sprawiają, że to działa:**

1. **Tylko `winapi.py` dotyka Windows.** Importy `wmi` i `ctypes.windll` są **wyłącznie** tam i są wykonywane
   dopiero przy wywołaniu funkcji (tzw. *leniwy import*), a nie przy starcie programu. Dzięki temu każdy inny moduł
   da się zaimportować i przetestować na Linuxie.
2. **Testy podmieniają `winapi`, nie WMI.** W testach jednostkowych zamieniamy funkcje `winapi.wmi_query()`
   i `winapi.run_powershell()` na „udawane”, które zwracają przygotowane dane. Kolektor nie widzi różnicy.
3. **Prawdziwe próbki danych z Windows.** Program ma ukrytą flagę `--dump-raw`, która zapisuje surowe odpowiedzi
   WMI/PowerShell do plików JSON. Wgrywasz je do `tests/fixtures/` i testy na Linuxie pracują na **prawdziwych**
   danych z prawdziwych komputerów (także zepsutych – to najcenniejsze próbki!).

Zależność `wmi` w `pyproject.toml` oznaczamy jako tylko-Windows (`wmi; sys_platform == "win32"`),
więc `pip install` na Linuxie jej nie instaluje i nie zgłasza błędu.

**Czego nie da się zrobić na Linuxie:** uruchomienia programu na prawdziwych danych i zbudowania `.exe`
(PyInstaller buduje tylko dla systemu, na którym działa) – to zawsze robi Windows.

-------|--------|------|
| **A. Pisać i uruchamiać na Windows 11** (zalecane) | wszystko działa od razu, prawdziwy sprzęt | – |
| B. Pisać na Linuxie, testować na Windows (maszyna wirtualna lub drugi komputer) | możesz używać obecnego komputera | w VM nie ma prawdziwych czujników, SMART, baterii; ciągłe kopiowanie kodu |

Testy jednostkowe (`tests/`) z „udawanymi” danymi (mock) mogą działać wszędzie – to dodatkowy powód, żeby je pisać.

---

## 12. Jak to skaluje się w przyszłości

| Chcesz… | Co robisz |
|---------|-----------|
| nowy obszar (np. USB) | `collectors/usb.py` + linijka w rejestrze |
| nowy test (np. test GPU) | `diagnostics/gpu_stress.py` + linijka w rejestrze |
| nowy format raportu (np. PDF) | `reports/pdf_report.py` + flaga w CLI |
| inny próg alarmu | zmiana jednej liczby w `thresholds.py` (później: plik konfiguracyjny) |
| GUI zamiast CLI | nowy plik, który woła ten sam `runner.py` – reszta bez zmian |
| porównanie „przed/po” naprawie | wczytanie dwóch plików JSON i porównanie `metrics` |
| (kiedyś) obsługa Linuxa | w module: `if platform.system() == "Windows": ...` – architektura na to pozwala |

Gdy modułów będzie bardzo dużo, można zamienić funkcje na klasy ze wspólną klasą bazową
lub automatyczne wykrywanie modułów – ale **nie teraz**. Najpierw prostota.

---

## 13. Zasady projektowe (ściąga)

1. **Jeden plik = jedna odpowiedzialność.**
2. **Wspólny format wyniku** – zawsze `CheckResult`.
3. **Nigdy nie wywalaj całego programu** – `try/except` w runnerze, `SKIPPED`, gdy czegoś brak.
4. **Progi w jednym miejscu** – `thresholds.py`, nie „magiczne liczby” porozrzucane po kodzie.
5. **Kolektory tylko czytają** – nic nie zmieniają w systemie.
6. **Testy obciążające mają limit czasu** i zawsze sprzątają po sobie (pliki tymczasowe, procesy).
7. **Surowe dane zawsze w `metrics`** – nawet jeśli status to OK; przydadzą się do porównań.
8. **Każde wywołanie zewnętrzne (PowerShell, WMI, ping) ma timeout** – na zepsutym komputerze mogą wisieć w nieskończoność.
9. **Kod specyficzny dla Windows tylko w `winapi.py`** – reszta musi dać się zaimportować i przetestować na Linuxie.

---

## Historia decyzji

| Data | Decyzja | Powód |
|------|---------|-------|
| 2026-09-28 | Funkcje zamiast klas dla modułów | Prostsze dla początkującego; łatwa migracja później |
| 2026-09-28 | Ręczny rejestr (słownik) zamiast auto-wykrywania | Widać wprost, co jest uruchamiane; zero „magii” |
| 2026-09-28 | CLI (`argparse`) zamiast GUI na start | Mniej kodu, łatwiejsze testowanie; GUI można dodać później |
| 2026-09-28 | **Platforma docelowa: tylko Windows 11** | Wymaganie użytkownika; brak obsługi wielu systemów upraszcza kod |
| 2026-09-28 | Źródła danych: psutil → WMI → PowerShell → programy zewnętrzne | psutil nie daje na Windows wszystkiego (np. temperatur) |
| 2026-09-28 | PowerShell + `ConvertTo-Json` zamiast parsowania tekstu | Wynik niezależny od języka systemu i kodowania znaków |
| 2026-09-28 | LibreHardwareMonitor jako źródło temperatur | Windows nie udostępnia wiarygodnych temperatur bez zewnętrznego sterownika |
| 2026-09-28 | Dystrybucja jako `.exe` (PyInstaller) | Uruchamianie na komputerach klientów bez instalowania Pythona |
| 2026-09-28 | **Kod i repozytorium na serwerze Linux, uruchamianie na Windows** | Użytkownik chce ćwiczyć pracę na Linuxie; izolacja Windows w `winapi.py` + mocki + próbki z `--dump-raw` pozwalają testować na Linuxie |
| 2026-09-29 | GitHub jako `origin` (zamiast repozytorium bare na serwerze) | Wybór użytkownika; dodatkowo kopia zapasowa poza serwerem |
