# pc-checker

Narzędzie w Pythonie (uruchamiane z wiersza poleceń) do diagnostyki komputerów i laptopów z **Windows 11**.

**Co robi:**
1. **Zbiera metryki** – wszystkie informacje o sprzęcie i systemie, które da się odczytać z poziomu kodu
   (CPU, RAM, dyski, SMART, sieć, Wi-Fi, bateria, temperatury, GPU, procesy, dziennik zdarzeń Windows…).
2. **Przeprowadza testy** – aktywne sprawdzenia, np. łączność sieciowa, stress test CPU, szybkość dysku.
3. **Ocenia wyniki** – każdy wynik dostaje status `OK` / `WARN` / `FAIL`, żeby od razu było widać, gdzie jest problem.
4. **Tworzy raport** – w terminalu (kolorowo) i do pliku (JSON, później HTML).
5. **(docelowo) Działa jako jeden plik `pc-checker.exe`** – do uruchomienia z pendrive'a na komputerze klienta, bez instalowania Pythona.

> Status projektu: **projektowanie**. Kod jeszcze nie istnieje – powstaje krok po kroku według planu.

## Dokumentacja

| Plik | Co zawiera |
|------|------------|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Jak zbudowany jest projekt i dlaczego tak |
| [docs/METRICS.md](docs/METRICS.md) | Katalog metryk i testów – co zbieramy, skąd i jakie są progi |

Materiały edukacyjne (plan pracy, słowniczek, konwencje) leżą lokalnie w `docs/learning/` i nie są częścią repozytorium.

## Docelowe użycie (gdy powstanie)

W PowerShell **uruchomionym jako administrator**:

```powershell
pc-checker                     # zbierz wszystkie metryki i pokaż raport
pc-checker --only cpu,disk     # tylko wybrane obszary
pc-checker --test network      # uruchom test łączności
pc-checker --test cpu_stress --duration 60
pc-checker --all --json raport.json
```

## Wymagania

- Windows 11 (Windows 10 prawdopodobnie też zadziała, ale nie jest celem)
- Python 3.12+ (tylko do developmentu – użytkownik końcowy dostanie `.exe`)
- Development: kod i repozytorium na serwerze Linux, uruchamianie na Windows 11 (szczegóły: ARCHITECTURE.md §11)
- Uprawnienia administratora – bez nich część metryk (SMART, temperatury, część logów) zostanie pominięta
- Opcjonalnie: [LibreHardwareMonitor](https://github.com/LibreHardwareMonitor/LibreHardwareMonitor) (dokładne temperatury i wentylatory), `smartctl` ze [smartmontools](https://www.smartmontools.org/) (pełne dane SMART)
