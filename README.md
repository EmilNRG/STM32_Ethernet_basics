# NUCLEO-F429ZI: UDP-Daten per Ethernet an den Laptop senden

Dieses Projekt zeigt, wie ein **NUCLEO-F429ZI** über ein Netzwerkkabel direkt mit einem Windows-Laptop verbunden wird und jede Sekunde einen Text per **UDP** (User Datagram Protocol, ein einfaches Übertragungsprotokoll) an den Laptop schickt. Auf dem Laptop empfängt ein kurzes **PowerShell**-Skript die Daten. Es wird weder Python noch ein anderes Zusatzprogramm benötigt.

## Inhaltsverzeichnis

1. [Überblick](#1-überblick)
2. [Voraussetzungen](#2-voraussetzungen)
3. [Netzwerk-Grundlagen für Einsteiger](#3-netzwerk-grundlagen-für-einsteiger)
4. [Einrichtung Schritt für Schritt](#4-einrichtung-schritt-für-schritt)
5. [Der STM32-Code](#5-der-stm32-code)
6. [Der PowerShell-Empfänger](#6-der-powershell-empfänger)
7. [Board zu Board: Daten an ein zweites F429ZI senden und empfangen (Anleitung)](#7-board-zu-board-daten-an-ein-zweites-f429zi-senden-und-dort-empfangen-anleitung)
8. [Testen](#8-testen)
9. [Fehlersuche](#9-fehlersuche)
10. [Mögliche Erweiterungen](#10-mögliche-erweiterungen)

---

## 1. Überblick

| Gerät | IP-Adresse | Rolle |
|---|---|---|
| NUCLEO-F429ZI | `192.168.0.67` | Sender (UDP) |
| Laptop (Windows) | `192.168.0.20` | Empfänger auf **Port 5005** |

Netzmaske für beide: `255.255.255.0`. Ein Gateway wird nicht gebraucht, da es keinen Router gibt.

```
 STM32 (192.168.0.67)                          Laptop (192.168.0.20)
 ┌────────────────────┐   Netzwerkkabel   ┌──────────────────────────┐
 │ main.c             │ ================> │ PowerShell-Skript        │
 │  udp_sendto(...)   │   UDP, Port 5005  │  UdpClient(5005)         │
 │ LwIP + ETH + PHY   │                   │  Ausgabe im Terminal     │
 └────────────────────┘                   └──────────────────────────┘
```

Ablauf: Der STM32 baut jede Sekunde den Text `Hallo vom STM32: <Zähler>` zusammen und sendet ihn an `192.168.0.20:5005`. Das PowerShell-Skript wartet auf diesem Port und zeigt jedes empfangene Paket mit der Absenderadresse an.

Zusätzlich beschreibt Abschnitt 7, wie ein **zweites F429ZI-Board** (IP `192.168.0.68`, lauscht auf **Port 5006**) die Daten des ersten Boards empfängt, ganz ohne Laptop.

## 2. Voraussetzungen

**Hardware**

- NUCLEO-F429ZI (RJ45-Buchse und Ethernet-Chip sind bereits auf dem Board)
- Normales Netzwerkkabel (kein gekreuztes Kabel nötig, moderne Netzwerkkarten erkennen das automatisch, *Auto-MDI-X*)
- USB-Kabel für Stromversorgung, Flashen und Debuggen (ST-LINK)
- Windows-Laptop mit Ethernet-Anschluss (ggf. USB-Ethernet-Adapter)

**Software**

- STM32CubeMX (Konfigurationswerkzeug von ST, erzeugt den Initialisierungscode)
- Visual Studio Code mit der Erweiterung „STM32CubeIDE for Visual Studio Code" (Toolchain: **CMake**, im CubeMX unter *Project Manager → Project → Toolchain / IDE*)
- Windows PowerShell (in Windows enthalten)
- Optional: Wireshark (Programm zum Mitschneiden von Netzwerkverkehr) zur Fehlersuche

## 3. Netzwerk-Grundlagen für Einsteiger

### Glossar

| Begriff | Bedeutung |
|---|---|
| **Ethernet** | Standard für kabelgebundene Netzwerke |
| **MAC** (Media Access Control) | Im Mikrocontroller eingebaute Einheit, die Daten in Ethernet-Frames verpackt. Auch: die weltweit eindeutige Hardware-Adresse einer Netzwerkkarte |
| **PHY** (Physical Layer) | Chip auf dem Board (LAN8742), der die Signale für das Kabel erzeugt |
| **RMII** (Reduced Media Independent Interface) | Die Leitungen zwischen MAC und PHY |
| **DMA** (Direct Memory Access) | Einheit, die Daten ohne Zutun des Prozessors zwischen Speicher und Ethernet-Hardware verschiebt |
| **IP** (Internet Protocol) | Adressierung der Geräte, z. B. `192.168.0.67` |
| **Netzmaske** | Legt fest, welche Adressen zum selben Netz gehören. `255.255.255.0` bedeutet: Alle `192.168.0.x` liegen im selben Netz |
| **Gateway** | Router, über den man das eigene Netz verlässt. Hier nicht vorhanden (`0.0.0.0`) |
| **DHCP** (Dynamic Host Configuration Protocol) | Dienst, der IP-Adressen automatisch vergibt. Ohne Router gibt es keinen, deshalb feste IPs |
| **UDP** (User Datagram Protocol) | Einfaches Protokoll: Paket losschicken, ohne Verbindungsaufbau und ohne Empfangsbestätigung |
| **TCP** (Transmission Control Protocol) | Zuverlässiges Protokoll mit Verbindungsaufbau und Bestätigungen |
| **Port** | Nummer (0 bis 65535), die beim Empfänger das richtige Programm auswählt. Die IP ist die Hausadresse, der Port die Tür. Hier: **5005** |
| **ARP** (Address Resolution Protocol) | Findet zur IP-Adresse die passende MAC-Adresse heraus. Läuft automatisch |
| **ICMP** (Internet Control Message Protocol) | Wird vom `ping`-Befehl benutzt. LwIP beantwortet Pings selbständig |
| **LwIP** (Lightweight IP) | Fertige Software-Bibliothek für IP, UDP, TCP, ARP und ICMP, die CubeMX einbindet |
| **Frame** | Ein Ethernet-Datenpaket |
| **HAL** (Hardware Abstraction Layer) | Treiberbibliothek von ST |

### Wie ein Text zum Laptop kommt

Die Daten werden in mehrere „Hüllen" verpackt:

```
| Ethernet-Header | IP-Header | UDP-Header | deine Daten |
|  MAC-Adressen   | IP-Adressen | Ports     | Text        |
```

LwIP erledigt das Verpacken. Im eigenen Code werden nur Text, Ziel-IP und Ziel-Port angegeben.

## 4. Einrichtung Schritt für Schritt

### 4.1 Laptop

1. *Einstellungen → Netzwerk und Internet → Ethernet → IP-Zuweisung → Manuell* (IPv4 einschalten).
2. Eintragen:
   - IP-Adresse: `192.168.0.20`
   - Subnetzmaske: `255.255.255.0`
   - Gateway und DNS leer lassen
3. Netzwerkkabel zwischen Laptop und RJ45-Buchse des Nucleo stecken.

### 4.2 CubeMX

- *Connectivity → ETH*: Mode **RMII**
- *Middleware → LWIP*:
  - **DHCP** ausschalten
  - IP `192.168.0.67`, Netzmaske `255.255.255.0`, Gateway `0.0.0.0`
  - *Key Options*: **LWIP_UDP** auf *Enabled* (meist schon Standard)
  - *Platform Settings*: PHY-Treiber **LAN8742**
- *Project Manager → Project → Toolchain / IDE*: **CMake**
- Kein FreeRTOS aktivieren (LwIP läuft hier ohne Betriebssystem). Falls doch, muss im Pinout unter SYS die Timebase Source von SysTick auf einen freien Timer (z. B. TIM6) gestellt werden.
- Code generieren.

### 4.3 Code ergänzen, bauen, flashen

Den Code aus Abschnitt 5 in `Core/Src/main.c` einfügen, **nur zwischen den `USER CODE BEGIN/END`-Markierungen**. Alles außerhalb wird beim erneuten Generieren von CubeMX überschrieben. Danach in VS Code bauen und über den ST-LINK auf das Board laden.

## 5. Der STM32-Code

### 5.1 Vollständiger eigener Code in `main.c`

**Includes**

```c
/* USER CODE BEGIN Includes */
#include "lwip/udp.h"
#include <string.h>
#include <stdio.h>
/* USER CODE END Includes */
```

**Variablen**

```c
/* USER CODE BEGIN PV */
static struct udp_pcb *udp_tx;    /* PCB = Protocol Control Block, LwIPs Verbindungsstruktur */
static ip_addr_t pc_ip;           /* Ziel-IP: der Laptop */
static uint32_t letzter_versand = 0;
static uint32_t zaehler = 0;
/* USER CODE END PV */
```

**Funktionen**

```c
/* USER CODE BEGIN 0 */
#define PC_PORT 5005

static void udp_sender_init(void)
{
  udp_tx = udp_new();
  IP4_ADDR(&pc_ip, 192, 168, 0, 20);   /* IP-Adresse des Laptops */
}

static void udp_send_text(const char *text)
{
  uint16_t len = strlen(text);
  struct pbuf *p = pbuf_alloc(PBUF_TRANSPORT, len, PBUF_RAM);  /* pbuf = Paketpuffer */
  if (p != NULL)
  {
    pbuf_take(p, text, len);
    udp_sendto(udp_tx, p, &pc_ip, PC_PORT);
    pbuf_free(p);
  }
}
/* USER CODE END 0 */
```

**Initialisierung (nach `MX_LWIP_Init()`)**

```c
/* USER CODE BEGIN 2 */
udp_sender_init();
/* USER CODE END 2 */
```

**Hauptschleife**

```c
while (1)
{
  MX_LWIP_Process();

  if (HAL_GetTick() - letzter_versand >= 1000)   /* einmal pro Sekunde */
  {
    letzter_versand = HAL_GetTick();
    char msg[32];
    snprintf(msg, sizeof(msg), "Hallo vom STM32: %lu", (unsigned long)zaehler++);
    udp_send_text(msg);
  }
  /* USER CODE END WHILE */

  /* USER CODE BEGIN 3 */
}
/* USER CODE END 3 */
```

> **Wichtig:** `MX_LWIP_Process()` muss in jedem Schleifendurchlauf drankommen. Kein `HAL_Delay()` in der Schleife verwenden, sonst reagiert das Board nicht mehr auf Pings und eingehende Pakete.

### 5.2 Erklärung des eigenen Codes

**Includes**

- `lwip/udp.h`: UDP-Funktionen von LwIP (`udp_new`, `udp_sendto`, ...)
- `string.h`: `strlen()` liefert die Länge eines Textes
- `stdio.h`: `snprintf()` baut Text mit Zahlen zusammen

**Variablen**

- `static`: nur in dieser Datei sichtbar, vermeidet Namenskonflikte.
- `struct udp_pcb *udp_tx`: Zeiger auf die PCB (Protocol Control Block). Dort speichert LwIP alles zu einem UDP-„Kanal". Bei UDP gibt es keinen echten Verbindungsaufbau, die PCB ist nur der Sendekanal.
- `ip_addr_t pc_ip`: IP-Adresse des Ziels (Laptop).
- `uint32_t`: vorzeichenlose 32-Bit-Ganzzahl. `letzter_versand` merkt sich den Zeitpunkt des letzten Sendens, `zaehler` ist eine laufende Nummer, an der man am Laptop fehlende Pakete erkennt.

**`#define PC_PORT 5005`**

Textersetzung vor dem Übersetzen: Überall, wo `PC_PORT` steht, setzt der Compiler `5005` ein. Die Nummer muss zum Empfänger passen. Nummern unter 1024 sind für Systemdienste reserviert.

**`udp_sender_init()`**

- `udp_new()` legt die PCB an. Bei Speichermangel kommt `NULL` zurück.
- `IP4_ADDR(&pc_ip, 192, 168, 0, 20)` ist ein Makro von LwIP, das aus vier Zahlen die 32-Bit-IP-Adresse zusammensetzt. `&` übergibt die Adresse der Variable.

**`udp_send_text()`**

1. `strlen(text)` zählt die Zeichen (ohne das Ende-Zeichen `\0`).
2. `pbuf_alloc(PBUF_TRANSPORT, len, PBUF_RAM)` holt einen **pbuf** (Packet Buffer, LwIPs Paketspeicher). `PBUF_TRANSPORT` reserviert vor den Daten Platz für die UDP-, IP- und Ethernet-Header, `PBUF_RAM` bedeutet Speicher aus dem Heap.
3. `if (p != NULL)`: Schutz gegen vollen Speicher.
4. `pbuf_take()` kopiert den Text in den pbuf.
5. `udp_sendto()` sendet. LwIP ergänzt die Header, ermittelt per ARP die MAC-Adresse des Laptops und übergibt den Frame an den Ethernet-Treiber, der ihn per DMA an die Hardware weiterreicht.
6. `pbuf_free()` gibt den Speicher wieder frei. Ohne diesen Aufruf läuft der Speicher mit der Zeit voll.

Hinweis zu ARP: Vor dem allerersten Paket fragt LwIP per Broadcast „Wer hat 192.168.0.20?". Dabei kann das erste Paket verloren gehen. Das ist normal.

**Hauptschleife**

- `while (1)`: Endlosschleife, ein Mikrocontroller-Programm darf nie enden.
- `HAL_GetTick()` liefert die Millisekunden seit dem Start (vom SysTick, dem Systemzeitgeber des Prozessorkerns). Die Differenz `HAL_GetTick() - letzter_versand` funktioniert auch beim Zähler-Überlauf nach ca. 49 Tagen korrekt.
- `snprintf(msg, sizeof(msg), ...)` begrenzt die Textlänge auf die Puffergröße. `%lu` steht für eine vorzeichenlose `long`-Zahl, der Cast `(unsigned long)` passt den Typ an. `zaehler++` gibt den Wert aus und erhöht ihn danach um 1.

### 5.3 Von CubeMX erzeugter Code

**`main.c`**

| Funktion | Aufgabe |
|---|---|
| `HAL_Init()` | Startet die HAL, schaltet den Flash-Cache ein, stellt den SysTick auf 1 ms |
| `SystemClock_Config()` | Stellt die Takte ein (siehe unten) |
| `MX_GPIO_Init()` | Konfiguriert die Pins: LEDs LD1/LD2/LD3 als Ausgänge, blauer Taster als Eingang mit Interrupt, USB-Hilfspins |
| `MX_USART3_UART_Init()` | Serielle Schnittstelle (UART, Universal Asynchronous Receiver/Transmitter) über den ST-LINK, 115200 Baud. Für Ethernet nicht nötig, später nützlich für Debug-Ausgaben |
| `MX_USB_OTG_FS_PCD_Init()` | USB (On-The-Go, Full Speed). Für Ethernet nicht nötig |
| `MX_LWIP_Init()` | Startet LwIP und die Ethernet-Hardware |
| `Error_Handler()` | Schaltet Interrupts ab und bleibt in einer Endlosschleife. Bleibt das Board hier stehen, ist eine Initialisierung fehlgeschlagen |

**Takt in `SystemClock_Config()`**

- HSE (High Speed External, externer Takt) bekommt vom ST-LINK 8 MHz (`RCC_HSE_BYPASS`: fertiges Taktsignal, kein Quarz).
- PLL (Phase-Locked Loop, Taktvervielfacher): 8 MHz ÷ 4 = 2 MHz, × 168 = 336 MHz, ÷ 2 = **168 MHz** Systemtakt. `PLLQ = 7` ergibt 48 MHz für USB.
- Bus-Teiler: AHB ÷1 = 168 MHz, APB1 ÷4 = 42 MHz, APB2 ÷2 = 84 MHz.
- `FLASH_LATENCY_5`: 5 Wartetakte beim Lesen aus dem Flash bei 168 MHz.
- Ethernet braucht mindestens 25 MHz Bustakt, das ist erfüllt.

**`LWIP.c`**

| Element | Aufgabe |
|---|---|
| `struct netif gnetif` | LwIPs Beschreibung deiner Netzwerkschnittstelle (IP, MAC, Sende- und Empfangsfunktionen, Status) |
| `IP_ADDRESS[]`, `NETMASK_ADDRESS[]`, `GATEWAY_ADDRESS[]` | Adressen aus CubeMX als Einzelzahlen (hier `192.168.0.67`) |
| `MX_LWIP_Init()` | `lwip_init()` startet den Stack. `netif_add()` meldet die Schnittstelle an, mit `ethernetif_init` (initialisiert Ethernet-Hardware, MAC-Adresse, DMA-Puffer) und `ethernet_input` (Einstiegspunkt für empfangene Frames). Danach `netif_set_default()` und `netif_set_up()` |
| `MX_LWIP_Process()` | Muss zyklisch aufgerufen werden: `ethernetif_input()` holt empfangene Frames ab, `sys_check_timeouts()` erledigt LwIPs Zeitaufgaben (ohne Betriebssystem manuell nötig), `Ethernet_Link_Periodic_Handle()` prüft den Link |
| `Ethernet_Link_Periodic_Handle()` | Fragt alle 100 ms den PHY, ob ein Kabel steckt und mit welcher Geschwindigkeit und welchem Duplex-Modus verhandelt wurde |
| `ethernet_link_status_updated()` | Callback bei Link-Änderung, hier leer. Platz z. B. für eine LED |
| `sio_...`-Funktionen | Nur für den Keil-Compiler, werden bei GCC übersprungen |

Ein eingehender Ping wird von LwIP automatisch beantwortet (ICMP), ohne eigenen Code.

## 6. Der PowerShell-Empfänger

### 6.1 Das Skript

Als Datei `udp_empfaenger.ps1` speichern oder direkt in ein PowerShell-Fenster kopieren:

```powershell
$udp = New-Object System.Net.Sockets.UdpClient(5005)
$ep  = New-Object System.Net.IPEndPoint([System.Net.IPAddress]::Any, 0)
Write-Host "Warte auf Daten auf Port 5005 ... (Strg+C zum Beenden)"
try {
    while ($true) {
        if ($udp.Available -gt 0) {
            $bytes = $udp.Receive([ref]$ep)
            $text  = [System.Text.Encoding]::ASCII.GetString($bytes)
            Write-Host "$($ep.Address):$($ep.Port)  $text"
        } else {
            Start-Sleep -Milliseconds 50
        }
    }
}
finally {
    $udp.Close()
}
```

Falls das Ausführen einer `.ps1`-Datei von Windows blockiert wird (Execution Policy, Richtlinie für Skripte), das Skript einfach in das Fenster kopieren.

### 6.2 Erklärung

| Zeile | Bedeutung |
|---|---|
| `$udp = New-Object System.Net.Sockets.UdpClient(5005)` | Erzeugt ein UDP-Objekt aus dem **.NET Framework** (Microsoft-Programmbibliothek) und **bindet** es an Port 5005: Alle UDP-Pakete für diesen Port gehören ab jetzt dem Skript. Ein **Socket** (englisch „Steckdose") ist der Zugangspunkt eines Programms zum Netzwerk. Der Port ist danach exklusiv belegt, ein zweiter Start des Skripts gibt eine Fehlermeldung |
| `$ep = New-Object System.Net.IPEndPoint(...Any, 0)` | Ein **Endpoint** ist eine Kombination aus IP-Adresse und Port. Das hier ist ein leerer Platzhalter für den Absender. `Any` (`0.0.0.0`) heißt: von jedem Absender annehmen |
| `try { ... } finally { $udp.Close() }` | Sicherung: Egal wie die Schleife endet, auch bei Strg+C, wird der Port am Ende wieder freigegeben |
| `while ($true)` | Endlosschleife, Abbruch nur mit Strg+C |
| `$udp.Available -gt 0` | Anzahl der angekommenen, aber noch nicht abgeholten Bytes. `-gt` heißt „greater than" (größer als) |
| `$udp.Receive([ref]$ep)` | Holt ein Paket ab. `[ref]` übergibt den Platzhalter als Referenz, damit `Receive` hineinschreibt, von welcher IP und welchem Port das Paket kam. Ergebnis: ein Byte-Array (Zahlen von 0 bis 255) |
| `[System.Text.Encoding]::ASCII.GetString($bytes)` | Wandelt die Zahlen in Text um. **ASCII** (American Standard Code for Information Interchange) ordnet Zahlen Buchstaben zu, z. B. 72 = „H". Für Umlaute reicht ASCII nicht |
| `Write-Host "$($ep.Address):$($ep.Port)  $text"` | Gibt Absender-IP, Absender-Port und Text aus. `$( ... )` setzt einen Wert in den Text ein |
| `Start-Sleep -Milliseconds 50` | Wartet 50 ms, wenn nichts da ist, damit der Prozessor nicht dauernd ausgelastet wird |

**Warum die Abfrage mit `Available` statt direkt `Receive`?** `Receive` blockiert, bis ein Paket kommt, und währenddessen lässt sich das Skript nicht mit Strg+C abbrechen. Mit der Abfrage bleibt es jederzeit beendbar.

### 6.3 Windows-Firewall

Beim ersten Start fragt Windows, ob PowerShell auf Netzwerke zugreifen darf. **Zulassen**, für private **und** öffentliche Netzwerke, denn eine Direktverbindung ohne Router stuft Windows oft als „öffentlich" ein. Ohne Freigabe kommen die Pakete nie im Skript an. Falls keine Abfrage erscheint, in der Windows-Defender-Firewall eine eingehende Regel für **UDP-Port 5005** anlegen.

## 7. Board zu Board: Daten an ein zweites F429ZI senden und dort empfangen (Anleitung)

Bisher sendet das Board an den Laptop. Hier sendet **ein NUCLEO-F429ZI an ein zweites NUCLEO-F429ZI**. Dieser Abschnitt erklärt, **was du in welcher Reihenfolge einstellen und programmieren musst**. Den Code schreibst du selbst, die Anleitung sagt dir, welche Funktionen du brauchst, welche Parameter sie haben und worauf du achten musst. Eine Ausgabe über USART oder ein Terminalprogramm ist **nicht** nötig: Das empfangende Board zeigt den Empfang über eine LED und im Debugger.

### 7.1 Aufbau und Prinzip

| Board | Rolle | IP-Adresse | Port |
|---|---|---|---|
| Board A | Sender (das bisherige Projekt aus Abschnitt 5) | `192.168.0.67` | sendet an Board B |
| Board B | Empfänger (neu) | `192.168.0.68` | lauscht auf **5006** |

**Verkabelung:** Beide Boards hängen mit einem normalen Netzwerkkabel am Switch. Der Laptop kann ebenfalls am Switch hängen, er wird für diesen Test aber nicht gebraucht. Ein direktes Kabel zwischen den beiden Boards funktioniert normalerweise auch, weil der PHY-Chip (Physical Layer) die Leitungen bei Bedarf selbst vertauscht (*Auto-MDI-X*). Jedes Board bekommt außerdem über seinen USB-Anschluss Strom und ist darüber programmierbar.

**Prinzip:** Der Sender kennt Ziel-IP und Ziel-Port des Empfängers. Der Empfänger sagt LwIP, **auf welchem Port** es lauschen soll, und nennt eine eigene Funktion, die LwIP **aufruft**, sobald ein Paket ankommt. Eine solche Funktion nennt man **Callback** (Rückruffunktion). Du rufst sie also nie selbst auf, LwIP macht das.

Der Weg eines Pakets:

1. Board A baut den Text zusammen und sendet ihn an `192.168.0.68:5006`.
2. Der Switch leitet das Paket anhand der MAC-Adresse an den Anschluss von Board B weiter.
3. Auf Board B legt der Ethernet-Treiber den Frame im Empfangspuffer ab (per DMA, Direct Memory Access).
4. `MX_LWIP_Process()` in der Hauptschleife von Board B holt den Frame ab, LwIP entpackt die Header.
5. LwIP findet die Anmeldung für Port 5006 und ruft den Callback auf.
6. Der Callback kopiert die Daten heraus und setzt ein Flag (Merker). Die Hauptschleife erkennt das Flag und wertet aus.

Warum nicht alles im Callback erledigen? Ohne Betriebssystem läuft der Callback im normalen Programmfluss, nämlich während `MX_LWIP_Process()`. Er soll kurz sein, damit das Board nicht zu lange mit dem Netzwerk beschäftigt ist. Deshalb wird nur kopiert und gemerkt, ausgewertet wird in der Schleife.

### 7.2 Board B vorbereiten (CubeMX)

Board B bekommt ein eigenes Projekt, am einfachsten eine Kopie des bestehenden F429ZI-Projekts oder ein neu angelegtes Projekt mit denselben Einstellungen wie in Abschnitt 4.2. Zwei Dinge **müssen** sich vom Sender unterscheiden:

1. **Eigene IP-Adresse:** In CubeMX unter *Middleware → LWIP → General Settings* `192.168.0.68` eintragen. Netzmaske `255.255.255.0` und Gateway `0.0.0.0` bleiben wie bei Board A. DHCP bleibt **aus**. Zwei Geräte mit derselben IP-Adresse stören sich gegenseitig, die Kommunikation wird unzuverlässig.
2. **Eigene MAC-Adresse:** Der von CubeMX erzeugte Treiber setzt bei **jedem** Board dieselbe Standard-MAC (`00:80:E1:00:00:00`). Bei zwei Boards am selben Switch hat der Switch dann zwei Geräte mit derselben Hardware-Adresse, und die Pakete landen mal beim einen, mal beim anderen. Ändere bei **einem** der beiden Boards das letzte Byte, z. B. auf `01`. In CubeMX findest du die Einstellung unter *ETH → Parameter Settings* (Feld für die Ethernet-MAC-Adresse). Gibt es das Feld in deiner Version nicht, suche in `ethernetif.c` nach `MACAddr` und ändere dort das letzte Byte.

Alles andere (ETH auf RMII, LWIP_UDP aktiv, PHY-Treiber LAN8742, Toolchain CMake, kein FreeRTOS) ist wie in Abschnitt 4. Code generieren.

### 7.3 Board A anpassen (Sender)

Der Sendecode aus Abschnitt 5 bleibt bis auf **zwei Werte** gleich:

| Was | Bisher | Neu |
|---|---|---|
| Ziel-IP in `udp_sender_init()` (im Aufruf von `IP4_ADDR` die vier Zahlen) | `192.168.0.20` (Laptop) | `192.168.0.68` (Board B) |
| Ziel-Port (das `#define` für den Port, im Code `PC_PORT`) | `5005` | `5006` |

Zur besseren Lesbarkeit kannst du die Namen `pc_ip` und `PC_PORT` in etwas wie `ziel_ip` und `ZIEL_PORT` umbenennen. Dann musst du den neuen Namen aber an **allen** Stellen anpassen, an denen er benutzt wird.

Der Text („Hallo vom STM32: `<Zähler>`") und der 1-Sekunden-Takt bleiben unverändert. Board A sendet damit jede Sekunde ein Paket an Board B. Wenn du zusätzlich weiter an den Laptop senden möchtest, brauchst du für das zweite Ziel eine zweite Ziel-IP und einen zweiten Aufruf von `udp_sendto()`. Das ist für diesen Test nicht nötig.

### 7.4 Empfangscode für Board B schreiben

Alles gehört wie bisher **zwischen die `USER CODE BEGIN/END`-Markierungen** in `main.c`, sonst überschreibt CubeMX es beim nächsten Generieren. Zusätzliche Includes sind nicht nötig, `lwip/udp.h`, `string.h` und `stdio.h` kommen aus Abschnitt 5. Für Board B brauchst du den **Sendecode nicht**, nur den Empfang.

#### Schritt 1: Variablen anlegen

Ort: `USER CODE BEGIN PV` (Private Variables). Du brauchst fünf Variablen:

| Zweck | Typ | Hinweis |
|---|---|---|
| Empfangskanal | Zeiger auf `struct udp_pcb` | PCB = Protocol Control Block, LwIPs Verwaltungsstruktur für einen UDP-„Kanal" |
| Empfangspuffer | `char`-Feld, z. B. 128 Zeichen | Hier landen die empfangenen Daten. Die Größe bestimmt, wie viel Text maximal ankommt |
| Flag „neues Paket" | `uint8_t`, mit **`volatile`** | `volatile` sagt dem Compiler, dass sich der Wert „von außen" ändert (im Callback). Ohne dieses Wort kann der Compiler die Abfrage in der Schleife wegoptimieren, und du erkennst nie ein neues Paket |
| IP-Adresse des Absenders | `ip_addr_t` | Damit du im Debugger siehst, von wem das Paket kam |
| Port des Absenders | `uint16_t` | Ebenfalls zum Nachsehen im Debugger |

Gib allen Variablen `static`, damit sie nur in `main.c` sichtbar sind. Das Flag setzt du am Anfang auf 0. Merke dir die Namen, du suchst sie später im Debugger.

#### Schritt 2: Empfangsport festlegen

Ort: `USER CODE BEGIN 0`.

Lege mit `#define` einen Namen für die Portnummer an, z. B. `BOARD_PORT` mit dem Wert **5006**. Ein `#define` ist eine Textersetzung vor dem Übersetzen: Überall, wo der Name steht, setzt der Compiler die Zahl ein. Die Nummer muss **exakt** dem Ziel-Port in Board A entsprechen.

#### Schritt 3: Den Callback schreiben

Ort: `USER CODE BEGIN 0`, **vor** der Init-Funktion aus Schritt 4 (die Funktion muss dem Compiler bekannt sein, bevor sie angemeldet wird).

**Aufbau der Funktion:** Sie liefert nichts zurück (`void`) und hat **genau diese fünf Parameter in dieser Reihenfolge**, sonst meldet der Compiler einen Fehler beim Anmelden:

| Nr. | Parameter | Typ | Bedeutung |
|---|---|---|---|
| 1 | `arg` | `void *` | Ein frei wählbarer Zeiger, den du bei der Anmeldung mitgibst. Hier ungenutzt |
| 2 | `pcb` | `struct udp_pcb *` | Der Empfangskanal, über den das Paket kam. Brauchst du, wenn du direkt zurückantworten willst |
| 3 | `p` | `struct pbuf *` | Das empfangene Paket. **pbuf** = Packet Buffer, LwIPs Paketspeicher |
| 4 | `addr` | `const ip_addr_t *` | IP-Adresse des **Absenders** (als Zeiger) |
| 5 | `port` | `u16_t` | Port des Absenders |

**Was in der Funktion passiert, in dieser Reihenfolge:**

1. **Prüfen, ob `p` nicht `NULL` ist.** Nur dann gibt es Daten. Der ganze weitere Ablauf steht innerhalb dieser Prüfung.
2. **Länge bestimmen:** Die Gesamtlänge der Nutzdaten steht in `p->tot_len`. Ein pbuf kann aus mehreren verketteten Stücken bestehen, `tot_len` zählt alle zusammen.
3. **Länge begrenzen:** Ist das Paket größer als dein Empfangspuffer, kürzt du die Länge auf die Puffergröße **minus 1**. Sonst würdest du über das Ende des Puffers hinaus in fremden Speicher schreiben (*Pufferüberlauf*), das führt zu schwer auffindbaren Fehlern. Das „minus 1" reserviert Platz für das Text-Ende-Zeichen.
4. **Daten kopieren:** Mit `pbuf_copy_partial()`. Die Funktion kopiert über alle verketteten Stücke hinweg und braucht vier Angaben: den pbuf, den Zielpuffer, die Länge und den Startpunkt im Paket (hier **0**, also vom Anfang). Greife nicht direkt auf `p->payload` zu, das enthält nur das erste Stück.
5. **Text-Ende setzen:** Schreibe das Zeichen `'\0'` an die Stelle direkt hinter dem letzten kopierten Zeichen. C erkennt Text nur an diesem Ende-Zeichen. Ohne es zeigt der Debugger später Zeichen von älteren Paketen mit an.
6. **Absender merken:** `addr` ist ein **Zeiger**, der nur während des Callbacks gültig ist. Kopiere deshalb den **Inhalt** (`*addr`) in deine Variable, nicht den Zeiger selbst. Den Port speicherst du in der zweiten Variable.
7. **Flag setzen:** Auf 1, damit die Hauptschleife weiß, dass etwas Neues da ist.
8. **Paketspeicher freigeben: `pbuf_free(p)`.** Ganz wichtig! Bei UDP gehört das Paket nach dem Aufruf des Callbacks dir. Ohne Freigabe ist LwIPs Speicher nach einigen Paketen voll, und das Board empfängt nichts mehr. Das ist der häufigste Fehler.

#### Schritt 4: Die Init-Funktion schreiben

Ort: `USER CODE BEGIN 0`, **nach** dem Callback.

Eine `static void`-Funktion ohne Parameter, z. B. `udp_empfang_init`. Sie macht drei Dinge:

1. **`udp_new()`** erzeugt eine neue PCB und gibt den Zeiger zurück. Speichere ihn in der Empfangskanal-Variable aus Schritt 1. Schlägt das fehl (kein Speicher), kommt `NULL` zurück.
2. **`udp_bind()`** meldet die PCB für einen Port an. Übergeben werden drei Dinge: die PCB, die lokale IP-Adresse und der Port. Als IP-Adresse nimmst du `IP_ADDR_ANY`. Das bedeutet: Pakete an **jede** IP-Adresse des Boards annehmen, und zwar **von jedem Absender** im Netz. Als Port nimmst du deine Konstante `BOARD_PORT`.
3. **`udp_recv()`** meldet deinen Callback an. Drei Angaben: die PCB, **den Namen deiner Callback-Funktion** (ohne Klammern, du übergibst die Funktion selbst und rufst sie nicht auf) und den frei wählbaren Zeiger `arg` (hier `NULL`).

Die Reihenfolge ist wichtig: erst `udp_new`, dann `udp_bind`, dann `udp_recv`.

#### Schritt 5: Init-Funktion aufrufen

Ort: `USER CODE BEGIN 2`, also **nach** `MX_LWIP_Init()`. LwIP muss laufen, bevor du etwas anmeldest. Rufe dort die Init-Funktion aus Schritt 4 auf.

#### Schritt 6: Empfang in der Hauptschleife auswerten

Ort: innerhalb von `while (1)`. `MX_LWIP_Process()` muss in **jedem** Durchlauf stehen bleiben und darf nicht durch `HAL_Delay()` oder andere blockierende Befehle ausgebremst werden.

1. **Flag abfragen:** Ist es auf 1, gibt es etwas Neues.
2. **Flag sofort zurücksetzen** (auf 0). So erkennst du das nächste Paket.
3. **LED umschalten:** `HAL_GPIO_TogglePin()` mit Port und Pin der LED LD2 (`LD2_GPIO_Port`, `LD2_Pin`). Die LED wechselt bei jedem empfangenen Paket ihren Zustand. Sendet Board A jede Sekunde, blinkt sie im Sekundentakt.

Mehr ist für den Test nicht nötig. Die Daten selbst liegen im Empfangspuffer und lassen sich im Debugger ansehen (siehe 7.5).

#### Schritt 7: Bauen, flashen, prüfen

1. Projekt für Board B bauen. Typische Fehler: Callback hat falsche Parameter oder steht **nach** der Init-Funktion (dann kennt der Compiler den Namen noch nicht).
2. Auf Board B flashen. Dann Board A mit den geänderten Zielwerten flashen.
3. **Beide Boards am Laptop:** Jedes hat seinen eigenen ST-LINK. Beim Flashen und Debuggen musst du das richtige Board auswählen (die Boards unterscheiden sich über die Seriennummer des ST-LINK). Am sichersten ist es, beim Flashen nur das jeweilige Board per USB anzuschließen.

### 7.5 Ergebnis ohne USART ansehen

Es gibt zwei Wege, den Empfang auf Board B zu prüfen:

**LED:** LD2 auf Board B wechselt bei jedem Paket ihren Zustand. Wenn Board A jede Sekunde sendet, siehst du ein regelmäßiges Blinken im Sekundentakt. Das ist der schnellste Test.

**Debugger:** Starte Board B im Debug-Modus, lasse es laufen und ziehe die Variablen in die **Beobachtungsliste** (*Watch*, auch Live-Ausdrücke genannt):

| Variable | Erwarteter Inhalt |
|---|---|
| Empfangspuffer | Der Text, z. B. `Hallo vom STM32: 17`. Die Zahl am Ende ist der Zähler von Board A und steigt mit jedem Paket |
| Absender-IP | `192.168.0.67` (siehe Hinweis unten) |
| Absender-Port | Eine hohe Zahl, die Windows- bzw. LwIP-seitig zugeteilt wird, z. B. `49152` |

**Hinweis zur IP-Darstellung:** Der Debugger zeigt die IP-Adresse als 32-Bit-Zahl, und zwar in umgekehrter Byte-Reihenfolge (*Little Endian*, der Prozessor speichert das niedrigste Byte zuerst). `192.168.0.67` erscheint daher als `0x4300A8C0` (`C0` = 192, `A8` = 168, `00` = 0, `43` = 67).

Auch der Zähler im Text hilft: Springt er zwischen zwei Paketen um mehr als 1, ist ein Paket verloren gegangen.

### 7.6 Empfänger vorab mit dem Laptop testen (PowerShell, optional)

Wenn Board B nicht klappt, kannst du es ohne Board A prüfen, indem der Laptop ein Paket an Board B schickt. In ein PowerShell-Fenster kopieren und Enter drücken (der Laptop muss mit am Switch hängen und die IP `192.168.0.20` haben):

```powershell
$c = New-Object System.Net.Sockets.UdpClient
$bytes = [System.Text.Encoding]::ASCII.GetBytes("Hallo STM32")
$c.Send($bytes, $bytes.Length, "192.168.0.68", 5006) | Out-Null
$c.Close()
```

Was die Zeilen tun:

1. **UDP-Objekt erzeugen:** Die .NET-Klasse `System.Net.Sockets.UdpClient`, aber **ohne Portangabe**. Das Skript muss hier nicht lauschen, es sendet nur. Windows wählt selbst einen freien Absenderport.
2. **Text in Bytes umwandeln:** `ASCII.GetBytes()` wandelt Text in Zahlen um. Das ist das Gegenstück zu `GetString` im Empfänger aus Abschnitt 6. ASCII (American Standard Code for Information Interchange) ordnet Buchstaben Zahlen zu. Umlaute werden damit nicht korrekt übertragen.
3. **Senden:** `Send()` bekommt vier Angaben: die Bytes, deren Anzahl, die **Ziel-IP** und den **Ziel-Port**.
4. **Ausgabe unterdrücken:** `Send()` gibt die Zahl der gesendeten Bytes zurück. Mit `Out-Null` bleibt das Fenster sauber.
5. **Aufräumen:** `Close()` gibt die Ressourcen frei.

Jeder Durchlauf schickt **ein** Paket. LD2 auf Board B wechselt danach ihren Zustand, und der Empfangspuffer enthält `Hallo STM32`. Klappt das, ist Board B in Ordnung, und der Fehler liegt auf der Seite von Board A. Eine Oberfläche statt PowerShell bietet das Programm **Packet Sender** (kostenlos, packetsender.com): Ziel `192.168.0.68`, Port `5006`, Protokoll **UDP**.

### 7.7 Häufige Fehler beim Schreiben und Einrichten

| Fehler | Folge |
|---|---|
| Beide Boards haben dieselbe IP-Adresse | Unzuverlässige oder gar keine Kommunikation |
| Beide Boards haben dieselbe MAC-Adresse | Der Switch kann die Pakete nicht eindeutig zuordnen, sie landen beim falschen Board oder gehen verloren |
| Ziel-IP oder Ziel-Port in Board A passen nicht zu Board B | Das Paket kommt nie im Callback an |
| `pbuf_free()` im Callback vergessen | Empfang klappt einige Male, dann nie wieder (Speicher voll) |
| Länge nicht auf die Puffergröße begrenzt | Zufällige Abstürze oder verfälschte Variablen bei großen Paketen |
| Text-Ende-Zeichen `'\0'` nicht gesetzt | Im Debugger stehen Reste älterer Pakete hinter dem Text |
| `volatile` beim Flag vergessen | Die Schleife erkennt neue Pakete nicht |
| Absender nur als Zeiger gespeichert statt den Inhalt zu kopieren | Falsche oder zufällige Adresse im Debugger |
| Init-Funktion nicht aufgerufen | Es kommt nie ein Callback, das Board antwortet auf Pings, aber nicht auf die Daten |
| `MX_LWIP_Process()` fehlt oder die Schleife wird durch `HAL_Delay()` blockiert | Weder Ping noch Daten kommen an |
| Callback-Parameter in falscher Reihenfolge oder mit falschem Typ | Compilerfehler oder Warnung bei `udp_recv()` |

### 7.8 Wichtige Hinweise

- **Reihenfolge beim Einschalten:** Es spielt keine Rolle, welches Board zuerst startet. Da UDP keinen Verbindungsaufbau hat, gehen Pakete verloren, solange Board B noch nicht lauscht, das ist normal. Das allererste Paket kann zusätzlich fehlen, weil Board A erst per **ARP** (Address Resolution Protocol, ordnet IP-Adressen MAC-Adressen zu) die MAC-Adresse von Board B erfragen muss.
- **Text oder Rohdaten:** Die Anleitung geht von Text aus. Bei Binärdaten (z. B. Messwerte als Zahlen) hängst du kein Ende-Zeichen an, sondern liest die einzelnen Bytes des Puffers und merkst dir die empfangene Länge in einer eigenen Variable.
- **Mehrere Pakete kurz hintereinander:** Mit einem Puffer und einem Flag geht ein zweites Paket verloren oder überschreibt das erste, solange die Schleife das erste noch nicht abgeholt hat. Für schnelle Datenströme braucht man später einen Ringpuffer (Warteschlange).
- **LED nach Inhalt schalten:** Statt bei jedem Paket umzuschalten, kannst du den empfangenen Text auswerten, z. B. mit `strncmp()` auf einen Befehl wie `LED AN` prüfen, und die LED entsprechend schalten.
- **Antworten an den Absender:** Im Callback hast du Adresse und Port des Absenders. Mit `udp_sendto()` und einem neuen pbuf kann Board B direkt zurückantworten. So bauen beide Boards eine Zweiwege-Verbindung auf. Jedes Board braucht dann **Sende- und Empfangsteil**, mit jeweils eigener PCB.
- **Firewall:** Zwischen den Boards spielt die Windows-Firewall keine Rolle, der Laptop ist am Weg nicht beteiligt.
- **Mitschneiden mit Wireshark:** Ein einfacher (unmanaged) Switch schickt die Pakete nur an den Anschluss des Empfängers. Der Laptop sieht den Verkehr zwischen den beiden Boards deshalb nicht mit. Dafür bräuchte man einen Switch mit Spiegel-Port.

## 8. Testen

### 8.1 Ping

In der PowerShell:

```
ping 192.168.0.67
```

Antworten vom Board bedeuten, dass Kabel, IP-Einstellungen, Ethernet-Hardware und LwIP funktionieren.

> **Falle: führende Nullen.** Windows liest Zahlen mit führender Null als Oktalzahlen (Basis 8). `ping 192.168.000.010` pingt `192.168.0.8`, weil `010` oktal 8 ist, und `067` wäre 55. IP-Adressen immer ohne führende Nullen schreiben.

> **Falle: „Zielhost nicht erreichbar".** Kommt die Meldung von der **eigenen** Laptop-IP (`192.168.0.20`), hat der Laptop per ARP niemanden gefunden. Das Board antwortet dann nicht, das Paket zählt in der Statistik trotzdem als „Empfangen".

### 8.2 UDP-Empfang am Laptop (Board sendet)

1. PowerShell-Skript aus Abschnitt 6 starten.
2. STM32 flashen und starten.
3. Im Fenster sollte jede Sekunde eine Zeile erscheinen:

```
192.168.0.67:49152  Hallo vom STM32: 3
```

Das allererste Paket kann fehlen (ARP).

### 8.3 Board-zu-Board-Test (Abschnitt 7)

1. Beide Boards (Board A `192.168.0.67`, Board B `192.168.0.68`) mit Netzwerkkabeln an den Switch anschließen und mit Strom versorgen.
2. Board B flashen und starten, danach Board A.
3. LD2 auf Board B wechselt jede Sekunde den Zustand.
4. Im Debugger von Board B enthält der Empfangspuffer einen Text wie `Hallo vom STM32: 17`, dessen Zähler mit jedem Paket steigt.
5. Optional: Mit `ping 192.168.0.68` vom Laptop prüfen, ob Board B im Netz erreichbar ist.

## 9. Fehlersuche

| Symptom | Mögliche Ursache und Abhilfe |
|---|---|
| Keine Link-LED an der RJ45-Buchse | Kabel prüfen, ETH auf RMII, PHY-Treiber LAN8742 in LWIP Platform Settings, Takt prüfen |
| Ping ohne Antwort, LED leuchtet | Pingt man die richtige Adresse (`192.168.0.67`)? Läuft `MX_LWIP_Process()` in der Schleife? Laptop im selben Subnetz? Firewall blockiert ICMP? |
| Programm startet nicht richtig | Im Debugger pausieren. Steht es in `Error_Handler()`, ist eine Initialisierung fehlgeschlagen. Der Aufrufstapel (Call Stack) zeigt welche |
| Ping geht, aber im Skript kommt nichts an | Firewall-Freigabe für PowerShell bzw. UDP-Port 5005 prüfen. Laptop-IP im Code (`192.168.0.20`) mit `ipconfig` vergleichen. Stimmt der Port auf beiden Seiten (5005)? |
| Fehler beim Start des Skripts, Port belegt | Ein anderes Fenster hält den Port noch. Fenster schließen und erneut starten |
| Compilerfehler `lwip/udp.h` nicht gefunden | In CubeMX LWIP_UDP aktivieren und Code neu generieren |
| Eintrag in der ARP-Tabelle prüfen | `arp -a` zeigt, ob für `192.168.0.67` eine MAC-Adresse bekannt ist |
| Unklar, wo das Problem liegt | **Wireshark** auf dem Ethernet-Adapter mit Filter `udp.port == 5005`. Kommen die Pakete an, aber das Skript zeigt nichts, liegt es an der Firewall. Kommen sie nicht an, liegt es am Board oder am Code |
| Board B empfängt nichts (LD2 schaltet nicht um) | Stimmen Ziel-IP (`192.168.0.68`) und Ziel-Port (5006) in Board A? Wird `udp_empfang_init()` in `USER CODE BEGIN 2` aufgerufen? Läuft `MX_LWIP_Process()` in jedem Schleifendurchlauf? Antwortet Board B auf `ping 192.168.0.68` vom Laptop? Hilfreich ist auch der Test mit dem PowerShell-Befehl aus Abschnitt 7.6 |
| Empfang klappt nur einige Male, dann nicht mehr | Im Callback fehlt `pbuf_free(p)`, der Paketspeicher läuft voll |
| Beide Boards antworten durcheinander oder Pakete verschwinden | Haben beide Boards dieselbe IP oder dieselbe MAC-Adresse? Jedes Board braucht eine eigene (Abschnitt 7.2) |
| `main.c` ist nach dem Generieren verändert | Eigener Code stand außerhalb der `USER CODE`-Markierungen. Unter *Project Manager → Code Generator* muss „Keep User Code when re-generating" aktiv sein |

## 10. Mögliche Erweiterungen

- Statt des Zählers echte Messwerte senden, z. B. mit `snprintf(msg, sizeof(msg), "T=%.1f", temperatur)`. Dafür muss in den Projekteinstellungen Float-Unterstützung für `printf` aktiviert sein.
- Kommando-System: Im Empfangs-Callback den Text auswerten (z. B. `LED1 AN` / `LED1 AUS`), die LED entsprechend schalten und mit `udp_sendto()` eine Bestätigung an den Absender zurückschicken.
- Ringpuffer statt einzelnem Empfangspuffer, damit bei schnellen Datenströmen keine Pakete verloren gehen.
- Rückgabewert von `udp_sendto()` (Typ `err_t`) und von `udp_new()` auswerten und Fehler melden.
- Debug-Ausgaben über USART3 (virtueller COM-Port des ST-LINK).
- Umstieg auf **TCP**, wenn zuverlässige Übertragung nötig ist.