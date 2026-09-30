---
description: Roadmap für Zen-Garten, Crazy Daves Shop, Economy und HD-Assets
---

# Zen-Garten & Shop Roadmap

Diese Roadmap beschreibt die schrittweise Erweiterung des Browsergames um die fehlenden Originalmechaniken rund um Crazy Daves Shop, den Zen-Garten und die dazugehörige persistente Progression.

## Referenzen

Mechanik und Verhalten werden gegen zwei C++-Referenzen abgeglichen:

- `wszqkzqk/PvZ-Portable`
- `nasiftanjim/PvZ-QotL-Widescreen-NT`

Für Grafiken und visuelle Qualität dient zusätzlich `nasiftanjim/PVZHD` v2.1 als Referenz. Das vollständige Release-Assetpaket wird separat inventarisiert, sobald es lokal zur Verfügung steht.

Wichtig: C++-Quellcode und feste 800x600-Pixelkoordinaten werden nicht 1:1 übernommen. Die Browserimplementierung nutzt eigene JavaScript-Domänenmodelle und das adaptive Layout des Webgames.

## Leitlinien

- Mechanik vor Grafik.
- Persistente Daten werden versioniert und migrationsfähig gespeichert.
- Bestehende Spielstände und die bisherigen Legacy-Keys bleiben kompatibel.
- Shop, Zen-Garten und Spiel teilen dieselbe Economy.
- Keine Logik darf von einer bestimmten Bildschirmauflösung abhängen.
- Touch-/Maus-Hitboxes werden aus dem gerenderten Layout abgeleitet.
- HD-Assets ersetzen erst nach funktionierender Mechanik die Referenzgrafiken.
- Jede Phase wird mit automatisierten Tests und einem kleinen, reviewbaren PR abgeschlossen.

## Status

| Phase | Thema | Status |
| --- | --- | --- |
| 0 | Referenzanalyse & Asset-Inventar | In Arbeit |
| 1 | PlayerProgress & Save-System | PR [pvz-game#25](https://github.com/0b-ivan/pvz-game/pull/25) |
| 2 | Economy & Inventar | Geplant |
| 3 | Crazy Daves Shop | Geplant |
| 4 | Zen-Garten MVP | Geplant |
| 5 | Pilzgarten, Aquarium & Transport | Geplant |
| 6 | Stinky, Schokolade & Produktion | Geplant |
| 7 | Drops, Tree of Wisdom & Langzeitprogression | Geplant |
| 8 | PVZHD Asset-Pass | Blockiert bis Assetpaket vorliegt |
| 9 | Mobile/Widescreen QA & Balancing | Geplant |

## Phase 0 – Referenzanalyse & Asset-Inventar

### TODO

- [x] Zen-Garden-Implementierung in QotL und PvZ-Portable identifizieren.
- [x] Shop-Implementierung und Store-Items identifizieren.
- [x] PlayerInfo, Käufe, Coins und PottedPlant-Persistenz prüfen.
- [x] Vorhandene Zen-/Shop-Ressourcen im aktuellen Browsergame prüfen.
- [x] QotL-Assetgruppen für Store, Greenhouse, Mushroom Garden und Aquarium zuordnen.
- [ ] PVZHD-v2.1-Releasepaket vollständig inventarisieren.
- [ ] HD-Assets nach Shop, Garden, Pflanzen, UI, Reanimation und Hintergrund katalogisieren.
- [ ] Asset-Mapping `QotL asset -> PVZHD asset -> Browser asset` dokumentieren.

### Definition of Done

Die benötigten Mechaniken und ihre visuellen Abhängigkeiten sind eindeutig zugeordnet. Fehlende Assets sind bekannt und blockieren keine Logikimplementierung.

## Phase 1 – PlayerProgress & Save-System

Ziel ist eine zentrale, versionierte Persistenzschicht, auf der Shop und Zen-Garten aufbauen.

### Datenbereiche

- Adventure-Fortschritt
- Economy
- Store-Käufe
- Verbrauchsinventar
- Unlocks
- Zen-Garden-Pflanzen
- Wheelbarrow
- Stinky-Zustand
- Schema-Version und Revision

### TODO

- [x] `PlayerProgress.js` als zentrale Persistenzschicht hinzufügen.
- [x] Schema-Version 1 definieren.
- [x] Legacy-`level` und `levels` migrieren, ohne alte Keys zu löschen.
- [x] Persistenz gegen ungültiges JSON, unvollständige Daten und Rollbacks härten.
- [x] bestehende Adventure-Speicherung mit PlayerProgress spiegeln.
- [x] automatisierte Migration-/Persistenztests hinzufügen.
- [x] CI für PlayerProgress-Tests aktivieren.

### Definition of Done

Ein bestehender Browser-Spielstand wird automatisch in das neue Modell übernommen. Reloads verlieren keine Daten und bestehende Adventure-Funktionalität bleibt unverändert.

## Phase 2 – Economy & Inventar

### TODO

- [ ] zentrale Coin-Balance implementieren.
- [ ] Coin-Gutschrift und -Abzug atomar über PlayerProgress führen.
- [ ] permanente Käufe modellieren.
- [ ] Verbrauchsitems modellieren: Dünger, Bug Spray, Schokolade, Tree Food.
- [ ] Kauf-Limits und Bestandsgrenzen implementieren.
- [ ] Unlock-Regeln vom UI entkoppeln.
- [ ] Unit-Tests für Kosten, Limits und negative Bestände ergänzen.

### Definition of Done

Shop und spätere Garden-Mechaniken greifen auf dieselbe belastbare Economy-API zu.

## Phase 3 – Crazy Daves Shop

QotL definiert vier Shopseiten mit jeweils acht Slots.

### TODO

- [ ] Store-Domänenmodell und Item-Katalog anlegen.
- [ ] vier Shopseiten abbilden.
- [ ] Originalpreise und gestaffelte Seed-Slot-Kosten übernehmen.
- [ ] `locked`, `coming soon`, `sold out` und `owned` unterscheiden.
- [ ] Level-/Adventure-Freischaltungen implementieren.
- [ ] Kaufbestätigung und Nicht-genug-Geld-Dialog implementieren.
- [ ] Crazy-Dave-Dialogzustände anbinden.
- [ ] Tageslimit für Potted Marigolds implementieren.
- [ ] Shopzustand nach jedem Kauf persistieren.
- [ ] responsive Maus-/Touch-Hitboxes implementieren.

### Definition of Done

Der Shop ist vollständig bedienbar und alle Käufe überleben Reloads.

## Phase 4 – Zen-Garten MVP

Zuerst wird ausschließlich der Hauptgarten umgesetzt.

### TODO

- [ ] 8x4-Gartenraster modellieren.
- [ ] PottedPlant-Datenmodell implementieren.
- [ ] Sprout, Small, Medium und Full als Wachstumsstufen implementieren.
- [ ] zufällige 3-5 Bewässerungen pro Wachstumsstufe abbilden.
- [ ] Wasserbedarf und Bewässerung implementieren.
- [ ] Düngen und Wachstum implementieren.
- [ ] Full-Grown-Bedürfnisse implementieren: Water, Bug Spray, Phonograph.
- [ ] Bedürfnis-Overlays anzeigen.
- [ ] Belohnungen für Pflege und Wachstum erzeugen.
- [ ] Pflanzenverkauf und Verkaufspreise implementieren.
- [ ] Position, Zustand und Zeitstempel persistieren.
- [ ] Touchbedienung für Werkzeuge testen.

### Definition of Done

Pflanzen können gekauft/erhalten, platziert, gepflegt, hochgezogen, verkauft und nach Reload exakt wiederhergestellt werden.

## Phase 5 – Pilzgarten, Aquarium & Transport

### TODO

- [ ] Mushroom Garden freischaltbar machen.
- [ ] Aquarium freischaltbar machen.
- [ ] Pflanzenverträglichkeit pro Garten implementieren.
- [ ] Wheelbarrow als temporären Pflanzenplatz implementieren.
- [ ] Gartenwechsel implementieren.
- [ ] Garten-spezifische Raster und Hintergründe einbauen.
- [ ] nächtliche Pflanzenlogik im Hauptgarten berücksichtigen.
- [ ] aquatische Pflanzenlogik im Aquarium berücksichtigen.

### Definition of Done

Eine Pflanze kann regelkonform zwischen den unterstützten Gartenbereichen transportiert werden.

## Phase 6 – Stinky, Schokolade & Produktion

### TODO

- [ ] Stinky-Kauf und Persistenz implementieren.
- [ ] Schlaf-/Aufwachzustände implementieren.
- [ ] links/rechts laufen und Ziele wählen.
- [ ] automatisch Coins einsammeln.
- [ ] Position speichern.
- [ ] Schokolade für Pflanzen implementieren.
- [ ] Schokolade für Stinky implementieren.
- [ ] Happy-State und passive Münzproduktion implementieren.
- [ ] Produktionsrate nach Zeit und Schokolade abbilden.

### Definition of Done

Ein glücklicher Garten produziert Coins und Stinky kann sie selbstständig einsammeln.

## Phase 7 – Drops & Langzeitprogression

### TODO

- [ ] Potted-Plant-Drops aus regulären Levels anbinden.
- [ ] Schokoladen-Drops anbinden.
- [ ] Garden-Kapazität berücksichtigen.
- [ ] Tree of Wisdom implementieren.
- [ ] Tree Food und Wachstum implementieren.
- [ ] Unlock-Flow vom Adventure-Ende bis zum vollständigen Zen-System prüfen.
- [ ] fehlende Originalmechaniken erneut gegen die Referenzen auditieren.

### Definition of Done

Der Zen-Garten ist nicht nur ein separates Menü, sondern in die normale Progressionsschleife des Spiels integriert.

## Phase 8 – PVZHD Asset-Pass

Diese Phase beginnt nach Inventarisierung des v2.1-Releasepakets.

### TODO

- [ ] HD-Shophintergrund und iPad-Shoptexturen zuordnen.
- [ ] HD-Greenhouse, Mushroom Garden und Aquarium prüfen.
- [ ] Werkzeug- und Need-Icons zuordnen.
- [ ] Crazy Dave und Stinky prüfen.
- [ ] Pflanzen-/Pot-/Sprout-Assets prüfen.
- [ ] Reanimationsdaten auf Browserformate abbilden.
- [ ] notwendige Konvertierungen automatisieren.
- [ ] unnötige oder doppelte Low-Res-Assets entfernen.

### Definition of Done

Shop und Zen-Garten verwenden konsistente HD-Assets ohne mechanische Änderungen.

## Phase 9 – Mobile, Widescreen & QA

### TODO

- [ ] iPhone Portrait/Landscape prüfen.
- [ ] iPad prüfen.
- [ ] Desktop 16:9 und 16:10 prüfen.
- [ ] Maus-/Touch-Koordinaten testen.
- [ ] Werkzeug-Hitboxes testen.
- [ ] Gartenraster bei Resize testen.
- [ ] Save-Migration mit bestehenden Spielständen testen.
- [ ] Offline-/Service-Worker-Upgradepfad testen.
- [ ] Performance bei 32 Pflanzen und aktiver Coin-Produktion testen.

### Definition of Done

Die Erweiterung funktioniert auf den unterstützten Viewports ohne feste Pixelannahmen, Datenverlust oder Touch-Offset.

## Aktueller nächster Schritt

Phase 1 ist aktiv. Die erste Implementierung führt ein versioniertes `PlayerProgress`-Modell ein und migriert die bereits vorhandenen Adventure-Speicherdaten. Erst danach wird die Economy darauf aufgebaut.
