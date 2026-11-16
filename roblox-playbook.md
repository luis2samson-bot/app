# Playbook: Roblox-Spiele

Ziel: ein Roblox-Spiel, das im echten Spieltest funktioniert, auf dem Handy flüssig läuft und nicht ausgenutzt werden kann. Das allgemeine Playbook gilt weiter; hier steht nur, was Roblox besonders macht.

Wichtigster Grundsatz: Code, der nicht im Spieltest gelaufen ist, gilt als ungeprüft. Roblox-Fehler zeigen sich fast nie beim Lesen, sondern erst zur Laufzeit, oft nur mit zwei Spielern oder auf dem Server.

## Arbeitsumgebung
- Gearbeitet wird direkt im geöffneten Roblox Studio über dessen eingebaute Verbindung (MCP-Server). Die Studio-Datei ist die einzige Quelle für Skripte und Welt.
- Zu Beginn jeder Sitzung prüfen: Steht die Verbindung, welches Spiel ist geöffnet, und welche Werkzeuge bietet die Verbindung an. Nichts annehmen, was du nicht abgefragt hast.
- Vor jeder Änderung den aktuellen Zustand aus Studio lesen: das betroffene Skript vollständig und die Objekte, auf die es zugreift. Ich kann zwischendurch selbst etwas geändert haben.
- Ein Skript gezielt ändern, nicht komplett neu schreiben, außer ich verlange es. Vor dem Überschreiben eines bestehenden Skripts oder dem Löschen von Objekten einmal nachfragen.
- Es gibt keine Versionsgeschichte wie bei Git. Vor größeren Umbauten bitte mich, den Stand zu sichern (in Roblox veröffentlichen oder als Datei speichern), und warte auf meine Bestätigung.
- Ist die Verbindung nicht aktiv, nicht blind Code ausgeben. Sag es mir und nenne den Schritt: Assistant öffnen, "Manage MCP Servers", "Enable Studio as MCP server" einschalten.

## Projektaufbau
```
ServerScriptService/     Spielregeln, Daten, Käufe
ServerStorage/           Vorlagen und Module, die nur der Server sehen darf
ReplicatedStorage/
  Shared/                Konfiguration, Typen, Hilfsmodule
  Remotes/               alle RemoteEvents und RemoteFunctions an einem Ort
StarterPlayer/StarterPlayerScripts/   Eingabe, Kamera, Effekte
StarterGui/              Oberfläche
```
- Ein `Script` auf dem Server und ein `LocalScript` beim Spieler als Einstieg, alles andere sind `ModuleScript`s. So bleibt die Startreihenfolge bestimmbar.
- Zahlen zum Abstimmen (Preise, Schaden, Zeiten) stehen in Konfigurationsmodulen unter `Shared`, nicht verstreut im Code.
- Dienste immer mit `game:GetService("Name")` holen.
- Objekte, die Skripte suchen, bekommen eindeutige Namen oder `CollectionService`-Tags. Nichts über die Position in der Liste oder über Namen wie "Part" ansprechen.
- Neue Ordner, Remotes und Tags selbst anlegen, statt vorauszusetzen, dass sie da sind, und im Bericht nennen.

## Server und Spieler trennen
Der Server entscheidet, der Spieler zeigt an. Alles, was beim Spieler läuft, kann ein Schummler verändern.
- Über `RemoteEvent` und `RemoteFunction` schickt der Spieler nur Absichten ("ich will Gegenstand 3 kaufen"), nie Ergebnisse ("gib mir 500 Münzen").
- Jede Nachricht vom Spieler auf dem Server prüfen: Typ jedes Arguments, Wertebereich, ob der Spieler das gerade darf (Entfernung, Abklingzeit, Besitz, Guthaben), und wie oft er sendet.
- Der Server ruft nie `InvokeClient` auf, weil ein Spieler die Antwort beliebig verzögern oder verweigern kann.
- Geheimes und Serverlogik gehören nach `ServerScriptService` oder `ServerStorage`. Alles in `ReplicatedStorage` kann jeder Spieler lesen.
- Beim Spieler mit `WaitForChild` auf Objekte warten, die vom Server kommen. Mit eingeschaltetem Streaming können Teile der Welt fehlen; das gehört als Fall in den Code.

## Luau
- Jede Datei beginnt mit `--!strict`. Öffentliche Funktionen und Datenstrukturen bekommen Typen.
- `task.wait`, `task.spawn`, `task.delay` statt der veralteten `wait`, `spawn`, `delay`.
- Jede Verbindung (`:Connect`) hat einen Besitzer, der sie wieder trennt, etwa wenn ein Spieler geht oder eine Runde endet. Vergessene Verbindungen sind die häufigste Ursache für Speicherlecks.
- Keine Endlosschleifen, die pro Bild viele Objekte durchsuchen. Auf Ereignisse reagieren, und Objektgruppen über `CollectionService`-Tags führen.
- Veraltete Schnittstellen nicht aus dem Gedächtnis verwenden. Bei Unsicherheit in der Roblox-Dokumentation nachsehen.

## Spielerdaten
Datenverlust ist der schlimmste Fehler in einem Roblox-Spiel.
- Jeden `DataStoreService`-Aufruf in `pcall` setzen und Fehlschläge behandeln. Die Aufrufe scheitern im echten Betrieb regelmäßig.
- Schlägt das Laden fehl, bekommt der Spieler keine leeren Standarddaten, die später den echten Stand überschreiben. Stattdessen erneut versuchen oder den Spieler mit Hinweis abweisen.
- Speichern beim Verlassen, in `game:BindToClose` und in festen Abständen. `UpdateAsync` statt `SetAsync`, damit paralleles Schreiben nichts zerstört.
- Bevorzugt eine erprobte Bibliothek mit Sitzungssperre (ProfileStore) statt einer eigenen Lösung.
- Die Datenstruktur hat eine Versionsnummer und eine Umstellungsfunktion. Gespeicherte Felder nie umbenennen oder entfernen, ohne alte Stände umzuziehen.

## Käufe und Regeln der Plattform
- `ProcessReceipt` gibt den Kauf erst frei, wenn die Ware gutgeschrieben und gespeichert ist, und verträgt doppelte Aufrufe für denselben Kauf.
- Jeder Text, den ein Spieler eingibt und den andere sehen, läuft durch den Textfilter von Roblox. Ohne Filter droht die Sperrung des Spiels.
- Zufallskäufe, Altersfreigaben und Werbung unterliegen Roblox-Regeln, die sich ändern. Vor dem Einbau den aktuellen Stand nachlesen und mir nennen.

## Oberfläche und Geräte
- Die meisten Roblox-Spieler sind am Handy. Jede Funktion muss per Touch bedienbar sein; Tastatur und Gamepad kommen dazu.
- Oberflächen mit `Scale` statt fester Pixel, `UIAspectRatioConstraint` und sicherem Rand für Geräte mit Kerbe aufbauen.
- Eingaben über `ContextActionService` oder `UserInputService` binden, nicht an einzelne Tasten fest verdrahten.

## Leistung
- Unbewegte Teile verankern. Viele bewegliche Teile ohne Zweck kosten Physikleistung.
- Effekte (Partikel, Klänge, Animationen) laufen beim Spieler, der Server meldet nur das Ereignis.
- Streaming für große Welten einschalten. Vor der Abgabe in Studio Speicher und Skriptlast im Leistungsfenster ansehen.

## Prüfweg
Vor "fertig" in dieser Reihenfolge:
1. Skriptanalyse von Studio ohne Befund im eigenen Code.
2. Spieltest starten, Funktion wirklich auslösen, Ausgabe lesen. Keine Fehler und keine Warnungen aus eigenem Code. Danach den Spieltest beenden, bevor weiter geändert wird, weil Änderungen im laufenden Test verloren gehen.
3. Alles mit Mehrspielerbezug mit mindestens zwei Spielern im lokalen Servertest. Kann die Verbindung das nicht starten, bitte mich darum und sag, worauf ich achten soll.
4. Alles mit Daten: beitreten, Stand ändern, verlassen, neu beitreten, Stand vergleichen. Dafür muss in den Spieleinstellungen der Studio-Zugriff auf die API-Dienste erlaubt sein.
5. Oberfläche in der Geräteansicht für ein kleines Handy.
6. Für jede neue Nachricht vom Spieler: einmal mit falschen Werten senden und zeigen, dass der Server ablehnt.

Was davon nicht lief, im Bericht als "nicht getestet" nennen, mit dem Grund.

## Ablauf für ein neues Spiel
1. Konzept in fünf Sätzen: Kernablauf, Rundenlänge, Spielerzahl, Fortschritt, was es nicht enthält.
2. Gerüst: Ordner und Einstiegsskripte in Studio anlegen, ein leerer Spieltest ohne Fehler.
3. Kernablauf mit grauen Blöcken, spielbar von Anfang bis Ende einer Runde.
4. Daten und Fortschritt.
5. Oberfläche, dann Effekte, Klang und Feinschliff.
6. Käufe erst, wenn das Spiel ohne sie Spaß macht.
7. Vor der Veröffentlichung: kompletter Prüfweg, dann frage ich nach.

## Grenzen
- Veröffentlichen, Spieleinstellungen, Käufe mit echtem Geld und alles am Roblox-Konto nur nach meiner ausdrücklichen Freigabe.
- Keine fremden Modelle aus dem Marktplatz ungeprüft einfügen. Sie enthalten oft versteckte Skripte; vorher alle enthaltenen Skripte lesen.
- Keine Schlüssel oder Zugangsdaten in Skripte.
