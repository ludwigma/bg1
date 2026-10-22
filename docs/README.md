# BG1 – Dokumentation und Codeanalyse des Forks

Analysestand: 11. Oktober 2026 · Branch `mickey` · Commit
`f1f022a8959aeb1224e3523f214ba6859917139e` · Paketversion `0.2.0`.

## Kurzfassung

BG1 ist ein experimenteller Browserclient für Walt Disney World (WDW) und
Disneyland Resort (DLR). Die Anwendung verbindet React-Oberflächen mit
Disney-Endpunkten, lokalen Parkdaten und zusätzlichen BG1-Diensten. In diesem
Repository befindet sich kein eigenes Backend.

Die Architektur ist überschaubar: API-Clients normalisieren die Fremddaten;
React-Provider halten Pläne, Park, Datum und Umbuchungszustand; Screens bilden
die Bedienabläufe ab. Besonders umfangreich sind die Buchungsmodelle und die
resortspezifischen Lightning-Lane-Adapter.

**Der vorhandene Funktionsumfang ist nicht gleichbedeutend mit aktuell nutzbaren
Buchungen.** Für WDW und DLR gilt `rules.book = false`, außerdem
`rules.timeSelect = false`. Die normale Oberfläche zeigt deshalb LL-Zeiten statt
Buchungsbuttons und unterdrückt Umbuchung und Zeitauswahl. Die entsprechenden
API-Methoden existieren weiterhin. Stornierung bestehender, als stornierbar
gemeldeter LLs wird nicht über `rules.book` abgeschaltet. VQ- und DAS-Abläufe
sind im Code weiterhin vorgesehen; ihre reale Verfügbarkeit wurde nicht geprüft.

Der Checkout ist derzeit nicht vollständig baubar: Der DLR-Adapter importiert
das nicht mitgelieferte Modul `src/api/diu`. Tests sind teilweise an die frühere
LL-Oberfläche gekoppelt. Hinzu kommen reproduzierbare Fehler in Rate-Limit und
Zeitrechnung sowie Risiken bei parallelen Requests und fehlgeschlagenen
Stornierungen.

## Dokumente

| Dokument                                      | Inhalt                                                                                            |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| [Architektur](ARCHITEKTUR.md)                 | Einstieg, Origin-Auswahl, Komponenten, Provider, Datenmodell, Speicher und externe Abhängigkeiten |
| [Tiefe Funktionsanalyse](FUNKTIONSANALYSE.md) | Methodenverhalten, API-Abläufe, Zustandsübergänge, Sortierung, Zeitmodell und Randfälle           |
| [Befunde und Prioritäten](BEFUNDE.md)         | Belegte Probleme, Auswirkungen, Reproduktionen und Verbesserungsvorschläge                        |
| [Entwicklung und Validierung](ENTWICKLUNG.md) | Setup, Build-Eigenheiten, Testergebnisse und Grenzen der Prüfung                                  |

## Umfang und Aussagekraft

Untersucht wurden die Einstiegspunkte, sämtliche API-Implementierungen,
Zeit-/Netzwerk-/Speicherhilfen, Provider und Hooks sowie die zentralen
LL-, VQ- und DAS-Screens mit ihren Tests. `src` enthält 218 Dateien,
davon 61 Testdateien. Icons und einfache Darstellungskomponenten werden nach
ihrer Rolle eingeordnet; diese Dokumentation kommentiert nicht jede JSX-Zeile.

Die Aussagen beziehen sich auf den lokalen Quelltext und lokale Prüfungen.
Es wurden keine Disney-Anmeldung, echten Buchungen oder Live-API-Aufrufe
durchgeführt. Testdaten und Mocks belegen das erwartete lokale Verhalten,
nicht die heutige Kompatibilität der externen Dienste. Die Dokumentation
verändert keine Anwendungslogik.
