# Befunde und priorisierte Verbesserungen

Stand und Umfang: [Dokumentationsindex](README.md).
„Reproduziert“ bezeichnet eine lokale Prüfung; „Codebefund“ eine aus dem
Quelltext nachvollziehbare Aussage ohne vollständigen Browser-/Live-Test.
Die Prioritäten sind Vorschläge für die Weiterentwicklung dieses Forks.

## Übersicht

| ID  | Priorität | Befund                                                             | Evidenz                            |
| --- | --------- | ------------------------------------------------------------------ | ---------------------------------- |
| B01 | Hoch      | Fehlendes `diu` blockiert Build, Typecheck und LL-API-Tests        | Prüfungen ausgeführt               |
| B02 | Hoch      | Unterschiedliche POSTs können denselben Cacheeintrag verwenden     | Codebefund und lokale Reproduktion |
| B03 | Hoch      | Stornierungsfehler kann trotzdem lokal als Stornierung erscheinen  | Codebefund                         |
| B04 | Hoch      | Rate-Limiter bleibt nach Überschreitung gesperrt                   | Reproduziert                       |
| B05 | Mittel    | Ältere Requests können neueren Park-/Datumzustand überschreiben    | Codebefund                         |
| B06 | Mittel    | LL-Tracking und Ping können unbehandelte Promisefehler erzeugen    | Codebefund                         |
| B07 | Mittel    | VQ-Retry ohne Fortschritt / Status-0-Fehler verliert Kontext       | Codebefund                         |
| B08 | Mittel    | Zeitrechnung rückwärts über Mitternacht ist falsch                 | Reproduziert                       |
| B09 | Mittel    | OneID-/Timesync-Netzwerkpfade können unbegrenzt warten             | Codebefund                         |
| B10 | Mittel    | Responder validiert Origin und Nachrichtenform nicht               | Codebefund                         |
| B11 | Mittel    | Testannahmen passen teilweise nicht zu deaktivierten LL-Funktionen | Testausgabe und Quelltext          |
| B12 | Niedrig   | Unerreichbarer WDW-Code und doppelte Speicherung                   | Codebefund                         |

## B01 – Checkout ist nicht eigenständig baubar

**Belege:** [dlr.ts](../src/api/ll/dlr.ts), Zeilen 101 und 133;
[ll.test.ts](../src/api/ll.test.ts), Zeile 43;
[.gitignore](../.gitignore); [package.json](../package.json).

Der Adapter importiert `../diu`, aber weder dieses Modul noch eine passende
Deklaration ist vorhanden. Die Ignore-Datei enthält `src/api/diu`; das ist ein
Hinweis auf eine separat gehaltene Ressource, erklärt aber nicht ihre Funktion.
`npm run typecheck` meldet zwei TS2307-Fehler, Vite bricht bei der Auflösung ab,
die LL-Testsuite kann den Mockzielpfad nicht auflösen.

`rules.book = false` beseitigt diese Importabhängigkeit nicht. Ein deaktivierter
Button bewirkt kein automatisches Entfernen aller vom Bundle erfassten Methoden.
Das Buildskript löscht außerdem ausdrücklich `src/api/diu.ts` vor dem Build.

**Vorschlag:** Vorhandenen, tatsächlich beabsichtigten Lieferweg des Moduls
klären oder den deaktivierten Adapterpfad sauber aus dem öffentlichen Build
entkoppeln. Ein Dummy-Modul ist kein Beleg für funktionierende Buchungen.
Die Herkunft der Zusatzdaten darf aus dem fehlenden Code nicht erfunden werden.

## B02 – Deduplizierung schreibender Requests

**Beleg:** [fetch.ts](../src/fetch.ts), `checkCache`.

Der Cachekey ist `init.method + url`, der Cache ist modulweit und lebt 10 ms.
Zwei POSTs gegen dieselbe URL mit verschiedenen Gästen, Offer-IDs, Bodies oder
Authheadern teilen denselben Promise. Dies kann eine zweite Aktion verhindern
und ihr das Resultat einer anderen Aktion zuschreiben.

Die lokale Reproduktion mit gemocktem `fetch` ergab bei zwei sofortigen POSTs
mit unterschiedlichen Bodies: nur ein Fetch-Aufruf und identisches Ergebnis
des ersten Bodys für beide Aufrufer. Die öffentlichen async-Aufrufe liefern
verschiedene Promiseobjekte, übernehmen aber dasselbe intern gecachte Ergebnis.
Es wurde kein externer Request ausgeführt.

**Vorschlag:** Deduplizierung auf sichere Leseoperationen beschränken und
Requestidentität vollständig berücksichtigen. Für schreibende Aktionen wären
explizite Busy-Sperren und, soweit vom Server unterstützt, Idempotenzmechanismen
passender. Eine längere Cachezeit allein verschärft die Verwechslungsgefahr.

## B03 – Stornierung zeigt Erfolg trotz fehlgeschlagenem DELETE

**Belege:** [CancelGuests.tsx](../src/components/ll/screens/CancelGuests.tsx),
`cancelBooking`; [useDataLoader.tsx](../src/hooks/useDataLoader.tsx).

Der Screen wartet auf `loadData`. Dieser fängt den Fehler jedoch ab und erfüllt
sein Promise trotzdem. Anschließend führt der Screen unabhängig vom Ergebnis
`goBack()` und `onCancel` mit lokal entfernten Gästen aus.

**Auswirkung:** Bei einem Netzwerk-/API-Fehler kann die Detailansicht Gäste als
storniert ausblenden, obwohl die Serverbuchung weiterbesteht. Der Fehlerflash
liegt zudem auf dem Screen, von dem gerade wegnavigiert wird.

**Vorschlag:** Navigation und lokales Entfernen nur im erfolgreichen Callback
ausführen oder dem Loader ein eindeutiges Erfolgsergebnis geben. Ein gezielter
UI-Test sollte fehlgeschlagenes DELETE und erhaltene Gäste prüfen.

## B04 – Rate-Limit ist eine dauerhafte Instanzsperre

**Beleg:** [ratelimit.ts](../src/ratelimit.ts), `RateLimit.enforce`.

Pro ganzzahliger `performance.now()`-Sekunde werden fünf Requests erlaubt.
Eine Überschreitung setzt `limitExceededTime`; dieses Feld wird danach niemals
zurückgesetzt. Jeder spätere Request derselben Instanz wirft erneut.

Lokale Reproduktion mit Grenze 1: erster Request bei t=1000 ms erlaubt,
zweiter gesperrt, weiterer bei t=10000 ms weiterhin `RateLimitExceeded`.
Im Zeitfenster 0 ist die Situation zusätzlich inkonsistent: Gesetzt wird 0,
die Prüfung verlangt aber `> 0`.

**Vorschlag:** Gewünschtes Verhalten ausdrücklich festlegen: Fensterreset,
Cooldown oder dauerhafter Schutz. Für einen gewöhnlichen Rate-Limiter
Fenster-/Cooldown-Ablauf implementieren und Wiederfreigabe testen.
Die Grenze ist pro Clientinstanz, nicht global über alle Disney-Requests.

## B05 – Keine Absicherung gegen verspätete Antworten

**Belege:** [ExperiencesProvider](../src/providers/ExperiencesProvider.tsx),
[useThrottleable](../src/hooks/useThrottleable.ts),
[fetch.ts](../src/fetch.ts).

Park-/Datumwechsel löscht die sichtbare Liste und startet neue Requests.
Ein vorheriger Request wird weder abgebrochen noch anhand einer Versionsnummer
ignoriert. Eine langsame Antwort aus Park A kann deshalb die Liste von Park B
überschreiben. Throttling verhindert auch keine parallelen aktiven Requests.
Ein Caller-Abortsignal wird im Fetchwrapper derzeit ersetzt.

**Vorschlag:** Loadergeneration oder Contextkey prüfen, optional externe
Abortsignale unterstützen. Nur zur aktuellen Park-/Datumkombination passende
Ergebnisse übernehmen; dies mit bewusst vertauschter Antwortreihenfolge testen.

## B06 – Tracking und Nebenmeldungen sind nicht isoliert

**Belege:** [ClientsContext](../src/contexts/ClientsContext.ts),
[LLClient.track / LLTracker.update](../src/api/ll.ts),
[ping.ts](../src/ping.ts), [DasPartiesProvider](../src/providers/DasPartiesProvider.tsx).

LL-Tracking ist async, wird aber nicht durch die Planaktualisierung awaited
oder mit Catch versehen. Es kann zusätzliche API-Requests erzeugen,
das Rate-Limit treffen oder nach einem neueren Update noch alte Daten speichern.
Ping wird nach Buchungen ebenfalls ohne Fehlerbehandlung gestartet.
Andere als `RequestError` aus dem DAS-Partyeffect werden weitergeworfen,
ohne dass ein React-Errorboundary diesen asynchronen Fehler auffängt.

**Vorschlag:** Nebenwirkungen mit explizitem Catch und begrenzter Fehlerwirkung
versehen; Tracking serialisieren oder versionieren und bei Bedarf vor der
Experience-Neuberechnung abschließen. Telemetriefehler sollten einen bestätigten
Facherfolg nicht beeinträchtigen.

## B07 – VQ-Fehlerbehandlung benötigt Fortschrittsinvariante

**Beleg:** [vq.ts](../src/api/vq.ts), `joinQueue` und `post`.

Die Rekursion ist nur zuverlässig begrenzt, wenn jeder Retry mindestens einen
Gast entfernt. Ein `INVALID_GUEST` ohne Konflikte für die aktuelle Party erfüllt
diese Voraussetzung nicht. Weiterhin reicht eine OK-Position mit nur einem
überschneidenden Gast; eine vollständige erfolgreiche Party wird nicht geprüft.

`post` reicht auch Fehlerstatus 0 als Response weiter. Deren `data: null` führt
in `joinQueue` zu einem TypeError statt zu einem verständlichen Netzwerkfehler.

**Vorschlag:** Nur bei tatsächlich kleinerer Party erneut versuchen,
Versuchszahl begrenzen, Responseform validieren und Status 0 als Transportfehler
beibehalten. Teilweise erfolgreiche Partys im Ergebnis ausdrücklich abbilden.

## B08 – Uneinheitliche Zeitgrenzen

**Belege:** [datetime.ts](../src/datetime.ts),
[SelectReturnTime](../src/components/ll/screens/SelectReturnTime.tsx),
[Clock](../src/components/Clock.tsx), [timesync.ts](../src/timesync.ts).

Reproduziert: `new ParkTime(0,10).add({minutes:-20})` ergibt `00:00:00`.
Erwartete Uhrzeit wäre `23:50:00`. Negative Reste und anschließendes Clamping
verursachen den Fehler. Beim Zeitraster verhindern gewöhnliche Stundenvergleiche
die korrekte Behandlung einer Schließzeit nach Mitternacht.

Zusätzlich gilt der Serveroffset nur für die sichtbare Uhr. Fachliche Zeitchecks
verwenden die Gerätezeit. `DateTime.now` ist keine synchronisierte Zeitquelle.

**Vorschlag:** Modulo über 86400 Sekunden normalisieren, Zeitsource konsistent
festlegen und Uhrzeit, Kalenderdatum sowie Parkdatum ausdrücklich unterscheiden.
Grenzfälle: Mitternacht, 04:00, Resortzeitzonen, DST und falsche Geräteuhr.

## B09 – Login und Zeitabgleich ohne vollständigen Fehlerabschluss

**Belege:** [LoginForm](../src/components/LoginForm.tsx), [timesync.ts](../src/timesync.ts).

OneID-Warteschleifen haben keinen Timeout. Schlägt SDK-Laden oder Initialisieren
fehl, können wartende Aufrufe offen bleiben. Timesync wartet auf einen Fetch
ohne eigenen Timeout. Die Loginmaske rendert selbst kein Ersatz-/Fehler-UI.

**Vorschlag:** Scriptfehler, Initialisierungsfehler und Zeitlimits in einen
sichtbaren Retryzustand führen. Listener im Lebenszyklus bereinigen und für
parallel gestartete Initialisierung ein gemeinsames Promise verwenden.

## B10 – Responder vertraut jeder passenden Nachricht

**Beleg:** [responder.ts](../src/responder.ts).

Der Messagehandler greift direkt auf `event.data.event` zu, nutzt eine normale
Objekttabelle und validiert weder Payload, Quelle, Port noch Origin.
ACKs werden mit Target-Origin `*` versandt. Eine nullförmige Nachricht oder ein
fehlender Port kann Fehler auslösen; auch geerbte Objektschlüssel sind kein
sauberer Dispatchvertrag.

**Vorschlag:** Erwartete OneID-Origins und Nachrichtenformen anhand des echten
Protokolls bestimmen, dann Quelle/Payload prüfen und Dispatch auf eigene Keys
begrenzen. Aus dem gezeigten Code ist kein Tokenabfluss bewiesen; der Befund
betrifft die fehlende Vertrauensprüfung und Robustheit.

## B11 – Regressionstests sind nicht konsistent mit Feature-Regeln

**Beleg:** Vollständiger lokaler Jest-Lauf; [Validierungsbericht](ENTWICKLUNG.md).

53 Suiten bestehen, acht scheitern. Davon startet eine wegen `diu` nicht.
Mehrere UI-Tests erwarten LL-Buttons oder Zeitauswahl, die durch aktuelle Regeln
ausgeblendet sind. `YourDay.test.tsx` liefert keinen vollständigen Clientcontext;
die neue Abfrage `ll.rules.book` schlägt deshalb bereits beim Rendern fehl.

Das lässt sich nicht pauschal als elf unabhängige Produktdefekte interpretieren.
Es zeigt aber, dass Tests und aktueller Funktionsmodus auseinanderlaufen.

**Vorschlag:** Den tatsächlichen deaktivierten Modus prüfen und historische
Buchungsabläufe über explizite Fixture-Regeln getrennt testen. Kein globales
Reaktivieren von Buchungen nur zum Bestehen der Tests. Danach Tests zu B02–B08
ergänzen, insbesondere Fehlermutationen und konkurrierende Antworten.

## B12 – Kleine Wartungsreste

- In `LLClientWDW.experiences` ist nach dem ersten `return exps` ein größerer
  Availability-Block unerreichbar. Entfernen oder bewusst wieder integrieren.
- `BookingDateProvider` speichert denselben Datumszustand in zwei identischen
  Effects. Ein Effect genügt.
- Mehrere Clients deklarieren eigene `onUnauthorized`-Properties, obwohl der
  zentrale AuthStore-Callback tatsächlich verwendet wird. Zuständigkeit vereinheitlichen.
- `LiveDataClient` cached Rohshowdaten ohne Tageswechsel-/TTL-Regel; länger offene
  Instanzen erhalten keinen neuen Tagesdatensatz.
- `NavProvider` nutzt modulweiten Hashcallback und persistiert versteckte Screens;
  Aufräumen/Backgroundaktivität sollte ausdrücklich zur Navigation passen.
- API-Typen nutzen häufig `any` und non-null assertions. Fremdantworten werden
  nur punktuell validiert; Typchecks garantieren keine gültige Serverantwort.

## Sinnvolle Reihenfolge

1. Lieferbarkeit und gewünschte Feature-Regeln klären; Build/Typecheck und Tests
   zum tatsächlichen Modus wiederherstellen.
2. Schreibende Aktionen absichern: POST-Deduplizierung, Stornierungserfolg,
   Mehrfachklicks und Teilfehler nach erfolgreicher Buchung.
3. Rate-Limit und VQ-Retry korrigieren; Async-Fehler und alte Antworten kontrollieren.
4. Parkzeitrechnung und Tageswechsel konsolidieren; gezielte Grenztests ergänzen.
5. Login-/Responderrobustheit und Daten-/Navigationslebenszyklen verbessern.

Die vorhandene API/UI-Trennung ist eine gute Grundlage für gezielte Änderungen.
Ein kompletter Architekturwechsel ist aus den belegten Problemen nicht erforderlich.
