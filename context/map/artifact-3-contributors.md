# Mapa kontrybutorów — kto wie co i o co go zapytać (git)

> Zakres: ostatnie 12 miesięcy (2025-06-10 → 2026-06-10), commity bez merge'y.
> Filtrowanie botów/agentów: w historii **nie ma** commitów botów ani agentów
> (Claude/Codex/Copilot/dependabot) — adresy `*@users.noreply.github.com` to
> prywatne adresy GitHub realnych osób. Zunifikowano aliasy tożsamości
> (np. `Viacheslav Klimov` = `VIacheslavKlimov` = `ViacheslavKlimov`,
> `Vladyslav_Prykhodko` = `Vladyslav Prykhodko`, `ArtemDzhereleiko` =
> `Artem Dzhereleiko`, `dashevchenko` = `Daria Shevchenko`).

## Top 5 obszarów wymagających potencjalnego kontaktu

Wybrane na przecięciu aktywności (artifact-1) i ryzyka strukturalnego (artifact-2):

1. **Calculated Fields — backend** (`service/cf`, `actors/calculatedField`, `common/data/cf`) — dominujący temat roku; skomplikowany model aktorowy i stany.
2. **Splot widget/dashboard w ui-ngx** (`shared/models`, `core`, `home/components/widget`) — 265-modułowa silnie spójna składowa; każda zmiana tu jest ryzykowna.
3. **Alarm rules** (UI `alarm-rules` + backend) — nowa funkcjonalność z Q4 2025, częściowo oparta o CF.
4. **Integracja AI/LLM** (`common/data/ai`, `rule/engine/ai`, `service/ai`) — nowy podsystem (LangChain4j, multi-provider).
5. **Oś danych** (`dao` + `common/dao-api` + `common/data`) — kanoniczny konwój zmian encji (115 wspólnych commitów); dotyka też migracji SQL i `queue.proto`.

## Linia wsparcia per obszar

### 1. Calculated Fields — backend

| Osoba | Commity | Profil aktywności |
|---|---:|---|
| **IrynaMatveieva** | 142 | Rdzeń CF: cache per-tenant, persystencja debug eventów w systemie aktorów, obsługa błędów inicjalizacji CF, relacje encji. Pierwsza linia pytań o stany i cykl życia CF. |
| **dshvaika** | 95 | Najtrudniejsze scenariusze: propagacja po relacjach, geofencing CF, alarm CF (startTs), restore/readiness stanów po restarcie, limity interwałów w profilach tenantów. Pytania o semantykę i edge-case'y. |
| **Viacheslav Klimov** | 50 | Integracja CF z resztą platformy, testy (AlarmRulesTest), wersjonowanie. |
| dashevchenko | 20 | Wsparcie punktowe. |

### 2. Splot widget/dashboard (ui-ngx: models/core/widget)

| Osoba | Commity | Profil aktywności |
|---|---:|---|
| **Maksym Tsymbarov** | 90 | Bieżące utrzymanie UI: widgety mapowe (drift labelek, viewport), tabele/kolumny, obsługa błędów HTTP, CVE frontendowe. Dobry kontakt od zachowań istniejących widgetów. |
| **Vladyslav Prykhodko** | 89 | Infrastruktura UI: dynamic forms, key-filtry (warunki OR), pipes, sanitizacja markdown, zależności/CVE. Kontakt od `shared/components` i fundamentów formularzy. |
| **ArtemDzhereleiko** | 58 | Widgety gateway + UI dla CF/geofencing; styk widgetów z CF. |
| **Igor Kulikov** | 22 | Architekt frontendu: nowy widget HTML Container (settings framework, rejestracja widgetów), aktualizacje echarts/ngx-flowchart. **Najlepszy adresat pytań o architekturę splotu `widget.models.ts`/`core/utils.ts`** (autor większości historycznego frameworku). |
| Ekaterina Chantsova | 28 | Wsparcie w komponentach UI. |

### 3. Alarm rules

| Osoba | Commity | Profil aktywności |
|---|---:|---|
| **ArtemDzhereleiko** | 31 | Cała warstwa UI alarm-rules (tabele, dialogi, naprawy konsolowych błędów). |
| **Viacheslav Klimov** | 27 | Backend i testy (AlarmRulesTest), integracja z CF. |
| Vladyslav Prykhodko | 11 | Key-filtry w UI (warunki OR) używane przez alarm rules. |
| Maksym Tsymbarov | 7 | Poprawki UI. |

### 4. Integracja AI/LLM

| Osoba | Commity | Profil aktywności |
|---|---:|---|
| **Dmytro Skarzhynets** | 69 | Praktycznie jednoosobowy obszar: LangChain4j (upgrade'y, piny wersji), structured output (JSON Schema) cross-provider, parametry modeli (penalties dla Gemini/Vertex), autocomplete modeli, TbAiNode. **Bus factor = 1.** |
| dashevchenko | 8 | Wsparcie punktowe. |

### 5. Oś danych (dao + dao-api + common/data)

| Osoba | Commity | Profil aktywności |
|---|---:|---|
| **dashevchenko** | 105 | Logika domenowa i uprawnienia (moderacja komentarzy alarmów, walidacje), refaktoryzacje DAO. Najaktywniejsza osoba repo w ostatnim kwartale. |
| **Viacheslav Klimov** | 93 | Przekroje platformowe: transport (lock convoy), Netty/MQTT, bumpy CVE, OpenAPI/klienci, wersjonowanie. Kontakt od spraw „między modułami". |
| **Andrii Landiak** | 51 | Kafka/kolejki: OAUTHBEARER, konfiguracja MSA, `TbKafkaSettings` — naturalny kontakt także dla `queue.proto`. |
| IrynaMatveieva / dshvaika | 46/40 | Encje powiązane z CF. |
| Dmytro Skarzhynets | 37 | Walidatory, wydajność JPA (blocking queries). |

## Wnioski operacyjne

- **Aktualność kontaktów (ostatni kwartał, od 2026-03):** najaktywniejsi:
  Viacheslav Klimov (106), dashevchenko (87), Maksym Tsymbarov (57),
  Vladyslav Prykhodko (41), Andrii Landiak (39). **IrynaMatveieva i dshvaika
  nie commitują od ~Q1 2026** — wiedza o rdzeniu CF może być trudniej dostępna;
  pierwszym żywym kontaktem dla CF jest dziś Viacheslav Klimov.
- **Ryzyka bus-factor:** AI/LLM = Dmytro Skarzhynets (sam, i mocno zwolnił —
  9 commitów w ostatnim kwartale); architektura frameworku widgetów = Igor
  Kulikov (6 commitów/kwartał, ale unikatowa wiedza o splocie z artifact-2).
- **Uwaga na tożsamości:** `dshvaika` (dshvaika@thingsboard.io) i `Andrii
  Shvaika` (ashvayka@thingsboard.io) to **różne adresy** — nie łączyć ich bez
  potwierdzenia; w analizie traktowani osobno.
- Historia nie zawiera commitów botów ani agentów AI — wszystkie liczby to
  praca ludzi.

---
*Wygenerowano: 2026-06-10 na podstawie `git log --since=2025-06-10` (master).*
