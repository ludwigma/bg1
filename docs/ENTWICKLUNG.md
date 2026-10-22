# Entwicklung und Validierung

## Lokales Setup

Das Repository enthält `package-lock.json`. Für einen reproduzierbaren
Dependency-Stand:

```sh
npm ci --ignore-scripts --no-audit --no-fund
npm start
```

Die Analyse nutzte Node `v26.10.0` und npm `11.19.1`.
Das Manifest legt keine eigene Nodeversion über `engines` fest.
Beim Installieren wurden 698 Pakete hinzugefügt; `inflight` und `glob@7`
meldeten Deprecation-Warnungen. Ein Vulnerability-Audit wurde nicht durchgeführt.

Vite verwendet Root `src`, Base `/bg1/`, Port 3000, Host `0.0.0.0` und die
Zertifikatsdateien in `tls/`. Der Dev-Server ist damit grundsätzlich auch über
andere Interfaces erreichbar. Der Responseheader erlaubt Origin `*`.
Ob die TLS-Konfiguration der installierten Vite-Version unmittelbar als
Dateipfade funktioniert, wurde nicht mit einem gestarteten HTTPS-Server geprüft.

**Wichtig:** Die App entscheidet anhand des Browser-Origins. Ein gewöhnlicher
Besuch von localhost ist deshalb kein vollständiger Fachoberflächen-Smoke-Test:
`App` leitet unbekannte Origins auf die externe Startseite weiter.
Der Fork enthält keinen vollständigen eigenen Launcher oder Loginbackend.
`bg1-dev.ts` stellt den React-Refresh-Preamble für den Skriptbetrieb bereit.

## Befehle

| Befehl                    | Bedeutung / Eigenheit                                               |
| ------------------------- | ------------------------------------------------------------------- |
| `npm start`               | Vite-Devserver                                                      |
| `npm run build`           | Löscht zuerst `src/api/diu.ts`, dann Vite-Build                     |
| `npm run preview`         | Vorschau eines vorhandenen Builds mit Serverkonfiguration           |
| `npm test -- --runInBand` | Jest in UTC, serieller vollständiger Lauf                           |
| `npm run lint`            | ESLint nur über `src`                                               |
| `npm run typecheck`       | Strenger TypeScriptcheck ohne Emit                                  |
| `npm run checkall`        | Tests → Lint → Typecheck; stoppt am ersten Fehler                   |
| `npm run coverage`        | Coverageausgabe nach `.coverage`; separat, ohne TZ-UTC-Skriptpräfix |
| `npm run format`          | Formatiert `src`; erfasst die neue Markdown-Doku nicht              |

Für die Analyse wurde **direkt `./node_modules/.bin/vite build`** verwendet.
So konnte der echte Bundler geprüft werden, ohne den vorgeschalteten
Löschschritt des npm-Skripts auszuführen. Das beseitigt den Importfehler nicht.

## Buildstruktur

`vite.config.mts` hat drei Eingaben: `bg1.tsx`, `bg1.css`, `responder.html`.
Erzeugte Dateien haben feste Namen ohne Contenthash. Ausgabe ist `dist`;
`emptyOutDir: false` lässt ältere Dateien bei späteren Builds bestehen.
Es gibt keine Serviceworker-/Offlineimplementierung im untersuchten Code.
Obfuscation ist auf `src/api/diu/*` beschränkt, nicht auf das gesamte Projekt.

TypeScript: `strict`, `noUncheckedIndexedAccess`, Bundler-Modulauflösung,
React-JSX und Alias `@/` nach `src`. Inkrementelle Checks können die ignorierte
Datei `tsconfig.tsbuildinfo` erzeugen. Jest nutzt ts-jest und jsdom mit demselben
Alias; Testing Library überprüft die React-Oberfläche.

## Ergebnisse vom 11. Oktober 2026

Die vier Checks wurden unabhängig ausgeführt, damit ein früher Fehler die
anderen Ergebnisse nicht verdeckt.

| Prüfung                   | Ergebnis                                                                                             |
| ------------------------- | ---------------------------------------------------------------------------------------------------- |
| Dependencyinstallation    | Erfolgreich                                                                                          |
| `npm test -- --runInBand` | Exit 1; 53 Suiten bestanden, 8 fehlgeschlagen; 218 Tests bestanden, 11 fehlgeschlagen, 229 insgesamt |
| `npm run lint`            | Exit 0; keine ESLint-Befunde                                                                         |
| `npm run typecheck`       | Exit 2; zwei TS2307-Fehler durch fehlendes `../diu`                                                  |
| Direkter Vite-Build       | Exit 1; `Could not resolve "../diu" from "src/api/ll/dlr.ts"`                                        |

Der Jest-Lauf dauerte ungefähr 6,9 Sekunden. Die 229 Tests umfassen nur
ausführbare Suiten; die Testfälle der nicht startenden LL-API-Suite fehlen darin.
Die erfolgreichen Checks beweisen keine echte Disney-API-Kompatibilität.

### Fehlgeschlagene Testsuiten

| Suite                                  | Beobachtung                                                         |
| -------------------------------------- | ------------------------------------------------------------------- |
| `src/api/ll.test.ts`                   | Mockziel `./diu` fehlt; Suite startet nicht                         |
| `BookExperience.test.tsx`              | Erwarteter `Change`-Button fehlt                                    |
| `BookingDetails.test.tsx`              | Erwartete LL-Änderungsoberfläche fehlt                              |
| `Home/MultiPassList.test.tsx`          | Erwartete LL-Zeitanzeige im bisherigen Format fehlt                 |
| `ModifyButton.test.tsx`                | Fünf Tests scheitern an Erwartung des ausgeblendeten Modify-Buttons |
| `YourDay.test.tsx`                     | `ll.rules` fehlt im Testcontext; TypeError beim Rendern             |
| `Home.test.tsx`                        | Erwartete bisherige LL-Oberfläche passt nicht zum Stand             |
| `BookExperience/OfferDetails.test.tsx` | Erwarteter `Change`-Button fehlt                                    |

Die Suitebezeichnungen beziehen sich auf Unterpfade von
`src/components/ll/screens`, außer `ModifyButton` unter `src/components/ll`.

### Ergänzende lokale Reproduktionen

Die TypeScriptimplementierungen wurden im Speicher nach JavaScript übersetzt
und mit kontrollierter Uhr bzw. gemocktem Netzwerk aufgerufen. Dafür wurden
keine Produktionsdateien verändert und keine echten Requests versandt.

| Prüfung                                                                    | Beobachtung                                                                    |
| -------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `RateLimit(1)`, Überschreitung, dann neun Sekunden später neuer Request    | Weiterhin `RateLimitExceeded`                                                  |
| `ParkTime(0,10).add({minutes:-20})`                                        | `00:00:00` statt `23:50:00`                                                    |
| Zwei unterschiedliche POST-Bodies an dieselbe URL unmittelbar nacheinander | Ein Fetch-Aufruf, beide Antworten enthalten den ersten Body                    |
| Vergleich zweier ISO-normalisierter `DateTime`-Objekte                     | Stringbasierte Reihenfolge funktioniert; numerische Konvertierung ergibt `NaN` |

Die vollständigen Prüflogs lagen zur Analyse unter
`/tmp/bg1-analysis-{tests,typecheck,lint,build}.log`. Sie sind temporäre lokale
Hilfsdateien, kein Teil der versionierten Dokumentation.

## Empfohlene Validierung nach Änderungen

Zuerst den fehlenden Modulpfad und explizite Feature-Regeln lösen. Dann
Typecheck, Build und den vorhandenen Gesamtcheck ausführen. Bei Korrekturen
der beschriebenen Laufzeitprobleme gezielte Tests für falsche POST-Zuordnung,
fehlgeschlagene Stornierung, Rate-Limit-Erholung, Retry ohne Fortschritt,
vertauschte Antwortreihenfolge und Mitternacht ergänzen.

Ein echter Integrationsnachweis benötigt danach eine kontrollierte Umgebung
mit dem vorgesehenen Origin und autorisierter Anmeldung. Für eine reine lokale
Codeanalyse sind echte Buchungen kein sinnvoller Verifikationsschritt.

## Änderungen dieser Analyse

Ergänzt wurden fünf Markdown-Dateien in `docs/` und ein Verweis mit
Statushinweis im bestehenden README. Anwendungscode, Dependencymanifest und
Lockfile wurden nicht geändert. Abhängigkeiten wurden lokal installiert;
Build-/Typecheck-Hilfsartefakte sind durch bestehende Ignore-Regeln abgedeckt.
