# Konzept: Lichtschalter-App für das „RECORDING“-Leuchtschild

Status: **Konzept – noch kein Code.** Dieses Dokument sammelt alles, was wir
für die Umsetzung wissen müssen, und markiert klar, was noch offen ist.

---

## 1. Ziel

Eine Android-App, mit der sich das „RECORDING“-Leuchtschild per Bluetooth
**ein- und ausschalten** lässt – schnell, ohne die Hersteller-App, idealerweise
mit einem Tipp (z. B. über eine Schnelleinstellungs-Kachel).

Typischer Einsatz: Vor einer Aufnahme Schild an, danach aus.

## 2. Hardware (Stand: Foto vom Gerät)

| Bauteil | Was wir wissen | Offen |
|---|---|---|
| Leuchtschild | Schwarzer Rahmen, rote Schrift „RECORDING“, LED-Beleuchtung über dünnes Kabel | Einfarbig rot oder RGB? |
| Controller | Durchsichtiger USB-Stick-Controller, Beschriftung **„HCLEO“**, „INPUT DC5V“, „OUTPUT“ | Hersteller, Chip, Protokoll |
| Stromversorgung | USB-A, 5 V (Netzteil oder Computer) | – |
| QR-Code | Auf dem Controller aufgedruckt | Führt vermutlich zur Hersteller-App → **scannen und App-Namen notieren** |

**Annahme:** Der Controller spricht **Bluetooth Low Energy (BLE)**. Das ist bei
solchen USB-LED-Controllern üblich, muss aber bestätigt werden (siehe 5.).

## 3. Funktionsumfang

### Version 1 (MVP)
- Lampe in der Nähe finden und einmalig auswählen
- Lampe merken und beim nächsten Start automatisch verbinden
- Großer Schalter **AN / AUS**
- Verständliche Meldungen: Bluetooth aus, Berechtigung fehlt, Lampe nicht erreichbar

### Später (Ideen, nach Priorität)
1. **Schnelleinstellungs-Kachel** (Quick Settings Tile) – Schalten ohne App zu öffnen
2. **Homescreen-Widget**
3. Helligkeit (falls der Controller das kann)
4. Farbe/Effekte (nur falls RGB)
5. Automatik, z. B. Schild an, sobald eine Aufnahme-App läuft

## 4. Technik

| Thema | Entscheidung |
|---|---|
| Sprache / UI | Kotlin, Jetpack Compose, Material 3 |
| Mindestversion | Android 8.0 (API 26) |
| Zielversion | aktuelle Android-Version (API 36) |
| Bluetooth | Android BLE-API (`BluetoothLeScanner`, `BluetoothGatt`) – keine Fremdbibliothek |
| Speicher | `SharedPreferences` für die gemerkte Lampe (MAC-Adresse + Name) |
| Build | Gradle (Kotlin DSL) mit Version Catalog, GitHub Actions baut die APK |

### Berechtigungen

| Android-Version | Berechtigungen |
|---|---|
| 12 und neuer (API 31+) | `BLUETOOTH_SCAN` (mit `neverForLocation`), `BLUETOOTH_CONNECT` |
| 8 bis 11 (API 26–30) | `BLUETOOTH`, `BLUETOOTH_ADMIN`, `ACCESS_FINE_LOCATION` – zusätzlich muss der **Standortdienst eingeschaltet** sein, sonst findet die Suche nichts |

### Aufbau

```
UI (Compose)            Schalter, Geräteliste, Hinweise
   │
LampViewModel           Zustand, gemerkte Lampe, Auto-Verbinden
   │
BleLampClient           Suchen, Verbinden, Befehle schreiben
   │
LampConfig              UUIDs + Ein/Aus-Bytes  ← einzige Stelle, die vom Protokoll abhängt
```

Alles Gerätespezifische steckt in `LampConfig`. Sobald das Protokoll bekannt
ist (Abschnitt 5), ändern wir nur diese Datei.

### Ablauf in der App

```
Start → Berechtigung? ──nein──► Berechtigung anfragen
          │ja
        Bluetooth an? ──nein──► Einschalten anbieten
          │ja
        Lampe gemerkt? ──ja──► Verbinden ──► Schalter
          │nein                   │Fehler
        Suchen ► Liste ► Auswahl ─┘──► Meldung + Liste
```

## 5. Offene Punkte – das müssen wir herausfinden

Das Wichtigste ist das **Funkprotokoll**: welche Bytes an welche Stelle
geschickt werden, damit die Lampe an- bzw. ausgeht.

### Schritt 1: Hersteller-App ermitteln
- QR-Code auf dem Controller scannen
- App-Name und Link notieren → Tabelle unten

### Schritt 2: Gerät mit nRF Connect untersuchen
Kostenlose App **nRF Connect for Mobile** (Nordic Semiconductor) installieren.
Wichtig: Die Hersteller-App vorher schließen, viele Controller erlauben nur
**eine** Verbindung gleichzeitig.

1. Im Reiter *Scanner* nach Geräten suchen, Schild ein- und ausstecken, um das richtige Gerät zu erkennen
2. **Gerätename** und **MAC-Adresse** notieren
3. *Connect* → alle **Services** und **Characteristics** notieren, vor allem die mit Eigenschaft **WRITE** oder **WRITE NO RESPONSE**
4. Taucht das Gerät gar nicht auf, ist es vermutlich **klassisches Bluetooth** statt BLE → Konzept muss angepasst werden

### Schritt 3: Ein/Aus-Befehle mitschneiden
1. Android: *Entwickleroptionen* → **„Bluetooth-HCI-Snoop-Protokoll aktivieren“**
2. Bluetooth aus- und wieder einschalten
3. In der Hersteller-App die Lampe **mehrmals ein und aus** schalten, langsam, mit Pausen
4. Fehlerbericht erstellen (`adb bugreport`), darin liegt `btsnoop_hci.log`
5. In **Wireshark** öffnen, Filter `btatt.opcode == 0x52 || btatt.opcode == 0x12` (Write-Befehle)
6. Für „Ein“ und „Aus“ jeweils **Handle/UUID** und **Bytes** notieren

### Ergebnis-Tabelle (bitte ausfüllen)

| Frage | Antwort |
|---|---|
| Hersteller-App (aus QR-Code) | |
| Bluetooth-Art (BLE / klassisch) | |
| Gerätename beim Scannen | |
| MAC-Adresse | |
| Service-UUID | |
| Characteristic-UUID (Write) | |
| Write-Typ (mit / ohne Antwort) | |
| Bytes für **Ein** | |
| Bytes für **Aus** | |
| Kann man den Zustand auslesen? | |
| Pairing / PIN nötig? | |
| Helligkeit / Farbe möglich? | |

### Hypothese zum Vergleich (unbestätigt)
Viele günstige USB-LED-Controller nutzen das „ELK-BLEDOM“-Protokoll. Falls die
Werte aus Schritt 2/3 so aussehen, ist das ein Treffer:

| | Wert |
|---|---|
| Gerätename | beginnt oft mit `ELK-BLEDOM` oder `BLEDOM` |
| Service | `0000fff0-0000-1000-8000-00805f9b34fb` |
| Characteristic | `0000fff3-0000-1000-8000-00805f9b34fb` |
| Ein | `7e 00 04 f0 00 01 ff 00 ef` |
| Aus | `7e 00 04 00 00 00 ff 00 ef` |

Ob der „HCLEO“-Controller dazugehört, wissen wir **nicht** – erst messen.

## 6. Risiken

| Risiko | Folge | Umgang |
|---|---|---|
| Nur eine Verbindung gleichzeitig | Unsere App kann nicht verbinden, solange die Hersteller-App läuft | Hinweis in der App |
| Klassisches Bluetooth statt BLE | Andere API (RFCOMM/SPP) nötig | Nach Schritt 2 entscheiden |
| Zustand nicht auslesbar | App weiß nach dem Start nicht, ob die Lampe an ist | Anzeige „?“ plus getrennte Ein-/Aus-Tasten |
| Verschlüsseltes oder proprietäres Protokoll | Mitschnitt allein reicht nicht | Hersteller-App genauer analysieren oder Controller tauschen |
| Lampe außer Reichweite / ohne Strom | Verbindung schlägt fehl | Klare Fehlermeldung, erneut versuchen |

## 7. Nächste Schritte

1. [ ] QR-Code scannen, Hersteller-App notieren
2. [ ] nRF-Connect-Untersuchung (Schritt 2), Ergebnisse in Tabelle eintragen
3. [ ] Ein/Aus-Bytes mitschneiden (Schritt 3)
4. [ ] Protokoll in `LampConfig` übernehmen
5. [ ] MVP umsetzen und auf dem Handy testen
6. [ ] Schnelleinstellungs-Kachel

## 8. Quellen & UI-Inspiration

| Quelle | Wofür |
|---|---|
| [awesome-android-ui](https://github.com/wasabeef/awesome-android-ui) | Sammlung von Android-UI-Bibliotheken, als Ideenquelle für Schalter, Buttons und Animationen |

Hinweis: Die meisten Einträge dort sind ältere View-Bibliotheken (XML-Layouts),
nicht Jetpack Compose. Wir nutzen die Liste deshalb **nur als Inspiration** und
bauen das Gewünschte mit Compose/Material 3 selbst nach. Direkt für Compose
interessant ist der Abschnitt *Jetpack Compose*, z. B.
[ComposeCookBook](https://github.com/Gurupreet/ComposeCookBook) und
[neumorphic-compose](https://github.com/CuriousNikhil/neumorphic-compose)
(ggf. für einen auffälligen AN/AUS-Schalter).
