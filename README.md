# IoT TCP Proxy za Mirage Honeypot — Opis projekta za praksu

## Uvod

Mirage je mrežni honeypot koji otvara TCP portove, simulira realne servise i loguje događaje o konekcijama.

Cilj ovog projekta je razvoj firmware-a za embedded (IoT) uređaj koji služi kao **udaljeni mrežni senzor** — otvara TCP portove i transparentno prosleđuje saobraćaj ka centralnoj Mirage instanci, vraćajući odgovor nazad klijentu. Na ovaj način se Mirage-ov domet širi na mreže na kojima se nalazi IoT uređaj, bez potrebe za instalacijom servera na svakoj lokaciji.

## Arhitektura

```
                         ┌─────────────────┐
  Napadač/Skener         │   IoT uređaj    │              Mirage server
  ─────────────────►     │    (RTOS)       │  ──────────►  (TCP listener)
  TCP konekcija          │                 │  TCP proxy     port N
  na port N              │  port N listen  │               │
                         │       │         │               │
  ◄─────────────────     │       ▼         │  ◄──────────  │
  odgovor                │  forward ◄──────│  odgovor      │
                         └─────────────────┘
```

**Tok podataka:**==

1. Napadač otvara TCP konekciju ka IoT uređaju na portu N
2. IoT uređaj prihvata konekciju
3. IoT uređaj otvara novu TCP konekciju ka Mirage serveru na istom portu N
4. Podaci se bidirekciono prosleđuju: napadač ↔ IoT ↔ Mirage
5. IoT prosleđuje odgovor od Mirage-a nazad napadaču
6. Konekcije se zatvaraju

## Funkcionalni zahtevi

### F1: Konfiguracija (YAML)

IoT uređaj čita konfiguraciju iz YAML fajla sa flash memorije. Konfiguracioni parametri:

| Parametar | Tip | Opis |
|-----------|-----|------|
| `mirage_ip` | string | IP adresa Mirage servera |
| `ports` | lista integer-a | Lista TCP portova na kojima IoT uređaj sluša |
| `wifi_ssid` | string | SSID WiFi mreže (ako se koristi WiFi) |
| `wifi_password` | string | Lozinka WiFi mreže |
| `connection_timeout_ms` | integer | Timeout za uspostavljanje konekcije ka Mirage (default: 5000) |
| `idle_timeout_ms` | integer | Timeout za neaktivnu konekciju (default: 30000) |

Primer konfiguracije:

```yaml
mirage_ip: "192.168.1.100"
ports:
  - 22
  - 80
  - 443
  - 8080

wifi_ssid: "MojaMreza"
wifi_password: "lozinka123"

connection_timeout_ms: 5000
idle_timeout_ms: 30000
```

**Napomena**: Portovi u IoT konfiguraciji moraju da odgovaraju portovima na kojima Mirage sluša.

### F2: TCP Proxy — prosleđivanje konekcija

Za svaki port iz konfiguracije, IoT uređaj:

1. Otvara TCP listener (bind + listen)
2. Prihvata dolazne konekcije (accept)
3. Za svaku prihvaćenu konekciju otvara novu TCP konekciju ka `mirage_ip` na istom portu
4. Bidirekciono prosleđuje podatke između klijenta i Mirage-a
5. Kada bilo koja strana zatvori konekciju, zatvara i drugu stranu

**Zahtevi:**

- Podrška za minimum 4-5 simultanih konekcija
- Maksimalan buffer za prosleđivanje: 4096 bajtova
- Pravilno upravljanje resursima — zatvaranje soketa, oslobađanje memorije
- Connection timeout: ako Mirage ne odgovori u definisanom roku, zatvoriti konekciju ka klijentu
- Idle timeout: ako nema podataka u oba smera duže od definisanog perioda, zatvoriti obe konekcije

### F3: Mrežna konekcija

- WiFi i/ili Ethernet (zavisi od izabranog board-a)
- Automatsko povezivanje na mrežu pri pokretanju
- Listeners se otvaraju tek nakon uspešnog povezivanja na mrežu

### F4: Logovanje (Serial/UART)

Ispisivanje dijagnostičkih poruka na serijski port za debugging:

- Startup: verzija firmware-a, učitani portovi, Mirage IP
- Mrežni status: IP adresa nakon povezivanja
- Konekcije: nova konekcija (src IP:port → dst port), uspešno prosleđivanje, greška, zatvaranje
- Greške: Mirage nedostupan, timeout, nedostatak resursa

## Nefunkcionalni zahtevi

### NF1: Platforma

- **RTOS**: FreeRTOS, Zephyr, NuttX, RIOT ili drugi besplatni RTOS
- **Board**: Po izboru studenta. Mora imati internet konekciju (WiFi i/ili Ethernet).
- **Jezik**: C, C++, Rust ili Go

### NF2: Ograničenja resursa

Embedded TCP/IP stack-ovi (lwIP i sl.) imaju ograničen broj soketa. Voditi računa o:

- Maksimalnom broju otvorenih soketa (listeners + aktivne konekcije)
- Alokaciji memorije za buffere
- Stack veličini RTOS task-ova

### NF3: Robusnost

- Firmware ne sme da "padne" — greške se loguju i obrađuju gracefully
- Ako Mirage server nije dostupan, konekcija ka klijentu se zatvara i IoT uređaj nastavlja normalan rad
- Ako se mrežna konekcija prekine, pokušati rekonekt i nastaviti sa radom

## Deliverables

1. **Izvorni kod** — firmware projekat sa jasnom strukturom
2. **Konfiguracija build-a** — Makefile, CMake, ili alat specifičan za platformu
3. **Dokumentacija**:
   - README sa uputstvima za build i flash
   - Opis arhitekture firmware-a (task-ovi, komunikacija između task-ova)
   - Opis izabranog board-a i razlozi za izbor
   - Poznata ograničenja
4. **Demo** — funkcionalna demonstracija: skeniranje IoT uređaja (npr. nmap), prikaz event-a na Mirage strani

## Testiranje

Ključni kriterijum: **napadač mora dobiti identičan odgovor bez obzira da li se konektuje direktno na Mirage ili na IoT uređaj.** IoT uređaj je transparentan proxy — ne sme da menja sadržaj koji prolazi kroz njega.

### Scenario

```bash
# Setup:
#   Mirage server: 192.168.1.100, port 80 u konfiguraciji
#   IoT uređaj:    192.168.1.200, konfigurisan sa mirage_ip: "192.168.1.100", ports: [80]

# Test 1 — direktno na Mirage:
nmap -sV -p 80 192.168.1.100

# Test 2 — preko IoT uređaja:
nmap -sV -p 80 192.168.1.200

# Očekivani rezultat: IDENTIČAN odgovor servisa u oba slučaja.
```

### Primer očekivanog odgovora

Direktno na Mirage (`nmap -sV -p 80 192.168.1.100`):

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41
```

Preko IoT uređaja (`nmap -sV -p 80 192.168.1.200`):

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41
```

Odgovori moraju biti isti jer IoT uređaj samo prosleđuje podatke — Mirage generiše odgovor u oba slučaja. Jedina razlika u Mirage logu je `src_ip`: u prvom slučaju je IP napadača, u drugom je IP IoT uređaja.
