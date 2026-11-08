# Techstack und Architektur

Dieses Dokument legt einen umsetzbaren Startpunkt für den LOL Companion fest. Die App ist als privates Projekt für den Entwickler und eine kleine private Gruppe geplant, nicht für öffentliche Nutzung. Client- und Live-Daten bleiben auf dem Rechner; ein eigener Server gehört nicht zum MVP.

## Stack-Entscheidung

| Bereich                               | Entscheidung                                          | Einsatz                                                                                                                                                             |
|---------------------------------------|-------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sprache und Runtime                   | C# mit **.NET 10 LTS**                                | Gemeinsame Runtime für App, Domäne und Adapter. .NET 10 wird bis November 2028 unterstützt.                                                                         |
| Desktop-UI                            | **WPF**                                               | Hauptfenster, Einstellungen, Zweitbildschirm-Ansicht und erster Overlay-Prototyp. Windows-only ist für den LoL-Use-Case zunächst akzeptiert.                        |
| Veröffentlichung                      | **Self-contained Windows-Installer**                  | Die App bringt die benötigte .NET Runtime mit; Nutzer müssen .NET nicht separat installieren. Das konkrete Installer-Werkzeug wird bei der Implementierung gewählt. |
| Sprache                               | **Deutsch**                                           | Die erste Version und ihre Inhalte erscheinen ausschließlich auf Deutsch. Weitere Sprachen sind nicht Teil des MVP.                                                 |
| UI-Architektur                        | MVVM mit `CommunityToolkit.Mvvm`                      | ViewModels halten UI-Zustand und Aufrufe; Domänenlogik bleibt unabhängig von WPF.                                                                                   |
| Dependency Injection und Lebenszyklus | `Microsoft.Extensions.Hosting`                        | Start/Stop der App, Services, Konfiguration und geordnete Beendigung.                                                                                               |
| Lokale Speicherung                    | **SQLite** mit `Microsoft.EntityFrameworkCore.Sqlite` | Einstellungen, Feature-Toggles, lokale Match-Historie und später ein Daten-Cache. Schemaänderungen über EF-Core-Migrationen.                                        |
| Riot-Client-Integration               | Eigener `Lcu`-Adapter                                 | Lokale Client-Endpunkte für Spielphase, Champ Select und Importe; getrennt von UI und Domäne.                                                                       |
| Live-Spiel-Daten                      | Eigener `LiveClient`-Adapter                          | Live Client Data API während eines laufenden Spiels; nur notwendige Felder regelmäßig lesen.                                                                        |
| Riot-Web-API                          | `RiotApi`-Adapter über `HttpClient`                   | Nutzer hinterlegen ihren persönlichen Riot-API-Key lokal. Der Key wird im Windows Credential Manager gespeichert und nie in die App eingebettet.                    |
| Spielinhalte                          | Data Dragon zuerst                                    | Champion-, Item- und Rune-Metadaten passend zum Patch lokal cachen. CommunityDragon bleibt eine optionale Quelle, nicht die Laufzeit-Abhängigkeit des MVP.          |
| Protokolle                            | `Microsoft.Extensions.Logging`                        | Lokale, rotierende Logs ohne Zugangsdaten, Tokens oder unnötige Spielerdaten.                                                                                       |

Versionsupdates für .NET und NuGet-Pakete werden kontrolliert und gebündelt vorgenommen. Die App zielt auf `net10.0-windows`; die konkrete Windows-Mindestversion wird vor dem ersten Release festgelegt.

## Architektur

Die Anwendung startet als **modularer Monolith** in einem Prozess. Ein lokaler Backend-Dienst ist für das MVP nicht nötig. Abhängigkeiten zeigen nach innen:

```text
WPF Views / ViewModels
        ↓ Anwendungsfälle
Application (Use Cases, Zustandsfluss)
        ↓ Ports / Interfaces
Domain (Modelle, Regeln, Feature-Policies)
        ↑ Implementierungen
Infrastructure (LCU, Live API, Riot API, SQLite, Data Dragon)
```

- **Domain** enthält League-unabhängige Modelle wie `GamePhase`, `Matchup`, `FeatureSettings` und Empfehlungen. Keine WPF-, HTTP- oder SQLite-Typen.
- **Application** koordiniert Anwendungsfälle, etwa „Spielphase beobachten“ oder „Matchup-Daten laden“. Sie prüft Feature-Toggles, bevor Datenquellen gestartet oder abgefragt werden.
- **Infrastructure** implementiert Datenbank, Riot-Schnittstellen, Patch-Daten und Betriebssystemzugriffe. Fehler und Versionswechsel der externen APIs bleiben hier gekapselt.
- **Desktop** enthält WPF-Fenster, ViewModels, Ressourcen und den App-Start. Das Overlay bekommt eine eigene View und verwendet dieselben Anwendungsfälle wie das Hauptfenster.

### Vorgeschlagene Solution-Struktur

```text
src/
  LolCompanion.Domain/
  LolCompanion.Application/
  LolCompanion.Infrastructure/
  LolCompanion.Desktop/
```

Ein zusätzliches Testprojekt pro Schicht kann angelegt werden, sobald die Implementierung beginnt. Es wird kein separates API- oder Microservice-Projekt vorgezogen.

## Datenfluss und Betriebsverhalten

1. Beim Start lädt die App Einstellungen und gespeicherte Toggles aus SQLite.
2. Ein `GameSessionMonitor` erkennt Client-/Spielphasen und veröffentlicht Zustandswechsel. UI-Fenster abonnieren den App-Zustand, statt selbst APIs abzufragen.
3. Der LCU-Adapter verbindet sich nur bei verfügbarem League Client. Die LCU ist laut Riot nicht offiziell unterstützt; Endpunkte und Verhalten können sich ohne Ankündigung ändern. Der Adapter muss daher ausfallen dürfen, ohne die App zu beenden.
4. Der Live-Client-Adapter wird erst für die aktive Partie gestartet und beim Ende oder Verbindungsverlust gestoppt. Er publiziert normalisierte Snapshots und Events an die Application-Schicht.
5. Die Riot-Web-API ist unabhängig von den lokalen APIs. Anfragen laufen über zentrale Rate-Limit-, Cache- und Fehlerbehandlung. Der persönliche Key wird zur Laufzeit aus dem Windows Credential Manager geladen und niemals in ein verteiltes Client-Binary eingebaut.
6. Data-Dragon-Dateien werden patchbezogen geladen und lokal gecacht. Ist kein Netz verfügbar, nutzt die App den letzten vollständigen Cache und zeigt dessen Patchstand an.

Für Polling und Wiederverbindung gelten konfigurierbare Intervalle mit Abbruch-Token, Backoff und einem einzigen aktiven Poller je Datenquelle. Datenquellen melden `Unavailable` statt wiederholt Exceptions bis in die UI durchzureichen.

## Grenzen für das MVP

- Windows 11 und Summoner's Rift zuerst; andere Plattformen und Modi werden gesondert bewertet.
- Hauptfenster, Einstellungen und automatische Phasenerkennung bilden die technische Basis.
- Als erste Live-Funktion eignet sich ein schmaler eigener Spieler-Status. Daten für nicht freigegebene oder richtlinienkritische Features werden nicht vorsorglich gesammelt.
- Champ-Select-Unterstützung startet mit Champion-/Rollenstatus. Fremde Spielernamen werden nicht verwendet, wenn sie anonymisiert sein müssen.
- Das Overlay startet als WPF-Overlay-Prototyp mit manuell einstellbarem Fensterverhalten. Kompatibilität mit exklusivem Vollbild, Anti-Cheat und Riots aktuellen Overlay-Regeln ist vor einem Release zu verifizieren. Kein Prozess-Injection, Memory Reading oder Manipulieren des Spiels.
- Für den MVP nutzen Zweitbildschirm-Ansicht und Overlay dieselbe WPF-Anwendung; das Overlay ist ein separates Fenster. Eine alternative Overlay-Plattform wird erst bewertet, falls der Prototyp die Anforderungen oder Freigaben nicht erfüllt.
- Matchup-Tipps und Builds werden kuratiert und als versionierte, patchbezogene Inhalte gepflegt. Die App ergänzt sie mit Champion-, Item- und Runen-Metadaten aus Data Dragon.
- Statistiken zu den eigenen Matchups werden lokal aus den eigenen verfügbaren Matchdaten berechnet. Stichprobengröße und Zeitraum werden angezeigt; bei wenigen Partien wird die Aussagekraft kenntlich gemacht. Es werden keine globalen Winrates aus persönlichen Keys abgeleitet.
- Eine externe oder globale Matchup-Datenquelle ist nicht Teil des MVP. Falls sie später ergänzt werden soll, werden Herkunft, Nutzungsrechte und Riot-Freigabe separat geprüft.
- Objective-Timer und Jungler-Tracking bleiben deaktiviert, bis die Feature-Richtlinien aus `features.md` geprüft sind.

## Matchup-Inhalte und persönliche Statistiken

- Kuratierte Tipps, Builds, Runen und Skill-Reihenfolgen liegen als versionierte Inhalte mit Patchbezug vor. Ein Inhaltseintrag kann für ein konkretes Matchup oder als Champion-Standard gelten.
- Eigene Matchup-Winrates und Verlaufswerte entstehen aus den Matchdaten des jeweiligen Nutzers und bleiben lokal in SQLite. Jede Anzeige enthält die Anzahl der berücksichtigten Spiele und den Zeitraum.
- Bei zu kleiner Stichprobe zeigt die App „zu wenig eigene Daten“ und kann auf den kuratierten Champion-Standard zurückfallen. Sie erfindet keine Winrate und vermischt keine Nutzerprofile.
- Für eine spätere Community-Statistik braucht es eine gesonderte, zulässige Datenquelle und Freigabe. Persönliche API-Keys werden nicht zum massenhaften Sammeln oder Weitergeben von Matchdaten eingesetzt.

## Persönliche Riot-API-Keys

Jeder Nutzer hinterlegt seinen eigenen persönlichen Riot-API-Key in der Desktop-App. Das Projekt bleibt auf private Nutzung durch den Entwickler und eine kleine private Gruppe beschränkt; persönliche Keys sind nicht für ein öffentliches Produkt vorgesehen. Der Key wird mit einem klaren Einrichtungs- und Prüfschritt angenommen und im Windows Credential Manager abgelegt. Beim Start wird er nur bei Bedarf in den Speicher geladen; Logs und Fehlermeldungen dürfen weder den Key noch vollständige Request-Header enthalten.

Die App weist auf Ablauf oder fehlende Berechtigungen hin und bietet an, den Key zu ersetzen. API-Aufrufe laufen direkt vom Rechner des Nutzers über den `RiotApi`-Adapter. Ein eigener Companion-Service und ein gemeinsam genutzter Produktions-Key sind für das MVP nicht vorgesehen. Die niedrigen Rate Limits persönlicher Keys werden durch Cache, Drosselung und Wiederverwendung vorhandener Matchdaten berücksichtigt. Nutzungsbedingungen und Berechtigungen des jeweiligen Keys sind bei der Implementierung zu beachten. Eine öffentliche Veröffentlichung würde einen neuen API-Zugangs- und Freigabeentscheid erfordern.

## Sicherheit und Datenschutz

- Zugangsdaten und lokale Konfiguration gehören in den Benutzer-Konfigurationsordner oder Windows Credential Manager, niemals in Git oder Logs.
- `.env.example` enthält ausschließlich Variablennamen und Beispielwerte ohne echte Zugangsdaten.
- Standardmäßig werden Spieldaten lokal verarbeitet. Telemetrie oder Uploads sind aus und werden nur mit klarer Einwilligung ergänzt.
- Persönliche Daten von Mitspielern werden nur geladen, wenn das Feature aktiv und die Nutzung zulässig ist; Cache-Aufbewahrung wird begrenzt.
- Lokale LCU-Zugangsdaten sind besonders sensibel: nur im Speicher halten, nicht ausgeben und nach Ende der Sitzung verwerfen.

## Noch zu entscheiden


## Quellen

- [Microsoft: .NET Releases und Support](https://learn.microsoft.com/dotnet/core/releases-and-support) – .NET 10 LTS, Support bis November 2028.
- [Microsoft: WPF-Dokumentation](https://learn.microsoft.com/dotnet/desktop/wpf/) – WPF als Windows-UI-Framework für .NET.
- [Riot Developer Portal: League of Legends APIs](https://developer.riotgames.com/docs/lol) – LCU als nicht offiziell unterstützte Schnittstelle und Live Client Data API für aktive Spiele.

Riot-Richtlinien und API-Bedingungen ändern sich. Die Richtlinienprüfung in `features.md` ist vor jedem Release zu aktualisieren.
