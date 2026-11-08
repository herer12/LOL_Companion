# Features – LOL Companion

Dieses Dokument beschreibt den geplanten Funktionsumfang des LOL Companion. Es ist eine Arbeitsgrundlage für das Team und wird laufend ergänzt. Technische Details (Stack, Architektur) kommen in eigene Dokumente.

## Ziel

Der LOL Companion ist eine Begleit-Software für League of Legends, die dich in jeder Phase eines Spiels unterstützt: Champion-Auswahl, Ladebildschirm und laufendes Spiel. Vom Prinzip her ähnelt sie Tools wie Blitz oder Porofessor, setzt aber eigene Schwerpunkte:

- **Alles ist abschaltbar.** Jedes Feature kann einzeln an- und ausgeschaltet werden. Die Auswahl wird gespeichert und gilt für alle folgenden Spiele.
- **Matchup-Fokus.** Builds, Runen und Tipps richten sich nach deinem konkreten Lane-Gegner und nicht nur nach deinem Champion.
- **Zwei Ansichten im Spiel.** Ein schlankes Overlay direkt im Spiel und eine ausführlichere Begleiter-App für den Zweitbildschirm.

## Komponenten

| Komponente | Beschreibung |
|---|---|
| **Hauptanwendung** | Desktop-Fenster für Einstellungen, Feature-Toggles, Profil und Statistiken. Läuft im Hintergrund und erkennt automatisch, in welcher Phase du bist (Client, Champ Select, Ladebildschirm, Spiel). |
| **In-Game-Overlay** | Wird über dem Spiel angezeigt. Zeigt nur wenige, frei wählbare Infos (Statistiken, Skill-Reihenfolge, Objective-Timer). Position, Größe und Transparenz sind einstellbar. |
| **Begleiter-App (Second Screen)** | Eigenes Fenster, z. B. für den Zweitbildschirm. Zeigt ausführlichere Infos und Hinweise während des Spiels. |

## Riot-Richtlinien: unbedingt beachten

Riot schreibt vor, was Drittanbieter-Apps für League of Legends dürfen. Wer sich nicht daran hält, bekommt keine Freigabe für die API, und Nutzer riskieren Strafen bis hin zur Kontosperre. Stand der Recherche (November 2026):

- **Verboten:** Das Tracken gegnerischer **Ultimate-Cooldowns** (steht ausdrücklich in Riots Developer Policy) sowie Timer für **gegnerische Summoner-Spells** und Fähigkeiten, auch wenn man sie von Hand startet (laut LoL-Compliance-Guide von Overwolf).
- **Verboten:** **Power-Spike-Alerts** (z. B. „Gegner ist Level 6“) und **Anweisungen zum Spielgeschehen** (z. B. „Jetzt gankt der Jungler, zieh dich zurück“).
- **Eingeschränkt:** In Ranked Solo/Duo werden die Namen im Champ Select anonymisiert. Fremde Mitspieler darf man dort **nicht** identifizieren. Sie werden erst im Ladebildschirm sichtbar.
- **Eingeschränkt:** Die API liefert für andere Spieler nur den Keystone und die Runen-Bäume, aber **keine kleinen Runen** (z. B. Kosmische Einsicht).

Deshalb hat jedes Feature unten einen Status:

- ✅ **Erlaubt.** Nach aktuellem Stand unproblematisch.
- ⚠️ **Prüfen.** Grauzone, vor der Umsetzung mit den aktuellen Richtlinien abgleichen bzw. bei Riot anfragen.
- ⛔ **Nicht erlaubt.** Bleibt beschrieben, damit die Idee festgehalten ist, wird aber **nicht umgesetzt**, solange sich die Richtlinien nicht ändern.

Quellen: [Riot Developer Portal – League of Legends](https://developer.riotgames.com/docs/lol), [Overwolf – LoL Game Compliance](https://dev.overwolf.com/ow-electron/guides/game-compliance/league-of-legends/). Vor jedem Release erneut prüfen, die Regeln ändern sich regelmäßig.

## Priorisierung

| Priorität | Bedeutung |
|---|---|
| **P1** | Kern des Produkts, gehört in die erste Version (MVP). |
| **P2** | Wichtig, kommt direkt nach dem MVP. |
| **P3** | Später bzw. nice to have. |

## Feature-Übersicht

| # | Feature | Phase | Ort | Prio | Status |
|---|---|---|---|---|---|
| F0 | Feature-Toggles & Einstellungen | Immer | Hauptanwendung | P1 | ✅ |
| F1 | Build & Runen gegen das Matchup | Champ Select | Hauptanwendung | P1 | ✅ |
| F2 | Draft-Helfer (Champion-Empfehlung) | Champ Select | Hauptanwendung | P2 | ✅ |
| F3 | Matchup-Tipps | Ladebildschirm | Hauptanwendung / Begleiter-App | P1 | ✅ |
| F4 | Matchup-Winrate & Schwierigkeit | Ladebildschirm | Hauptanwendung / Begleiter-App | P1 | ✅ |
| F5 | Statistiken der Mitspieler (und Gegner) | Ladebildschirm | Hauptanwendung / Begleiter-App | P2 | ✅ |
| F6 | Statistik-Overlay (CS/min, Gold/min, KDA, KP …) | Spiel | Overlay | P1 | ✅ |
| F7 | Skill-Level-up-Anzeige | Spiel | Overlay | P1 | ✅ |
| F8 | Objective-Timer & Reminder | Spiel | Overlay + Begleiter-App | P1 | ⚠️ |
| F9 | Hinweise in der Begleiter-App | Spiel | Begleiter-App | P2 | ⚠️ |
| F10 | Jungler „zuletzt gesehen“ | Spiel | Begleiter-App | P3 | ⚠️ |
| F11 | Summoner-Spell-Tracker der Gegner | Spiel | Begleiter-App | – | ⛔ |
| F12 | Ultimate-Tracker der Gegner | Spiel | Begleiter-App | – | ⛔ |

---

## F0 – Feature-Toggles & Einstellungen

**Ziel:** Vor dem Spiel legst du per Häkchen fest, welche Features aktiv sind. Die Auswahl wird gespeichert und muss nicht vor jedem Spiel neu getroffen werden.

- Liste aller Features mit Checkbox (an/aus), gruppiert nach Phase: Champ Select, Ladebildschirm, Overlay, Begleiter-App.
- Einstellungen werden lokal gespeichert und beim Start automatisch geladen.
- Sinnvolle Voreinstellungen beim ersten Start. Ein „Zurücksetzen“-Button stellt sie wieder her.
- Ist ein Feature aus, läuft es auch im Hintergrund nicht (keine Datenabfragen, kein Overlay-Element).
- Unteroptionen pro Feature, zum Beispiel:
  - Overlay: welche Statistiken angezeigt werden, Position, Größe, Transparenz.
  - Objective-Timer: welche Objectives, Zeitpunkte der Reminder (z. B. 60 s und 20 s), Ton an/aus.
- Hotkey, um das Overlay ein- und auszublenden.
- Änderungen gelten sofort, ein Neustart ist nicht nötig. Während eines laufenden Spiels kann man sie ebenfalls vornehmen.
- *(P3)* Mehrere Profile, z. B. „Ranked“ (wenig Overlay) und „Normal/ARAM“ (alles an).

## F1 – Build & Runen gegen das Matchup *(wichtigstes Draft-Feature)*

**Ziel:** Sobald dein Lane-Gegner feststeht, zeigt dir die App, was du **gegen genau diesen Champion** bauen und spielen solltest.

- **Runen:** Primär- und Sekundärbaum, alle kleinen Runen, Shards.
- **Summoner-Spells:** Empfehlung für das Matchup (z. B. Ignite statt TP).
- **Items:** Start-Items, erster Back, Kern-Build, Stiefel, situative Items (z. B. gegen viel AP oder viel Heilung).
- **Skill-Reihenfolge:** Erster Skill und Maxing-Reihenfolge (wird in F7 im Spiel weiterverwendet).
- Daten pro Kombination aus Champion, Gegner und Lane. Gibt es zu wenig Daten für das Matchup, fällt die App auf den allgemeinen Build des Champions zurück und weist darauf hin.
- Optional berücksichtigt die Empfehlung auch die gesamte Gegner-Komposition (z. B. Rüstung gegen drei AD-Champions).
- **Ein-Klick-Import:** Runenseite und Item-Set werden direkt in den League-Client übernommen.
- Anzeige von Winrate und Stichprobengröße zu jeder Empfehlung, damit man die Daten einschätzen kann.

## F2 – Draft-Helfer (Champion-Empfehlung)

**Ziel:** Hilfe bei der Frage, welchen Champion du in diesem Draft wählen solltest.

- **Lane-Matchup:** Gute Picks gegen den gegnerischen Lane-Champion (sobald bekannt).
- **Ganzes Spiel:** Bewertung mit Blick auf alle Gegner **und** deine Teammates, etwa:
  - Schadensverteilung im Team (AD/AP),
  - Frontline, Engage, Peel, Waveclear,
  - Synergien mit den Picks der Teammates,
  - Counter gegen die gegnerischen Champions.
- Empfehlungen werden **auf deinen Champion-Pool gefiltert** (Mastery bzw. zuletzt gespielte Champions). Optional zeigt die App auch Champions außerhalb des Pools.
- Kurze Begründung pro Vorschlag (z. B. „Euer Team hat kaum Magieschaden“).
- *(P3)* Bann-Vorschläge.
- **Richtlinie:** In Ranked Solo/Duo sind fremde Namen im Champ Select anonym. Der Draft-Helfer arbeitet daher nur mit Champions und Rollen, nicht mit Spielerdaten.

## F3 – Matchup-Tipps im Ladebildschirm

**Ziel:** Während des Ladebildschirms erfährst du, worauf du in deinem Matchup achten musst.

- 3–5 kurze, konkrete Tipps pro Matchup, zum Beispiel:
  > **Lux vs. Naafiri:** Halte deine Q zurück, solange Naafiri nicht reingeht. Nutzt Naafiri ihre W, setze deine Q nicht sofort ein, sondern erst danach, wenn sie getroffen werden kann.
- Gliederung nach Phasen: Early Game / Level 1–3, Power Spikes, Trading, All-in, was man vermeiden sollte.
- Hinweise zu wichtigen gegnerischen Fähigkeiten (welche Fähigkeit man ausweichen bzw. abwarten sollte).
- Gibt es für das genaue Matchup keine Tipps, zeigt die App allgemeine Tipps gegen den gegnerischen Champion.
- Die Tipps bleiben in der Begleiter-App während des Spiels abrufbar.
- **Offen:** Woher kommen die Inhalte? Selbst geschrieben, von der Community, aus Daten generiert oder eine Mischung. Siehe „Offene Fragen“.

## F4 – Matchup-Winrate & Schwierigkeit

- Winrate deines Champions gegen den Lane-Gegner (aktueller Patch, Elo-Bereich einstellbar).
- Schwierigkeit als einfache Skala, z. B. „Leicht / Mittel / Schwer“ oder 1–5, abgeleitet aus Winrate und Lane-Daten (Gold-/CS-Differenz bei 15 Minuten).
- Optional: deine persönliche Bilanz in diesem Matchup, falls du es schon gespielt hast.

## F5 – Statistiken der Mitspieler (und Gegner)

**Ziel:** Im Ladebildschirm siehst du, wie deine Teammates gerade spielen.

- Pro Spieler: Rang, Winrate der Saison, Form der letzten Spiele (z. B. letzte 10–20), Sieg-/Niederlagenserie.
- Erfahrung mit dem gewählten Champion: Anzahl der Spiele, Winrate, Mastery. Kennzeichnung „Erstes Mal auf dem Champion“.
- Hauptrolle und Hinweis, wenn jemand vermutlich „autofilled“ ist.
- Dieselbe Ansicht optional auch für die Gegner.
- **Richtlinie:** Erst ab dem Ladebildschirm, nicht im Champ Select (siehe Riot-Richtlinien). Neutrale Darstellung ohne abwertende Labels.

## F6 – Statistik-Overlay im Spiel

**Ziel:** Kleines Overlay im Spiel mit Statistiken, die du selbst auswählst.

Auswählbare Werte (Beispiele):

| Wert | Hinweis |
|---|---|
| CS gesamt und CS/min | |
| Gold/min | Die Live-API liefert nur das aktuelle Gold, das gesamte verdiente Gold muss geschätzt werden (Item-Werte + aktuelles Gold). |
| KDA / K/D/A | |
| Kill-Beteiligung (KP) | (Kills + Assists) / Kills des Teams |
| Vision Score | |
| Spielzeit | |
| *(P2)* Vergleich mit Zielwert | z. B. „CS/min: 6,8 (Ziel 8,0)“. Zielwert einstellbar. |

- Nur Werte des eigenen Spielers bzw. Werte, die auch über das Scoreboard (Tab) im Spiel sichtbar sind.
- Position, Größe und Transparenz frei einstellbar. Das Overlay soll keine wichtigen Spielelemente verdecken.

## F7 – Skill-Level-up-Anzeige

- Beim Level-up zeigt das Overlay kurz an, welchen Skill du jetzt leveln solltest (Q/W/E/R).
- Grundlage ist die Skill-Reihenfolge aus F1 für das aktuelle Matchup.
- Hast du einen Skill anders gelevelt als empfohlen, passt sich die Anzeige an.
- Optional: kompakte Leiste mit der kompletten Reihenfolge (Level 1–18).

## F8 – Objective-Timer & Reminder

**Ziel:** Du wirst rechtzeitig erinnert, wenn ein Objective spawnt.

- Timer für Drachen, Baron, Herald/Grubs und weitere Objectives des aktuellen Patches.
- Respawn-Zeiten werden aus Spielzeit und Kill-Events der Live-API berechnet (z. B. Drache getötet → nächster Drache in X Minuten).
- Reminder auf dem Overlay zu einstellbaren Zeitpunkten, Standard **60 s** und **20 s** vor dem Spawn, optional mit Ton.
- Die Begleiter-App zeigt alle laufenden Timer als Liste.
- Spawn- und Respawn-Zeiten ändern sich mit den Patches. Sie werden deshalb als Daten gepflegt und nicht im Code fest eingetragen.
- **Richtlinie ⚠️:** Riot hat inzwischen eigene Jungle-Timer ins Spiel eingebaut. Prüfen, ob Objective-Timer von Drittanbietern noch erlaubt sind und ob es Unterschiede zwischen eigenen und gegnerischen Camps gibt. Bei Unklarheit nur das anzeigen, was das Spiel ohnehin zeigt.

## F9 – Hinweise in der Begleiter-App

**Ziel:** Die Begleiter-App gibt dir während des Spiels immer wieder hilfreiche Hinweise.

Erlaubte Beispiele:
- „Noch 250 Gold bis zu deinem nächsten Kern-Item.“
- Nächstes Item aus deinem Build, situative Empfehlungen je nach gegnerischen Items (z. B. „Der Gegner baut Heilung, denk an Anti-Heal“).
- Matchup-Tipps aus F3 zum Nachlesen.
- Anstehende Objectives (aus F8).

**Richtlinie ⚠️:** Nicht erlaubt sind Hinweise, die dir sagen, was du gerade tun sollst (z. B. „Geh zurück, der Jungler kommt“), und Power-Spike-Alerts der Gegner (z. B. „Gegner ist Level 6“). Hinweise müssen informativ bleiben und dürfen keine Anweisungen zum Spielgeschehen sein.

## F10 – Jungler „zuletzt gesehen“ ⚠️

**Idee:** Die Begleiter-App zeigt, wann der gegnerische Jungler zuletzt gesichtet wurde.

- Die Live-API liefert **keine Positionen** und **keine Pings**. Die Sichtung muss also von Hand eingetragen werden, per Hotkey oder Klick in der Begleiter-App („Jungler gesehen: Botlane“).
- Angezeigt wird nur die vergangene Zeit, z. B. „vor 45 s, Botlane“, ohne Warnungen oder Anweisungen.
- **Richtlinie:** Grauzone. Vor der Umsetzung prüfen bzw. bei Riot anfragen. Wird es nicht freigegeben, entfällt das Feature.

## F11 – Summoner-Spell-Tracker der Gegner ⛔

> **Nicht erlaubt.** Riot verbietet Timer für gegnerische Summoner-Spells, auch wenn man sie von Hand startet. Wird nicht umgesetzt, solange sich die Richtlinien nicht ändern.

Ursprüngliche Idee (zur Dokumentation):
- Benutzt ein Gegner z. B. Flash, trägst du das ein, und die App rechnet den Cooldown aus.
- Einberechnet werden Summoner-Spell-Haste aus Kosmischer Einsicht (Rune) und Ionischen Stiefeln der Klarheit (Item).
- Technische Hürden: Pings im Spiel kann die API nicht auslesen, die Eingabe müsste also per Klick oder Hotkey erfolgen. Automatisch in den Chat schreiben ist nicht erlaubt. Kosmische Einsicht ist über die API nicht erkennbar (nur Keystone und Runen-Bäume) und müsste von Hand angegeben werden.

## F12 – Ultimate-Tracker der Gegner ⛔

> **Nicht erlaubt.** Riot verbietet ausdrücklich das Tracken gegnerischer Ultimate-Cooldowns (seit März 2025). Wird nicht umgesetzt.

Ursprüngliche Idee (zur Dokumentation): Nutzt ein Gegner seine Ultimate, trägst du das ein, und die App berechnet den Cooldown aus Level, Ability Haste der Items und Rang der Ultimate.

---

## Datenquellen (Überblick)

| Quelle | Wofür |
|---|---|
| **League Client API (LCU)** | Champ Select (Picks, Bans, Rollen), Runenseiten und Item-Sets importieren, Spielphase erkennen. |
| **Live Client Data API** (lokal während des Spiels) | Eigene Werte (Gold, Level, Fähigkeiten), Scoreboard-Daten aller Spieler (KDA, CS, Items), Spielzeit, Events (Objective-Kills usw.). |
| **Riot Web API** | Ränge, Match-Historie, Mastery der Mitspieler; Grundlage für eigene Statistiken. Für den Produktivbetrieb ist ein freigegebener API-Key von Riot nötig. |
| **Eigene Statistik-Datenbank** | Aggregierte Winrates, Builds und Runen pro Matchup, gewonnen aus Match-Daten. |
| **Eigene Inhalte** | Matchup-Tipps (F3), Spawn-Zeiten der Objectives (F8). |

## Abgrenzung zu Blitz / Porofessor

- **Feingranulare Toggles:** Jedes Feature und jeder Overlay-Wert lässt sich einzeln abschalten.
- **Matchup statt Champion:** Builds, Runen, Skill-Reihenfolge und Tipps werden auf den Lane-Gegner zugeschnitten.
- **Second-Screen-Konzept:** Klare Aufteilung in ein minimales Overlay und eine ausführliche Begleiter-App.
- Eigenes Design und eigene Inhalte, keine Kopie bestehender Apps.

## Offene Fragen

1. **Plattform:** Eigenständige App (z. B. Electron) oder Overwolf bzw. ow-electron? Davon hängen Overlay-Technik, Verteilung und Riot-Freigabe ab.
2. **Woher kommen die Matchup-Tipps (F3)?** Selbst geschrieben, Community-Beiträge, aus Daten generiert oder KI-gestützt mit Prüfung?
3. **Woher kommen die Statistiken (F1, F2, F4)?** Eigene Auswertung von Match-Daten (aufwendig, aber unabhängig) oder externe Quelle?
4. **Objective-Timer und Jungler-Tracking (F8, F10):** Riot-Freigabe klären.
5. **Unterstützte Modi:** Nur Summoner's Rift (Ranked/Normal) oder auch ARAM und andere Modi?
6. **Sprache:** Nur Deutsch oder von Anfang an mehrsprachig (Deutsch/Englisch)?
