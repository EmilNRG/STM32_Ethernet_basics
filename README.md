Vibecode lol:



# NUCLEO-F429ZI: UDP-Daten per Ethernet an den Laptop senden



Dieses Projekt zeigt, wie ein \*\*NUCLEO-F429ZI\*\* über ein Netzwerkkabel direkt mit einem Windows-Laptop verbunden wird und jede Sekunde einen Text per \*\*UDP\*\* (User Datagram Protocol, ein einfaches Übertragungsprotokoll) an den Laptop schickt. Auf dem Laptop empfängt ein kurzes \*\*PowerShell\*\*-Skript die Daten. Es wird weder Python noch ein anderes Zusatzprogramm benötigt.



\## Inhaltsverzeichnis



1\. \[Überblick](#1-überblick)

2\. \[Voraussetzungen](#2-voraussetzungen)

3\. \[Netzwerk-Grundlagen für Einsteiger](#3-netzwerk-grundlagen-für-einsteiger)

4\. \[Einrichtung Schritt für Schritt](#4-einrichtung-schritt-für-schritt)

5\. \[Der STM32-Code](#5-der-stm32-code)

6\. \[Der PowerShell-Empfänger](#6-der-powershell-empfänger)

7\. \[Testen](#7-testen)

8\. \[Fehlersuche](#8-fehlersuche)

9\. \[Mögliche Erweiterungen](#9-mögliche-erweiterungen)



\---



\## 1. Überblick



| Gerät | IP-Adresse | Rolle |

|---|---|---|

| NUCLEO-F429ZI | `192.168.0.67` | Sender (UDP) |

| Laptop (Windows) | `192.168.0.20` | Empfänger auf \*\*Port 5005\*\* |



Netzmaske für beide: `255.255.255.0`. Ein Gateway wird nicht gebraucht, da es keinen Router gibt.



```

&#x20;STM32 (192.168.0.67)                          Laptop (192.168.0.20)

&#x20;┌────────────────────┐   Netzwerkkabel   ┌──────────────────────────┐

&#x20;│ main.c             	    │ ================> │ PowerShell-Skript     		    │

&#x20;│  udp\_sendto(...)   		│   UDP, Port 5005  │  UdpClient(5005)     			    │

&#x20;│ LwIP + ETH + PHY  	    │                   │  Ausgabe im Terminal  		    │

&#x20;└────────────────────┘                   └──────────────────────────┘

```



Ablauf: Der STM32 baut jede Sekunde den Text `Hallo vom STM32: <Zähler>` zusammen und sendet ihn an `192.168.0.20:5005`. Das PowerShell-Skript wartet auf diesem Port und zeigt jedes empfangene Paket mit der Absenderadresse an.



\## 2. Voraussetzungen



\*\*Hardware\*\*



\- NUCLEO-F429ZI (RJ45-Buchse und Ethernet-Chip sind bereits auf dem Board)

\- Normales Netzwerkkabel (kein gekreuztes Kabel nötig, moderne Netzwerkkarten erkennen das automatisch, \*Auto-MDI-X\*)

\- USB-Kabel für Stromversorgung, Flashen und Debuggen (ST-LINK)

\- Windows-Laptop mit Ethernet-Anschluss (ggf. USB-Ethernet-Adapter)



\*\*Software\*\*



\- STM32CubeMX (Konfigurationswerkzeug von ST, erzeugt den Initialisierungscode)

\- Visual Studio Code mit der Erweiterung „STM32CubeIDE for Visual Studio Code" (Toolchain: \*\*CMake\*\*, im CubeMX unter \*Project Manager → Project → Toolchain / IDE\*)

\- Windows PowerShell (in Windows enthalten)

\- Optional: Wireshark (Programm zum Mitschneiden von Netzwerkverkehr) zur Fehlersuche



\## 3. Netzwerk-Grundlagen für Einsteiger



\### Glossar



| Begriff | Bedeutung |

|---|---|

| \*\*Ethernet\*\* | Standard für kabelgebundene Netzwerke |

| \*\*MAC\*\* (Media Access Control) | Im Mikrocontroller eingebaute Einheit, die Daten in Ethernet-Frames verpackt. Auch: die weltweit eindeutige Hardware-Adresse einer Netzwerkkarte |

| \*\*PHY\*\* (Physical Layer) | Chip auf dem Board (LAN8742), der die Signale für das Kabel erzeugt |

| \*\*RMII\*\* (Reduced Media Independent Interface) | Die Leitungen zwischen MAC und PHY |

| \*\*DMA\*\* (Direct Memory Access) | Einheit, die Daten ohne Zutun des Prozessors zwischen Speicher und Ethernet-Hardware verschiebt |

| \*\*IP\*\* (Internet Protocol) | Adressierung der Geräte, z. B. `192.168.0.67` |

| \*\*Netzmaske\*\* | Legt fest, welche Adressen zum selben Netz gehören. `255.255.255.0` bedeutet: Alle `192.168.0.x` liegen im selben Netz |

| \*\*Gateway\*\* | Router, über den man das eigene Netz verlässt. Hier nicht vorhanden (`0.0.0.0`) |

| \*\*DHCP\*\* (Dynamic Host Configuration Protocol) | Dienst, der IP-Adressen automatisch vergibt. Ohne Router gibt es keinen, deshalb feste IPs |

| \*\*UDP\*\* (User Datagram Protocol) | Einfaches Protokoll: Paket losschicken, ohne Verbindungsaufbau und ohne Empfangsbestätigung |

| \*\*TCP\*\* (Transmission Control Protocol) | Zuverlässiges Protokoll mit Verbindungsaufbau und Bestätigungen |

| \*\*Port\*\* | Nummer (0 bis 65535), die beim Empfänger das richtige Programm auswählt. Die IP ist die Hausadresse, der Port die Tür. Hier: \*\*5005\*\* |

| \*\*ARP\*\* (Address Resolution Protocol) | Findet zur IP-Adresse die passende MAC-Adresse heraus. Läuft automatisch |

| \*\*ICMP\*\* (Internet Control Message Protocol) | Wird vom `ping`-Befehl benutzt. LwIP beantwortet Pings selbständig |

| \*\*LwIP\*\* (Lightweight IP) | Fertige Software-Bibliothek für IP, UDP, TCP, ARP und ICMP, die CubeMX einbindet |

| \*\*Frame\*\* | Ein Ethernet-Datenpaket |

| \*\*HAL\*\* (Hardware Abstraction Layer) | Treiberbibliothek von ST |



\### Wie ein Text zum Laptop kommt



Die Daten werden in mehrere „Hüllen" verpackt:



```

| Ethernet-Header | IP-Header | UDP-Header | deine Daten |

|  MAC-Adressen   | IP-Adressen | Ports     | Text        |

```



LwIP erledigt das Verpacken. Im eigenen Code werden nur Text, Ziel-IP und Ziel-Port angegeben.



\## 4. Einrichtung Schritt für Schritt



\### 4.1 Laptop



1\. \*Einstellungen → Netzwerk und Internet → Ethernet → IP-Zuweisung → Manuell\* (IPv4 einschalten).

2\. Eintragen:

&#x20;  - IP-Adresse: `192.168.0.20`

&#x20;  - Subnetzmaske: `255.255.255.0`

&#x20;  - Gateway und DNS leer lassen

3\. Netzwerkkabel zwischen Laptop und RJ45-Buchse des Nucleo stecken.



\### 4.2 CubeMX



\- \*Connectivity → ETH\*: Mode \*\*RMII\*\*

\- \*Middleware → LWIP\*:

&#x20; - \*\*DHCP\*\* ausschalten

&#x20; - IP `192.168.0.67`, Netzmaske `255.255.255.0`, Gateway `0.0.0.0`

&#x20; - \*Key Options\*: \*\*LWIP\_UDP\*\* auf \*Enabled\* (meist schon Standard)

&#x20; - \*Platform Settings\*: PHY-Treiber \*\*LAN8742\*\*

\- \*Project Manager → Project → Toolchain / IDE\*: \*\*CMake\*\*

\- Kein FreeRTOS aktivieren (LwIP läuft hier ohne Betriebssystem). Falls doch, muss im Pinout unter SYS die Timebase Source von SysTick auf einen freien Timer (z. B. TIM6) gestellt werden.

\- Code generieren.



\### 4.3 Code ergänzen, bauen, flashen



Den Code aus Abschnitt 5 in `Core/Src/main.c` einfügen, \*\*nur zwischen den `USER CODE BEGIN/END`-Markierungen\*\*. Alles außerhalb wird beim erneuten Generieren von CubeMX überschrieben. Danach in VS Code bauen und über den ST-LINK auf das Board laden.



\## 5. Der STM32-Code



\### 5.1 Vollständiger eigener Code in `main.c`



\*\*Includes\*\*



```c

/\* USER CODE BEGIN Includes \*/

\#include "lwip/udp.h"

\#include <string.h>

\#include <stdio.h>

/\* USER CODE END Includes \*/

```



\*\*Variablen\*\*



```c

/\* USER CODE BEGIN PV \*/

static struct udp\_pcb \*udp\_tx;    /\* PCB = Protocol Control Block, LwIPs Verbindungsstruktur \*/

static ip\_addr\_t pc\_ip;           /\* Ziel-IP: der Laptop \*/

static uint32\_t letzter\_versand = 0;

static uint32\_t zaehler = 0;

/\* USER CODE END PV \*/

```



\*\*Funktionen\*\*



```c

/\* USER CODE BEGIN 0 \*/

\#define PC\_PORT 5005



static void udp\_sender\_init(void)

{

&#x20; udp\_tx = udp\_new();

&#x20; IP4\_ADDR(\&pc\_ip, 192, 168, 0, 20);   /\* IP-Adresse des Laptops \*/

}



static void udp\_send\_text(const char \*text)

{

&#x20; uint16\_t len = strlen(text);

&#x20; struct pbuf \*p = pbuf\_alloc(PBUF\_TRANSPORT, len, PBUF\_RAM);  /\* pbuf = Paketpuffer \*/

&#x20; if (p != NULL)

&#x20; {

&#x20;   pbuf\_take(p, text, len);

&#x20;   udp\_sendto(udp\_tx, p, \&pc\_ip, PC\_PORT);

&#x20;   pbuf\_free(p);

&#x20; }

}

/\* USER CODE END 0 \*/

```



\*\*Initialisierung (nach `MX\_LWIP\_Init()`)\*\*



```c

/\* USER CODE BEGIN 2 \*/

udp\_sender\_init();

/\* USER CODE END 2 \*/

```



\*\*Hauptschleife\*\*



```c

while (1)

{

&#x20; MX\_LWIP\_Process();



&#x20; if (HAL\_GetTick() - letzter\_versand >= 1000)   /\* einmal pro Sekunde \*/

&#x20; {

&#x20;   letzter\_versand = HAL\_GetTick();

&#x20;   char msg\[32];

&#x20;   snprintf(msg, sizeof(msg), "Hallo vom STM32: %lu", (unsigned long)zaehler++);

&#x20;   udp\_send\_text(msg);

&#x20; }

&#x20; /\* USER CODE END WHILE \*/



&#x20; /\* USER CODE BEGIN 3 \*/

}

/\* USER CODE END 3 \*/

```



> \*\*Wichtig:\*\* `MX\_LWIP\_Process()` muss in jedem Schleifendurchlauf drankommen. Kein `HAL\_Delay()` in der Schleife verwenden, sonst reagiert das Board nicht mehr auf Pings und eingehende Pakete.



\### 5.2 Erklärung des eigenen Codes



\*\*Includes\*\*



\- `lwip/udp.h`: UDP-Funktionen von LwIP (`udp\_new`, `udp\_sendto`, ...)

\- `string.h`: `strlen()` liefert die Länge eines Textes

\- `stdio.h`: `snprintf()` baut Text mit Zahlen zusammen



\*\*Variablen\*\*



\- `static`: nur in dieser Datei sichtbar, vermeidet Namenskonflikte.

\- `struct udp\_pcb \*udp\_tx`: Zeiger auf die PCB (Protocol Control Block). Dort speichert LwIP alles zu einem UDP-„Kanal". Bei UDP gibt es keinen echten Verbindungsaufbau, die PCB ist nur der Sendekanal.

\- `ip\_addr\_t pc\_ip`: IP-Adresse des Ziels (Laptop).

\- `uint32\_t`: vorzeichenlose 32-Bit-Ganzzahl. `letzter\_versand` merkt sich den Zeitpunkt des letzten Sendens, `zaehler` ist eine laufende Nummer, an der man am Laptop fehlende Pakete erkennt.



\*\*`#define PC\_PORT 5005`\*\*



Textersetzung vor dem Übersetzen: Überall, wo `PC\_PORT` steht, setzt der Compiler `5005` ein. Die Nummer muss zum Empfänger passen. Nummern unter 1024 sind für Systemdienste reserviert.



\*\*`udp\_sender\_init()`\*\*



\- `udp\_new()` legt die PCB an. Bei Speichermangel kommt `NULL` zurück.

\- `IP4\_ADDR(\&pc\_ip, 192, 168, 0, 20)` ist ein Makro von LwIP, das aus vier Zahlen die 32-Bit-IP-Adresse zusammensetzt. `\&` übergibt die Adresse der Variable.



\*\*`udp\_send\_text()`\*\*



1\. `strlen(text)` zählt die Zeichen (ohne das Ende-Zeichen `\\0`).

2\. `pbuf\_alloc(PBUF\_TRANSPORT, len, PBUF\_RAM)` holt einen \*\*pbuf\*\* (Packet Buffer, LwIPs Paketspeicher). `PBUF\_TRANSPORT` reserviert vor den Daten Platz für die UDP-, IP- und Ethernet-Header, `PBUF\_RAM` bedeutet Speicher aus dem Heap.

3\. `if (p != NULL)`: Schutz gegen vollen Speicher.

4\. `pbuf\_take()` kopiert den Text in den pbuf.

5\. `udp\_sendto()` sendet. LwIP ergänzt die Header, ermittelt per ARP die MAC-Adresse des Laptops und übergibt den Frame an den Ethernet-Treiber, der ihn per DMA an die Hardware weiterreicht.

6\. `pbuf\_free()` gibt den Speicher wieder frei. Ohne diesen Aufruf läuft der Speicher mit der Zeit voll.



Hinweis zu ARP: Vor dem allerersten Paket fragt LwIP per Broadcast „Wer hat 192.168.0.20?". Dabei kann das erste Paket verloren gehen. Das ist normal.



\*\*Hauptschleife\*\*



\- `while (1)`: Endlosschleife, ein Mikrocontroller-Programm darf nie enden.

\- `HAL\_GetTick()` liefert die Millisekunden seit dem Start (vom SysTick, dem Systemzeitgeber des Prozessorkerns). Die Differenz `HAL\_GetTick() - letzter\_versand` funktioniert auch beim Zähler-Überlauf nach ca. 49 Tagen korrekt.

\- `snprintf(msg, sizeof(msg), ...)` begrenzt die Textlänge auf die Puffergröße. `%lu` steht für eine vorzeichenlose `long`-Zahl, der Cast `(unsigned long)` passt den Typ an. `zaehler++` gibt den Wert aus und erhöht ihn danach um 1.



\### 5.3 Von CubeMX erzeugter Code



\*\*`main.c`\*\*



| Funktion | Aufgabe |

|---|---|

| `HAL\_Init()` | Startet die HAL, schaltet den Flash-Cache ein, stellt den SysTick auf 1 ms |

| `SystemClock\_Config()` | Stellt die Takte ein (siehe unten) |

| `MX\_GPIO\_Init()` | Konfiguriert die Pins: LEDs LD1/LD2/LD3 als Ausgänge, blauer Taster als Eingang mit Interrupt, USB-Hilfspins |

| `MX\_USART3\_UART\_Init()` | Serielle Schnittstelle (UART, Universal Asynchronous Receiver/Transmitter) über den ST-LINK, 115200 Baud. Für Ethernet nicht nötig, später nützlich für Debug-Ausgaben |

| `MX\_USB\_OTG\_FS\_PCD\_Init()` | USB (On-The-Go, Full Speed). Für Ethernet nicht nötig |

| `MX\_LWIP\_Init()` | Startet LwIP und die Ethernet-Hardware |

| `Error\_Handler()` | Schaltet Interrupts ab und bleibt in einer Endlosschleife. Bleibt das Board hier stehen, ist eine Initialisierung fehlgeschlagen |



\*\*Takt in `SystemClock\_Config()`\*\*



\- HSE (High Speed External, externer Takt) bekommt vom ST-LINK 8 MHz (`RCC\_HSE\_BYPASS`: fertiges Taktsignal, kein Quarz).

\- PLL (Phase-Locked Loop, Taktvervielfacher): 8 MHz ÷ 4 = 2 MHz, × 168 = 336 MHz, ÷ 2 = \*\*168 MHz\*\* Systemtakt. `PLLQ = 7` ergibt 48 MHz für USB.

\- Bus-Teiler: AHB ÷1 = 168 MHz, APB1 ÷4 = 42 MHz, APB2 ÷2 = 84 MHz.

\- `FLASH\_LATENCY\_5`: 5 Wartetakte beim Lesen aus dem Flash bei 168 MHz.

\- Ethernet braucht mindestens 25 MHz Bustakt, das ist erfüllt.



\*\*`LWIP.c`\*\*



| Element | Aufgabe |

|---|---|

| `struct netif gnetif` | LwIPs Beschreibung deiner Netzwerkschnittstelle (IP, MAC, Sende- und Empfangsfunktionen, Status) |

| `IP\_ADDRESS\[]`, `NETMASK\_ADDRESS\[]`, `GATEWAY\_ADDRESS\[]` | Adressen aus CubeMX als Einzelzahlen (hier `192.168.0.67`) |

| `MX\_LWIP\_Init()` | `lwip\_init()` startet den Stack. `netif\_add()` meldet die Schnittstelle an, mit `ethernetif\_init` (initialisiert Ethernet-Hardware, MAC-Adresse, DMA-Puffer) und `ethernet\_input` (Einstiegspunkt für empfangene Frames). Danach `netif\_set\_default()` und `netif\_set\_up()` |

| `MX\_LWIP\_Process()` | Muss zyklisch aufgerufen werden: `ethernetif\_input()` holt empfangene Frames ab, `sys\_check\_timeouts()` erledigt LwIPs Zeitaufgaben (ohne Betriebssystem manuell nötig), `Ethernet\_Link\_Periodic\_Handle()` prüft den Link |

| `Ethernet\_Link\_Periodic\_Handle()` | Fragt alle 100 ms den PHY, ob ein Kabel steckt und mit welcher Geschwindigkeit und welchem Duplex-Modus verhandelt wurde |

| `ethernet\_link\_status\_updated()` | Callback bei Link-Änderung, hier leer. Platz z. B. für eine LED |

| `sio\_...`-Funktionen | Nur für den Keil-Compiler, werden bei GCC übersprungen |



Ein eingehender Ping wird von LwIP automatisch beantwortet (ICMP), ohne eigenen Code.



\## 6. Der PowerShell-Empfänger



\### 6.1 Das Skript



Als Datei `udp\_empfaenger.ps1` speichern oder direkt in ein PowerShell-Fenster kopieren:



```powershell

$udp = New-Object System.Net.Sockets.UdpClient(5005)

$ep  = New-Object System.Net.IPEndPoint(\[System.Net.IPAddress]::Any, 0)

Write-Host "Warte auf Daten auf Port 5005 ... (Strg+C zum Beenden)"

try {

&#x20;   while ($true) {

&#x20;       if ($udp.Available -gt 0) {

&#x20;           $bytes = $udp.Receive(\[ref]$ep)

&#x20;           $text  = \[System.Text.Encoding]::ASCII.GetString($bytes)

&#x20;           Write-Host "$($ep.Address):$($ep.Port)  $text"

&#x20;       } else {

&#x20;           Start-Sleep -Milliseconds 50

&#x20;       }

&#x20;   }

}

finally {

&#x20;   $udp.Close()

}

```



Falls das Ausführen einer `.ps1`-Datei von Windows blockiert wird (Execution Policy, Richtlinie für Skripte), das Skript einfach in das Fenster kopieren.



\### 6.2 Erklärung



| Zeile | Bedeutung |

|---|---|

| `$udp = New-Object System.Net.Sockets.UdpClient(5005)` | Erzeugt ein UDP-Objekt aus dem \*\*.NET Framework\*\* (Microsoft-Programmbibliothek) und \*\*bindet\*\* es an Port 5005: Alle UDP-Pakete für diesen Port gehören ab jetzt dem Skript. Ein \*\*Socket\*\* (englisch „Steckdose") ist der Zugangspunkt eines Programms zum Netzwerk. Der Port ist danach exklusiv belegt, ein zweiter Start des Skripts gibt eine Fehlermeldung |

| `$ep = New-Object System.Net.IPEndPoint(...Any, 0)` | Ein \*\*Endpoint\*\* ist eine Kombination aus IP-Adresse und Port. Das hier ist ein leerer Platzhalter für den Absender. `Any` (`0.0.0.0`) heißt: von jedem Absender annehmen |

| `try { ... } finally { $udp.Close() }` | Sicherung: Egal wie die Schleife endet, auch bei Strg+C, wird der Port am Ende wieder freigegeben |

| `while ($true)` | Endlosschleife, Abbruch nur mit Strg+C |

| `$udp.Available -gt 0` | Anzahl der angekommenen, aber noch nicht abgeholten Bytes. `-gt` heißt „greater than" (größer als) |

| `$udp.Receive(\[ref]$ep)` | Holt ein Paket ab. `\[ref]` übergibt den Platzhalter als Referenz, damit `Receive` hineinschreibt, von welcher IP und welchem Port das Paket kam. Ergebnis: ein Byte-Array (Zahlen von 0 bis 255) |

| `\[System.Text.Encoding]::ASCII.GetString($bytes)` | Wandelt die Zahlen in Text um. \*\*ASCII\*\* (American Standard Code for Information Interchange) ordnet Zahlen Buchstaben zu, z. B. 72 = „H". Für Umlaute reicht ASCII nicht |

| `Write-Host "$($ep.Address):$($ep.Port)  $text"` | Gibt Absender-IP, Absender-Port und Text aus. `$( ... )` setzt einen Wert in den Text ein |

| `Start-Sleep -Milliseconds 50` | Wartet 50 ms, wenn nichts da ist, damit der Prozessor nicht dauernd ausgelastet wird |



\*\*Warum die Abfrage mit `Available` statt direkt `Receive`?\*\* `Receive` blockiert, bis ein Paket kommt, und währenddessen lässt sich das Skript nicht mit Strg+C abbrechen. Mit der Abfrage bleibt es jederzeit beendbar.



\### 6.3 Windows-Firewall



Beim ersten Start fragt Windows, ob PowerShell auf Netzwerke zugreifen darf. \*\*Zulassen\*\*, für private \*\*und\*\* öffentliche Netzwerke, denn eine Direktverbindung ohne Router stuft Windows oft als „öffentlich" ein. Ohne Freigabe kommen die Pakete nie im Skript an. Falls keine Abfrage erscheint, in der Windows-Defender-Firewall eine eingehende Regel für \*\*UDP-Port 5005\*\* anlegen.



\## 7. Testen



\### 7.1 Ping



In der PowerShell:



```

ping 192.168.0.67

```



Antworten vom Board bedeuten, dass Kabel, IP-Einstellungen, Ethernet-Hardware und LwIP funktionieren.



> \*\*Falle: führende Nullen.\*\* Windows liest Zahlen mit führender Null als Oktalzahlen (Basis 8). `ping 192.168.000.010` pingt `192.168.0.8`, weil `010` oktal 8 ist, und `067` wäre 55. IP-Adressen immer ohne führende Nullen schreiben.



> \*\*Falle: „Zielhost nicht erreichbar".\*\* Kommt die Meldung von der \*\*eigenen\*\* Laptop-IP (`192.168.0.20`), hat der Laptop per ARP niemanden gefunden. Das Board antwortet dann nicht, das Paket zählt in der Statistik trotzdem als „Empfangen".



\### 7.2 UDP-Empfang



1\. PowerShell-Skript aus Abschnitt 6 starten.

2\. STM32 flashen und starten.

3\. Im Fenster sollte jede Sekunde eine Zeile erscheinen:



```

192.168.0.67:49152  Hallo vom STM32: 3

```



Das allererste Paket kann fehlen (ARP).



\## 8. Fehlersuche



| Symptom | Mögliche Ursache und Abhilfe |

|---|---|

| Keine Link-LED an der RJ45-Buchse | Kabel prüfen, ETH auf RMII, PHY-Treiber LAN8742 in LWIP Platform Settings, Takt prüfen |

| Ping ohne Antwort, LED leuchtet | Pingt man die richtige Adresse (`192.168.0.67`)? Läuft `MX\_LWIP\_Process()` in der Schleife? Laptop im selben Subnetz? Firewall blockiert ICMP? |

| Programm startet nicht richtig | Im Debugger pausieren. Steht es in `Error\_Handler()`, ist eine Initialisierung fehlgeschlagen. Der Aufrufstapel (Call Stack) zeigt welche |

| Ping geht, aber im Skript kommt nichts an | Firewall-Freigabe für PowerShell bzw. UDP-Port 5005 prüfen. Laptop-IP im Code (`192.168.0.20`) mit `ipconfig` vergleichen. Stimmt der Port auf beiden Seiten (5005)? |

| Fehler beim Start des Skripts, Port belegt | Ein anderes Fenster hält den Port noch. Fenster schließen und erneut starten |

| Compilerfehler `lwip/udp.h` nicht gefunden | In CubeMX LWIP\_UDP aktivieren und Code neu generieren |

| Eintrag in der ARP-Tabelle prüfen | `arp -a` zeigt, ob für `192.168.0.67` eine MAC-Adresse bekannt ist |

| Unklar, wo das Problem liegt | \*\*Wireshark\*\* auf dem Ethernet-Adapter mit Filter `udp.port == 5005`. Kommen die Pakete an, aber das Skript zeigt nichts, liegt es an der Firewall. Kommen sie nicht an, liegt es am Board oder am Code |

| `main.c` ist nach dem Generieren verändert | Eigener Code stand außerhalb der `USER CODE`-Markierungen. Unter \*Project Manager → Code Generator\* muss „Keep User Code when re-generating" aktiv sein |



\## 9. Mögliche Erweiterungen



\- Statt des Zählers echte Messwerte senden, z. B. mit `snprintf(msg, sizeof(msg), "T=%.1f", temperatur)`. Dafür muss in den Projekteinstellungen Float-Unterstützung für `printf` aktiviert sein.

\- Rückrichtung: Mit `udp\_recv()` Daten \*\*vom Laptop zum STM32\*\* empfangen, z. B. um die LED LD2 per Paket zu schalten.

\- Rückgabewert von `udp\_sendto()` (Typ `err\_t`) und von `udp\_new()` auswerten und Fehler melden.

\- Debug-Ausgaben über USART3 (virtueller COM-Port des ST-LINK).

\- Umstieg auf \*\*TCP\*\*, wenn zuverlässige Übertragung nötig ist.

