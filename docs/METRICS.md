# Katalog metryk i testów (Windows 11)

Tutaj jest lista tego, **co** zbieramy, **skąd** i **kiedy uznajemy to za problem**.
Progi są wartościami startowymi – możesz je zmieniać w `thresholds.py`.

Legenda źródeł: **psutil** · **WMI** (klasa, np. `Win32_BIOS`) · **PS** (komenda PowerShell) · **LHM** (LibreHardwareMonitor) · 🔑 wymaga administratora

> 💡 Chcesz podejrzeć, co zwraca dana klasa WMI, zanim napiszesz kod? W PowerShell:
> `Get-CimInstance Win32_BIOS | Format-List *`

---

## Część A – Kolektory (odczyt)

### system
| Metryka | Źródło | Próg / po co |
|---------|--------|--------------|
| wersja i build Windows, edycja | WMI `Win32_OperatingSystem` | stary build → WARN (brak aktualizacji) |
| aktywacja Windows | WMI `SoftwareLicensingProduct` | nieaktywowany → WARN |
| czas od uruchomienia (uptime) | `psutil.boot_time()` | informacyjnie; > 30 dni → WARN (brak restartu po aktualizacjach) |
| producent, model, numer seryjny | WMI `Win32_ComputerSystem`, `Win32_BIOS` | identyfikacja sprzętu |
| wersja i data BIOS | WMI `Win32_BIOS` | bardzo stary BIOS → informacja |
| płyta główna | WMI `Win32_BaseBoard` | – |
| Secure Boot, TPM | PS `Confirm-SecureBootUEFI` 🔑, WMI `Win32_Tpm` 🔑 | wymagane przez Win 11 – wyłączone → WARN |
| ostatnie aktualizacje | PS `Get-HotFix` | brak aktualizacji > 60 dni → WARN |

### cpu
| Metryka | Źródło | Próg WARN / FAIL |
|---------|--------|------------------|
| model, rdzenie fizyczne/logiczne | `py-cpuinfo`, `psutil.cpu_count()` | – |
| taktowanie aktualne / max | `psutil.cpu_freq()`, WMI `Win32_Processor` | aktualne < 50% max pod obciążeniem → WARN (throttling) |
| obciążenie % (ogólne i na rdzeń) | `psutil.cpu_percent()` | > 80% w spoczynku → WARN |
| wirtualizacja włączona w BIOS | WMI `Win32_Processor.VirtualizationFirmwareEnabled` | informacyjnie |

> Uwaga: `psutil.getloadavg()` na Windows jest tylko emulowane – nie używamy go.

### memory
| Metryka | Źródło | Próg WARN / FAIL |
|---------|--------|------------------|
| RAM całkowity / użyty / wolny | `psutil.virtual_memory()` | użycie > 85% WARN, > 95% FAIL |
| plik stronicowania | `psutil.swap_memory()` | użycie > 50% WARN |
| moduły RAM: sloty, pojemność, prędkość, producent | WMI `Win32_PhysicalMemory` | różne prędkości/pojemności modułów → WARN; mniej RAM niż suma modułów → WARN |
| zainstalowany vs dostępny RAM | WMI vs psutil | duża różnica (np. zarezerwowane przez GPU) → informacja |

### disk
| Metryka | Źródło | Próg WARN / FAIL |
|---------|--------|------------------|
| partycje, litery dysków, system plików | `psutil.disk_partitions()` | – |
| zajętość każdej partycji | `psutil.disk_usage()` | > 85% WARN, > 95% FAIL; dysk C: < 10 GB wolnego → FAIL |
| dyski fizyczne: model, typ (HDD/SSD/NVMe), rozmiar | PS `Get-PhysicalDisk` | HDD jako dysk systemowy → informacja („wymiana na SSD przyspieszy komputer”) |
| stan wg Windows | PS `Get-PhysicalDisk` (`HealthStatus`, `OperationalStatus`) | `Warning` → WARN, `Unhealthy` → FAIL |

### smart 🔑
Dwa źródła – najpierw wbudowane, a jeśli jest zainstalowany `smartctl`, to także pełne dane.

| Metryka | Źródło | Próg WARN / FAIL |
|---------|--------|------------------|
| temperatura, godziny pracy, zużycie (wear) | PS `Get-PhysicalDisk \| Get-StorageReliabilityCounter` | temp > 55°C WARN; wear > 80% WARN |
| błędy odczytu/zapisu (nienaprawione) | j.w. (`ReadErrorsUncorrected`, `WriteErrorsUncorrected`) | > 0 FAIL |
| przewidywana awaria | WMI `MSStorageDriver_FailurePredictStatus` (namespace `root\wmi`) | `PredictFailure = True` → FAIL |
| realokowane sektory (ID 5), oczekujące (197), nienaprawialne (198) | `smartctl -A -j` (opcjonalnie) | ID 5 > 0 WARN, > 50 FAIL; 197/198 > 0 FAIL |
| NVMe: `percentage_used`, `media_errors` | `smartctl -a -j` (opcjonalnie) | used > 80% WARN, errors > 0 FAIL |

### network
| Metryka | Źródło | Próg WARN / FAIL |
|---------|--------|------------------|
| karty sieciowe, IP, MAC, stan | `psutil.net_if_addrs()`, `psutil.net_if_stats()` | brak IP poza 127.0.0.1 i 169.254.x.x → FAIL (169.254 = brak DHCP) |
| prędkość łącza | `psutil.net_if_stats()` / PS `Get-NetAdapter` | Ethernet 10/100 Mb/s zamiast 1000 → WARN (kabel/port) |
| brama, serwery DNS | PS `Get-NetIPConfiguration` | brak bramy → FAIL |
| błędy i zgubione pakiety | `psutil.net_io_counters(pernic=True)` | błędy > 0.1% pakietów → WARN |
| Wi-Fi: SSID, sygnał %, pasmo, standard | `netsh wlan show interfaces` | sygnał < 40% WARN |
| sterownik karty sieciowej | PS `Get-NetAdapter` (`DriverVersion`, `DriverDate`) | informacyjnie |

> `netsh` zwraca tekst w języku systemu – parsowanie musi uwzględnić polską i angielską wersję (lub szukać po pozycji).

### sensors
| Metryka | Źródło | Próg WARN / FAIL |
|---------|--------|------------------|
| temperatury CPU (pakiet, rdzenie), GPU, płyty, dysków | **LHM** przez WMI `root\LibreHardwareMonitor` → klasa `Sensor` | CPU > 80°C w spoczynku WARN, > 95°C FAIL |
| obroty wentylatorów | LHM | 0 RPM przy temp > 70°C → FAIL |
| napięcia | LHM | informacyjnie |
| (awaryjnie) strefy termiczne ACPI | WMI `MSAcpi_ThermalZoneTemperature` 🔑 | często niedostępne lub niedokładne – tylko informacyjnie |

> Brak uruchomionego LibreHardwareMonitor → `SKIPPED` z komunikatem „Uruchom LibreHardwareMonitor jako administrator, aby odczytać temperatury”.

### battery (laptopy)
| Metryka | Źródło | Próg WARN / FAIL |
|---------|--------|------------------|
| poziom %, ładowanie, szacowany czas | `psutil.sensors_battery()` | – (brak baterii → `SKIPPED`) |
| pojemność projektowa | WMI `BatteryStaticData.DesignedCapacity` (namespace `root\wmi`) | – |
| pojemność obecna (pełne naładowanie) | WMI `BatteryFullChargedCapacity` (`root\wmi`) | zdrowie = obecna / projektowa; < 80% WARN, < 50% FAIL |
| liczba cykli | WMI `BatteryCycleCount` (`root\wmi`) lub `powercfg /batteryreport /xml` | informacyjnie |
| plan zasilania | `powercfg /getactivescheme` | informacyjnie (tryb oszczędzania spowalnia komputer) |

### gpu
| Metryka | Źródło | Próg WARN / FAIL |
|---------|--------|------------------|
| model, wersja i data sterownika, VRAM | WMI `Win32_VideoController` | sterownik „Microsoft Basic Display Adapter” → FAIL (brak sterownika) |
| stan urządzenia | WMI `Win32_VideoController.Status` / `ConfigManagerErrorCode` | ≠ 0 → FAIL |
| NVIDIA: temp, użycie, VRAM, wentylator | `nvidia-ml-py` (opcjonalnie) | temp > 85°C WARN |
| AMD/Intel: temp | LHM | temp > 85°C WARN |

> `Win32_VideoController.AdapterRAM` pokazuje maksymalnie 4 GB (błąd Windows) – przy większych kartach bierzemy VRAM z nvidia-ml-py lub rejestru.

### processes
| Metryka | Źródło | Po co |
|---------|--------|-------|
| top 10 procesów wg CPU i RAM | `psutil.process_iter()` | znalezienie „winowajcy” spowolnień |
| liczba procesów | j.w. | > 300 → WARN |
| programy w autostarcie | WMI `Win32_StartupCommand` | > 15 → WARN (wolny start systemu) |
| zainstalowany antywirus i jego stan | WMI `root\SecurityCenter2` → `AntiVirusProduct` | brak lub wyłączony → WARN |

### drivers
| Metryka | Źródło | Próg WARN / FAIL |
|---------|--------|------------------|
| urządzenia z błędem (żółty wykrzyknik w Menedżerze urządzeń) | PS `Get-PnpDevice -Status ERROR,DEGRADED,UNKNOWN` lub WMI `Win32_PnPEntity` (`ConfigManagerErrorCode ≠ 0`) | każde → WARN; kod błędu tłumaczymy na tekst (np. 28 = brak sterownika) |

### eventlog
Dziennik zdarzeń Windows – najlepsze źródło informacji „co się działo, gdy komputer się psuł”.
Wszystko przez PS `Get-WinEvent -FilterHashtable @{...}` z ostatnich 7 dni.

| Zdarzenie | Dziennik / ID | Znaczenie | Próg |
|-----------|---------------|-----------|------|
| nagła utrata zasilania / twardy reset | System, **Kernel-Power 41** | zawieszenia, problemy z zasilaczem, przegrzewanie | > 0 → WARN, > 3 → FAIL |
| niebieski ekran (BSOD) | System, **BugCheck 1001** | krytyczny błąd sterownika/sprzętu | > 0 → FAIL |
| błąd sprzętowy | System, **WHEA-Logger** (ID 1, 17, 18, 19, 47) | CPU/RAM/PCIe zgłasza błędy | > 0 → FAIL |
| błędy dysku | System, **disk 7, 51, 153**; **Ntfs 55** | złe sektory, problemy z kablem/kontrolerem | > 0 → WARN |
| nieoczekiwane zamknięcie | System, **EventLog 6008** | j.w. co Kernel-Power | > 0 → WARN |
| awarie aplikacji | Application, **1000** | informacyjnie – top 5 najczęściej padających programów | – |
| minidumpy BSOD | pliki w `C:\Windows\Minidump` | liczba i daty | > 0 → informacja |

---

## Część B – Testy (aktywne)

### network_test
| Krok | Jak | Wynik |
|------|-----|-------|
| 1. brama domyślna | ping do routera (adres z `Get-NetIPConfiguration`) | brak odpowiedzi → FAIL („problem z siecią lokalną / Wi-Fi”) |
| 2. internet po IP | ping `1.1.1.1`, `8.8.8.8` | brak → FAIL („brak internetu”) |
| 3. DNS | `socket.gethostbyname("google.com")` | nie działa przy działającym kroku 2 → FAIL („problem z DNS”) |
| 4. HTTP(S) | `urllib.request` do znanej strony | błąd → WARN (proxy/firewall/antywirus) |
| 5. opóźnienie i utrata pakietów | 20 pingów (`ping -n 20`) | strata > 2% lub ping > 100 ms → WARN |
| 6. (opcjonalnie) prędkość | pobranie pliku testowego, pomiar Mb/s | informacyjnie |

Kolejność ma znaczenie: dzięki niej raport mówi **gdzie** jest problem, a nie tylko „nie działa”.
Na Windows `ping` używa `-n` (liczba pakietów) i `-w` (timeout w ms).

### cpu_stress
| Element | Opis |
|---------|------|
| obciążenie | `multiprocessing` – jeden proces liczący na każdy rdzeń logiczny |
| czas | domyślnie 60 s (parametr `--duration`) |
| pomiar co 1 s | temperatura (LHM, jeśli dostępny), taktowanie, obciążenie |
| FAIL | temp > 95°C, samoczynne zamknięcie procesu, błąd obliczeń (wynik niezgodny z oczekiwanym), Kernel-Power 41 po teście |
| WARN | spadek taktowania > 30% (throttling), temp > 85°C |
| bezpieczeństwo | przerwanie przy temp > 100°C i przy Ctrl+C |
| bez LHM | test działa, ale ocenia tylko taktowanie i stabilność – w raporcie informacja o braku temperatur |

### disk_speed
| Element | Opis |
|---------|------|
| zapis sekwencyjny | zapis pliku tymczasowego (np. 1 GB) z `os.fsync` |
| odczyt sekwencyjny | odczyt tego pliku (uwaga: Windows cache'uje pliki – wynik może być zawyżony; rozwiązanie w planie) |
| WARN | HDD < 80 MB/s, SSD SATA < 300 MB/s, NVMe < 1000 MB/s |
| sprzątanie | plik tymczasowy usuwany **zawsze** (`try/finally`) |
| bezpieczeństwo | sprawdzenie wolnego miejsca przed testem |

### memory_test
| Element | Opis |
|---------|------|
| działanie | alokacja części wolnego RAM, zapis wzorców, sprawdzenie odczytu |
| ograniczenie | to **nie** zastępuje testu przy starcie komputera – Windows nie da nam całej pamięci; wykryje tylko grube usterki |
| uzupełnienie | w raporcie podpowiedź: „dokładny test: Diagnostyka pamięci systemu Windows (`mdsched.exe`) lub MemTest86” |
| FAIL | niezgodność odczytanego wzorca |
