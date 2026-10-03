# Edomi → OBS Migrations-Hilfe

Dieses Repository sammelt Material, das anderen **Edomi**-Nutzern hilft, ihre visuellen
Logikseiten nach **Open Bridge Server (OBS)** zu migrieren — unter Einsatz eines
KI-Coding-Assistenten (Claude) als aktivem Migrationspartner.

Hier liegt bewusst **nichts Haus-/Installationsspezifisches** (keine eigenen Gruppenadressen,
Datenpunkt-Namen, Logikgraphen o.ä.) — das gehört in ein privates, separates Repo pro Nutzer.
Dieses Repo enthält nur, was für *jede* Edomi→OBS-Migration wiederverwendbar ist: generisches
Referenzmaterial zu den Edomi-Bausteinen (LBS) sowie die Methodik/das Setup, mit dem sich eine
solche Migration zusammen mit einem KI-Assistenten effizient durchführen lässt.

---

## Warum dieser Ansatz?

- **Edomi hat keine API.** Der einzige Weg, eine Logikseite maschinenlesbar zu erfassen, sind
  Screenshots der visuellen Flow-Ansicht.
- **OBS hat eine REST-API** für seine Logic Engine (Graphen lesen/schreiben, Node-Typen abfragen,
  Datenpunkte nachschlagen usw.).
- Ein KI-Assistent mit Zugriff auf ein Terminal/Dateisystem kann also: Edomi-Screenshots lesen und
  interpretieren → die Logik verstehen → einen äquivalenten OBS-Graphen über die REST-API bauen —
  und das pro Logikseite in wenigen Minuten statt Stunden von Hand.

Diese Methode wurde über mehr als zehn vollständige Logikseiten-Migrationen hinweg entwickelt und
verfeinert. Die hier gesammelten Erkenntnisse sollen anderen die Lernkurve verkürzen.

---

## Repo-Struktur

```
edomi-help-content/
├── README.md                  ← diese Datei
├── CLAUDE.md                  ← Vorlage: generische Verhaltensregeln für Claude Code
└── edomi-lbs-hilfe/           ← Referenz-Screenshots der Edomi-eigenen LBS-Hilfe
    ├── 00000000_Uebersicht.png
    ├── 12000002_Eingangsbox.png
    ├── 12000011_Ausgangsbox_ungleich0.png
    ├── 12000012_Ausgangsbox_ungleich_leer.png
    ├── 12000015_Ausgangsbox_gleich0.png
    ├── 13000012_Klemme.png
    ├── 13000031_Inverter.png
    ├── 14000029_Sperre.png
    ├── 16000101_Timer.png
    ├── 13000022_Wertauslöser.png
    └── ...
```

Der Ordner `edomi-lbs-hilfe/` enthält Screenshots der **Edomi-eigenen Hilfe-Popups** (das
"?"-Symbol an jedem Logikbaustein) für die gängigen LBS-Typen. Diese Screenshots sind die
präziseste verfügbare Beschreibung, wie ein Baustein tatsächlich funktioniert — deutlich
zuverlässiger, als sich auf den Namen oder das Aussehen in der Flow-Ansicht zu verlassen.

`00000000_Uebersicht.png` zeigt Edomis eigene Kategorisierung der Logikbausteine (Baustein-Browser
in der Logikseiten-Ansicht) und damit, wie sich die LBS-Typ-IDs in Kategorien gliedern:

| Kategorie | ID-Präfix |
|---|---|
| Grundfunktionen (Ein-/Ausgangsboxen, KO-Initialisierung) | `12...` |
| Grundfunktionen & Auslöser | `13...` |
| Gatter & Logik | `14...` |
| Mathematik, Vergleicher & Filter | `15...` |
| Timer & Zeitfunktionen | `16...` |
| Experimentell | `17...` |
| Sonstige | `18...` |
| Eigene Logikbausteine | `19...` |

**Wichtig zur Kategorie `19...`:** Das sind **keine Standard-Edomi-Bausteine**, sondern von
Usern selbst erstellte Spezialbausteine für ganz konkrete Einzelaufgaben (z. B. "Fertigmeldung
mit Aktorsteuerung", "Sensor Watchdog"). Begegnet einem beim Migrieren ein `19...`-Baustein, lohnt
sich ein bewusster Zwischenschritt — siehe dazu den Abschnitt zu Spezialbausteinen in
[`CLAUDE.md`](CLAUDE.md).

Die Dateien sind nach dem Schema **`<LBS-Typ-ID>_<Name>.png`** benannt — die LBS-Typ-ID ist die
interne, numerische Edomi-Kennung des Bausteins (z. B. `12000015` für "Ausgangsbox =0"), sichtbar
im Hilfe-Popup selbst sowie im Baustein-Browser aus der Übersicht. Das macht die Dateien eindeutig
zuordenbar, auch wenn mehrere Bausteine einen ähnlichen Anzeigenamen haben (z. B. die drei
Ausgangsbox-Varianten) oder der Anzeigename sich zwischen Edomi-Versionen ändert — und die
Kategorie lässt sich allein am Dateinamen ablesen.

**Mitmachen erwünscht:** Stösst du beim Migrieren auf einen LBS-Typ, der hier noch fehlt,
screenshotte dessen Hilfe-Popup und füge ihn nach demselben Schema hinzu
(`<LBS-Typ-ID>_<Name>.png`, siehe bestehende Beispiele) — die passende ID findest du im
Baustein-Browser, siehe `00000000_Uebersicht.png`.

---

## Setup

### Zwei-Repo-Prinzip

Für eine eigene Migration empfiehlt sich dieselbe Aufteilung wie hier:

1. **Dieses Repo (oder ein Fork davon)** — generisches, wiederverwendbares Material. Wird bei
   Bedarf einfach als zusätzliche Quelle/Referenz neben dem eigenen Projekt verwendet.
2. **Ein eigenes, privates Repo** für die konkrete Migration — dort hinein gehören:
   - eine Projekt-Instruktionsdatei (siehe unten) mit den eigenen Infrastruktur-Eckdaten
     (OBS-Instanz-URL, KNX-Projektstruktur, Namenskonventionen, bekannte Datenpunkt-IDs etc.)
   - die eigenen Edomi-Screenshots pro zu migrierender Logikseite
   - Backups der erzeugten OBS-Logikgraphen (als JSON, siehe unten)

### Werkzeuge

- **[Claude Code](https://claude.com/claude-code)** — der eigentliche KI-Assistent mit
  Datei-/Terminal-/Web-Zugriff. Läuft entweder:
  - als eigenständiges CLI-Tool in einem beliebigen Terminal (editor-unabhängig), oder
  - über das **"Claude Code"-Plugin für JetBrains-IDEs** (z. B. IntelliJ IDEA) aus dem
    JetBrains-Marketplace — bringt ein integriertes Claude-Terminal-Panel direkt in die IDE,
    inklusive Datei-Diff-Ansicht für vorgeschlagene Änderungen. Funktional identisch zur
    CLI-Variante, nur komfortabler integriert.
- Ein **Claude-Abo/API-Zugang**, der Claude Code abdeckt.
- Eine erreichbare **OBS-Instanz** mit aktivierter REST-API (`/api/v1`) und einem API-Zugang mit
  ausreichenden Rechten (zum Lesen *und* Schreiben von Logikgraphen/Datenpunkten — je nach
  OBS-Version per API-Key-Header oder per Bearer-Token aus `/api/v1/auth/login`).
- Das OBS-API-Credential **nicht** im Git-Repo ablegen — z. B. in einer lokalen,
  `.gitignore`-ten Datei (bzw. generell ausserhalb des Repos) speichern und den Assistenten
  anweisen, es von dort zu lesen.

### Projekt-Instruktionsdatei

Claude Code liest beim Start automatisch eine `CLAUDE.md` im Projekt-Root (falls vorhanden) als
Dauerkontext — im Unterschied zu diesem README, das nur gelesen wird, wenn man (oder der
Assistent) es explizit öffnet. Die generischen, projektübergreifenden Verhaltensregeln (z. B.
"neue Graphen immer mit `enabled: false` anlegen", Knotenraster, die Engine-Eigenheiten rund um
`timer_delay`/`compare`/`memory` usw.) liegen deshalb hier im Repo als Vorlage:
**[`CLAUDE.md`](CLAUDE.md)**.

Im eigenen, privaten Migrations-Repo diese Datei per Verweis einbinden oder die relevanten
Abschnitte übernehmen, und dort um die eigenen, stabilen Eckdaten ergänzen — z. B.:

- OBS-Instanz-URL, API-Version, wo/wie das Credential zu finden ist
- bekannte API-Eigenheiten der eigenen OBS-Version (ändert sich zwischen Releases!)
- Namenskonventionen für neue Logikgraphen/Datenpunkte
- Hinweis, wo Edomi-Screenshots pro Logikseite abgelegt werden (z. B.
  `edomi-migration/<Seitenname>/`)

Diese hausspezifischen Eckdaten gehören **nicht** in dieses generische Repo.

---

## Empfohlener Workflow pro Logikseite

1. **Screenshots anfertigen.** Die komplette Logikseite in der Edomi-Flow-Ansicht erfassen —
   bei Bedarf mehrere Screenshots (Übersicht + Zoom auf einzelne Bausteine), wenn Beschriftungen
   auf der Gesamtübersicht nicht lesbar sind. **Besonders wichtig:** bei jeder Ausgangsbox die
   "Befehle"-Popup-Ansicht separat screenshotten (der tatsächlich konfigurierte Text/Befehl kann
   vom angezeigten Label abweichen!), ebenso Initialwert/Fixwert-Felder bei Wertauslösern.
2. **Unbekannte Bausteine dokumentieren.** Für jeden LBS-Typ, der noch nicht in
   `edomi-lbs-hilfe/` vorhanden ist: dessen "?"-Hilfe-Popup screenshotten und ergänzen (siehe oben).
3. **Den Assistenten die Logik decodieren lassen** — mit der klaren Vorgabe, **nichts zu
   erraten**: jede Verkabelung, jeden konfigurierten Wert, jedes Pin-Paar (A1 vs. A2 o. ä.)
   wirklich am Pixel/Screenshot zu verifizieren, nicht aus dem Blocknamen zu schliessen.
4. **Bestehende OBS-Datenpunkte bevorzugen**, statt Edomis Rohsignal+Inverter-Verdrahtung 1:1
   nachzubauen. OBS-Projekte (gerade mit KNX-Anbindung) haben oft schon granularere,
   fertig aufbereitete Status-Objekte, die die Logik erheblich vereinfachen.
5. **Graphen deaktiviert anlegen lassen.** Der Assistent sollte neue Logikgraphen über die API
   zunächst mit `enabled: false` erstellen, damit man sie in der OBS-UI in Ruhe prüfen/anordnen
   kann, bevor sie live gehen.
6. **In der OBS-UI reviewen, anordnen, aktivieren, testen.**
7. **Bei sicherheits-/aktorkritischen Logiken** (Türschlösser, Tore, alles mit physischer
   Wirkung) zusätzliche Vorsicht: Knotenverhalten vorher an Wegwerf-Testgraphen gegen harmlose
   Test-Datenpunkte verifizieren, niemals den echten Graphen testhalber "scharf" laufen lassen.

---

## Allgemeine OBS-Logic-Engine-Stolperfallen

Diese Punkte sind unabhängig vom eigenen Setup und haben in der Praxis wiederholt Zeit gekostet —
lohnt sich, sie vorher zu kennen:

- **`timer_delay`/`timer_pulse` feuern nicht von selbst**, wenn im selben Graphen kein anderer
  Trigger den Graphen "tickt". Für alles, was zeitgesteuert ohne weiteren Reiz auslösen soll
  (z. B. "nach N Sekunden Stille"), ist `timer_cron` der einzige verlässlich selbst-planende
  Node-Typ.
- **`compare` mit Operator "=" vergleicht numerisch koerziert**, auch bei String-Eingaben
  (`"0123" == "123"` → `true`). Bei echtem String-Vergleich (z. B. PINs mit führender Null) vorher
  beide Seiten mit einem nicht-numerischen Zeichen verketten, um echten Textvergleich zu
  erzwingen.
- **Ein `memory`-Node-Feedback-Loop wird durch JEDEN anderen Trigger im selben Graphen
  verfälscht** (die Engine wertet bei jedem Tick den gesamten Graphen neu aus). Für Akkumulatoren
  stattdessen einen externen Datenpunkt plus explizit getriggertes Schreiben verwenden.
- **STRING-Datenpunkte von physischen KNX-Geräten können rohe Einzelbytes sein, kein ASCII-Text**
  (z. B. Tastatur-Ziffern, Seitenindizes von Displays) — vor jedem Vergleich mit
  menschenlesbaren Werten per Bus-Historie verifizieren, wie der Wert tatsächlich reinkommt.
- **Node-Typen mit dynamisch konfigurierbarer Eingangszahl** (z. B. "Strings verbinden") zeigen
  ihre tatsächlichen Port-Namen oft nicht im generischen Node-Typ-Schema — per Wegwerf-Graph und
  `/run` gezielt austesten, statt zu raten.
- **Immer `GET /api/v1/logic/node-types` als Quelle der Wahrheit** für Node-Schemas nutzen, nicht
  aus alten Graph-Exporten oder Dokumentation ableiten — Felder/Einheiten ändern sich zwischen
  OBS-Versionen.
- Vor dem "Scharfschalten" eines neuen Graphen: auf doppelt verkabelte Eingänge (mehrere Edges auf
  dasselbe Ziel-Handle — bricht in der OBS-UI mit "Duplicate connection") und auf Knoten ohne
  jede ausgehende Verbindung ("tote" Knoten) prüfen.

---

## Lizenz / Nutzung

Dieses Material darf frei von anderen Edomi-Nutzern für ihre eigene Migration verwendet und
erweitert werden.
