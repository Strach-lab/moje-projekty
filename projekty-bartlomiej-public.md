# 🚀 Projekty IT - Bartłomiej

> Ostatnia aktualizacja: Listopad 2024

---

## 📊 Status projektów

| Projekt | Status | Priorytet |
|---------|--------|-----------|
| AdGuard Home (DNS) | ✅ Działa | - |
| Portainer | ✅ Działa | - |
| Home Assistant | ✅ Działa | - |
| Homepage/Homarr | 🔄 W trakcie | Średni |
| Polskie listy AdGuard | 📋 Do zrobienia | Wysoki |
| Help desk system | 📋 Do zrobienia | Średni |
| Smart zamek deadbolt | 📋 Do zrobienia | Niski |

---

## 🖥️ Infrastruktura

### Sprzęt
- **Raspberry Pi** - główny serwer self-hosted
- **Orange Pi Zero LTS** - do wykorzystania
- **PC z RTX 4070** - desktop

### Stack technologiczny

| Kategoria | Narzędzie |
|-----------|-----------|
| Wirtualizacja | Proxmox VE |
| Storage | Synology |
| Sieć | FortiGate, UniFi |
| DNS/Adblock | AdGuard Home |
| Kontenery | Docker + Portainer |
| Smart Home | Home Assistant, WiZ |
| Automatyzacja | n8n |
| OS | Windows 11, Windows Server |

---

## 🏠 Self-hosted na RPi

### ✅ Działające usługi

#### AdGuard Home
- **Rola:** DNS (nie DHCP - zostaje na routerze)
- **Problem:** Reklamy na Onet/WP wciąż przechodzą
- **Do zrobienia:** Dodać polskie listy blokujące

#### Portainer
- **Rola:** Zarządzanie kontenerami Docker
- **Status:** Działa bez problemów

#### Home Assistant
- **Rola:** Centrum smart home
- **Integracje:** WiZ (żarówki, wtyczki)

### 🔄 W trakcie

#### Dashboard (Homepage / Homarr)
- **Dashy** - porzucone (problemy ze stabilnością)
- **Homepage** - testowane, były problemy z "Host validation failed"
- **Homarr** - alternatywa do przetestowania

**Ostatni działający stack Homepage:**
```yaml
version: "3.8"

services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    ports:
      - "3000:3000"
    environment:
      - TZ=Europe/Warsaw
    volumes:
      - homepage_config:/app/config
    restart: unless-stopped

volumes:
  homepage_config:
```

### 📋 Planowane

#### Polskie listy blokujące dla AdGuard
Listy do dodania:
- `https://raw.githubusercontent.com/MajkiIT/polish-ads-filter/master/polish-adblock-filters/adblock.txt`
- `https://raw.githubusercontent.com/FiltersHeroes/KAD/master/KAD.txt`
- `https://raw.githubusercontent.com/PolishFiltersTeam/PolishAnnoyanceFilters/master/PPB.txt`

#### Narzędzia do dodania
- **Uptime Kuma** - monitoring statusu usług
- **Glances** - statystyki systemu
- **File Browser** - przeglądarka plików

---

## 💼 Środowisko pracy

### Planowane wdrożenia

#### System Help Desk
Rozważane opcje:
1. **FreeScout** - lekki, open-source
2. **Zammad** - bardziej rozbudowany
3. **osTicket** - klasyka

---

## 🏠 Smart Home

### Obecne urządzenia
- **WiZ** - żarówki, wtyczki (WiFi)
- **Home Assistant** - centrum sterowania

### Planowane
- **Smart zamek do deadbolt** - szukany model kompatybilny z HA
- **Czujniki dymu** - integracja z systemem

---

## 🎮 Gaming

### Guild Wars 2
- **Build:** Condi (Mesmer vs Scourge - do sprawdzenia przeżywalność)
- **Cele:** Raidy, Open World
- **Addony:** BlishHUD
- **Farmy:** Provisioner Tokens, Legendary Armor
- **Tip:** Black Friday - zniżki na dodatki

---

## 💰 Inwestycje

### Trading 212
- Portfel PIE
- ETF-y: iShares MSCI World Quality
- Akcje: AAPL, MSFT (niewielka ekspozycja)
- **Uwaga:** Sprawdzić opłaty ADR

---

## 📝 Notatki

### Problemy do rozwiązania
1. Reklamy na Onet/WP mimo AdGuard
2. Dashboard - wybrać Homepage vs Homarr
3. Konfiguracja Synology Drive dla stanowisk

### Porzucone projekty
- **Dashy** - niestabilny, problemy z configiem
- **Nextcloud jako zamiennik Qsync** - problemy z GUI uprawnień

---

*Wygenerowane z eksportu ChatGPT przez Claude*
