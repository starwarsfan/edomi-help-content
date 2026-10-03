# Edomi → OBS Migration — Verhaltensregeln für Claude Code

Dieses Dokument ist eine **Vorlage**, keine fertige Projekt-Instruktion. Es sammelt die
Verhaltensregeln, die sich projektübergreifend bei der Migration von Edomi-Logikseiten nach OBS
bewährt haben — unabhängig von einer konkreten Hausinstallation.

**So verwenden:** Im eigenen, privaten Migrations-Repo entweder diese Datei per Verweis einbinden
("lies zusätzlich `<Pfad-zu-edomi-help-content>/CLAUDE.md`") oder die relevanten Abschnitte direkt
in die eigene `CLAUDE.md` übernehmen und dort um die eigenen Eckdaten ergänzen (OBS-Instanz-URL,
Credential-Ort, eigene Namenskonventionen, Ablageort der eigenen Edomi-Screenshots usw.). Diese
Datei selbst bleibt bewusst frei von jeglichen Instanz-/Hausdetails.

---

## Grundhaltung: nichts raten, alles verifizieren

- Eine Edomi-Logikseite **nie** aus dem Blocknamen oder der Übersichtsansicht heraus
  interpretieren. Bei jeder Ausgangsbox das "Befehle"-Popup, bei jedem Wertauslöser das
  Initialwert/Fixwert-Feld explizit einsehen (per Screenshot) — der angezeigte Blockname/Label
  kann vom tatsächlich konfigurierten Inhalt abweichen.
- Verkabelung (welcher Pin → welcher Pin, z. B. A1 vs. A2 bei einer Sperre) am Pixel im
  Screenshot nachvollziehen, nicht aus der "naheliegenden" Lesart ableiten.
- Für jeden unbekannten LBS-Typ zuerst `edomi-lbs-hilfe/<ID>_<Name>.png` konsultieren (siehe
  README dieses Repos). Fehlt der Baustein dort, den Nutzer um das Hilfe-Popup-Screenshot bitten,
  statt die Semantik zu raten.

## Spezialbausteine (Kategorie `19...`)

Bausteine mit einer Typ-ID ab `19...` sind **keine Standard-Edomi-Bausteine**, sondern von Nutzern
selbst erstellte Spezialbausteine für eine ganz konkrete Einzelaufgabe (z. B. "Fertigmeldung mit
Aktorsteuerung", "Sensor Watchdog"). Begegnet einem beim Migrieren so ein Baustein, lohnt sich eine
bewusste Zwischenüberlegung — in beide Richtungen:

- **Es kann sinnvoll sein, dafür einen neuen, nativen OBS-Funktionsblock vorzuschlagen/zu bauen**,
  statt die Logik mühsam aus vielen generischen OBS-Primitiven (compare/math_formula/edge_detect/
  timer_cron...) nachzubilden — vor allem, wenn der Spezialbaustein ein wiederkehrendes,
  klar abgegrenztes Muster kapselt (Schwellwertüberwachung mit Nachlaufzeit, Staleness-Watchdog
  o. ä.), das vermutlich auch in anderen Logikseiten/Projekten wieder auftaucht. In diesem Fall
  dem Nutzer aktiv vorschlagen, einen entsprechenden Feature-Request/Issue im OBS-Projekt zu
  erfassen, statt stillschweigend nur die Einzellösung zu bauen.
- **Genauso oft ist das Gegenteil der Fall:** Die Idee hinter dem Spezialbaustein lässt sich in
  OBS mit dessen reichhaltigerem Node-Angebot (oder weil im OBS-Projekt bereits granularere
  Datenpunkte existieren als in Edomi) deutlich einfacher als generischer Mini-Graph abbilden —
  ganz ohne eigenen neuen Node-Typ. Nicht automatisch von "Spezialbaustein in Edomi" auf
  "braucht Spezial-Node in OBS" schliessen.

Welcher der beiden Fälle vorliegt, lässt sich erst nach dem vollständigen Verstehen der
Baustein-Semantik (siehe Hilfe-Popup) und einem Blick auf die in OBS tatsächlich verfügbaren
Node-Typen/Datenpunkte beurteilen — diese Abwägung dem Nutzer transparent machen, nicht
stillschweigend in die eine oder andere Richtung entscheiden.
- Bei Unklarheit über eine konkrete Edomi-Konfiguration (Text, Schwellwert, Verkabelung) lieber
  einmal mehr nachfragen, als eine falsche Annahme in eine produktive Logik zu bauen.

## Beim Bauen eines neuen OBS-Logikgraphen

- **Immer `enabled: false`** beim Erstellen über die API setzen. Der Nutzer reviewed, ordnet die
  Knoten in der OBS-UI an und aktiviert selbst.
- Vor dem Nachbauen von Edomis Rohsignal+Inverter-Verdrahtung prüfen, ob OBS (bzw. das
  darunterliegende KNX-/HA-Projekt) nicht bereits einen granulareren, direkt nutzbaren
  Datenpunkt bereitstellt — das vereinfacht den Graphen oft erheblich gegenüber einer 1:1-Portierung.
- Knotenpositionen nach einem **festen Raster** vergeben (z. B. 320×180 px pro Spalte/Zeile),
  nie nach Augenmass/ad-hoc-Pixel-Offsets — sonst überlappen sich Knoten in der UI, sobald sie
  mehr als eine Zeile Inhalt haben.
- Vor dem Anlegen prüfen: keine zwei Knoten auf derselben Rasterposition, kein Knoten ohne
  jede ausgehende Verbindung ("toter" Knoten — bricht nicht die Validierung, fällt aber beim
  manuellen Review negativ auf), keine zwei Kanten auf dasselbe Ziel-Handle (bricht in der OBS-UI
  mit "Duplicate connection... use a Merge node", wird aber von der Preflight-Prüfung nicht
  erkannt).
- `datapoint_read`/`datapoint_write`-Knoten immer mit **sowohl `datapoint_id` als auch
  `datapoint_name`** anlegen — fehlt `datapoint_name`, zeigt die OBS-UI das Objektfeld als "nicht
  ausgewählt" an, obwohl der Knoten technisch korrekt und funktionsfähig ist.
- Graphnamen **ohne Sonderzeichen wie Gedankenstrich "—"** wählen (normaler Bindestrich "-")
  — je nach OBS-Version kann das den Graph-Export brechen.

## OBS-Schema immer live abfragen, nie aus altem Wissen ableiten

- `GET /api/v1/logic/node-types` ist die alleinige Quelle der Wahrheit für Node-Schemas
  (Eingänge/Ausgänge/Config-Felder/Handle-Namen). Felder, Einheiten und Handle-Namen ändern sich
  zwischen OBS-Versionen — nie aus einem alten Graph-Export oder aus Dokumentation (auch nicht
  aus dieser Datei!) übernehmen, ohne gegen die aktuell laufende Instanz zu verifizieren.
- Node-Typen mit dynamisch konfigurierbarer Eingangszahl (z. B. "Strings verbinden") listen ihre
  tatsächlichen Port-Namen oft nicht im Schema. Per Wegwerf-Graph + `/run` gezielt austesten,
  nicht raten.

## Technische Eigenheiten der OBS Logic Engine

- **`timer_delay`/`timer_pulse` feuern nicht von selbst**, solange kein anderer Trigger im
  selben Graphen den Graphen "tickt". Für alles, was rein zeitgesteuert (nach Ablauf einer Zeit
  oder nach festem Zeitplan) auslösen soll, ist **`timer_cron`** der einzige verlässlich
  selbst-planende Node-Typ.
- **`compare` mit Operator "=" vergleicht numerisch koerziert**, auch bei zwei String-Eingängen
  (`"0123" == "123"` → `true`). Für echten Textvergleich (z. B. Werte mit führender Null) beide
  Seiten vorher mit einem nicht-numerischen Zeichen verketten, um String-Modus zu erzwingen.
- **Ein `memory`-Node-Feedback-Loop wird durch JEDEN anderen Trigger im selben Graphen
  verfälscht** — die Engine wertet bei jedem Tick alle Value-Domain-Knoten des gesamten Graphen
  neu aus, auch unbeteiligte Feedback-Loops. Für Akkumulatoren stattdessen einen externen
  Datenpunkt mit explizit getriggertem Schreiben verwenden (Trigger ausschliesslich von
  `.changed` des relevanten Quell-Events, nicht von einer Value-Domain-Bedingung).
- **STRING-Datenpunkte, die von einem physischen KNX-Gerät stammen, können rohe Einzelbytes
  sein, kein ASCII-Text** (z. B. Tastatur-Ziffern, Display-Seitenindizes). Vor jedem Vergleich
  mit menschenlesbaren Werten per Bus-/Datenpunkt-Historie verifizieren, wie der Wert
  tatsächlich ankommt — `data_type: STRING` allein garantiert keinen lesbaren Text.
- Bei sicherheits-/aktorkritischen Aktionen (Türschlösser, Tore, alles mit physischer Wirkung):
  **nur das echte Ereignis selbst** (ein `.changed`/Tastendruck-Trigger) darf entscheiden, OB
  geschrieben wird. Value-Domain-Logik (compare/and/etc.) darf nur entscheiden, WAS geschrieben
  wird — nie, ob.

## Testen

- Für alles Neue/Unsichere: einen **Wegwerf-Testgraphen** gegen harmlose Test-Datenpunkte bauen,
  über `/run` verifizieren, danach löschen — bevor der echte Graph angefasst wird.
- **Nie** `/run` gegen einen Graphen mit echten Seiteneffekten (Aktoren, Benachrichtigungen)
  aufrufen, auch nicht testhalber.
- Bei einem gemeldeten Live-Bug: wenn möglich Datenpunkt-/Bus-Historie konsultieren
  (`GET /api/v1/history/{datapoint_id}?from=...&to=...` — Achtung, die Parameter heissen
  `from`/`to`, nicht `start`/`end`) statt rein aus der statischen Graph-Struktur zu raten.
- Nach jedem schreibenden API-Aufruf (insbesondere bei Visu-Seiten) den Stand erneut abrufen und
  verifizieren, dass die Änderung tatsächlich angekommen ist — manche Endpunkte quittieren mit
  Erfolg, ohne dass die Änderung inhaltlich wirkt, wenn der Request-Body nicht exakt die
  erwartete Form hat.
