# Mapa terytorium — gdzie projekt żyje (historia gita)

> Zakres analizy: ostatnie 12 miesięcy (2025-06-10 → 2026-06-10), branch `master`,
> 2323 commity (bez merge'y). Odfiltrowano szum: lockfile'y, `pom.xml`/`package.json`
> (bumpy wersji), pliki locale, configi yml/properties, SQL-e migracyjne, grafiki.

## 1. Aktywność — gdzie projekt był realnie dotykany

### TOP modułów (liczba dotknięć plików w commitach)

| # | Moduł | Dotknięcia | Co to jest |
|---|-------|-----------:|------------|
| 1 | `ui-ngx/src` | ~10 100 | Frontend Angular — zdecydowanie najgorętszy obszar |
| 2 | `application/src` | ~4 500 | Monolityczny serwer (serwisy, aktory, kontrolery REST) |
| 3 | `common/data` | ~3 100 | Współdzielone modele danych (DTO, konfiguracje) |
| 4 | `dao/src` | ~1 700 | Warstwa dostępu do danych (JPA/SQL) |
| 5 | `rule-engine/rule-engine-components` | ~620 | Nody rule engine'u |
| 6 | `common/transport` | ~580 | Wspólna warstwa transportowa (MQTT/CoAP/HTTP/LwM2M) |
| 7 | `msa/black-box-tests` | ~210 | Testy black-box mikroserwisów |
| 8 | `common/queue` | ~200 | Abstrakcja kolejek (Kafka itd.) |
| 9 | `common/dao-api` | ~185 | Interfejsy serwisów DAO |
| 10 | `common/message` | ~106 | Wspólne typy wiadomości |

### TOP katalogów hands-on (poziom niżej)

| # | Katalog | Dotknięcia | Temat |
|---|---------|-----------:|-------|
| 1 | `application/src/main/data/json/system/widget_types` | 694 | Definicje widgetów systemowych (JSON, pół-generowane — duże paczki zmian) |
| 2 | `ui-ngx/src/app/shared/components` | 572 | Wspólne komponenty UI |
| 3 | `application/.../server/controller` | 447 | Kontrolery REST API |
| 4 | `common/data/.../cf/configuration` | 298 | Konfiguracja Calculated Fields |
| 5 | `ui-ngx/.../components/alarm-rules` | 286 | UI reguł alarmowych (nowa funkcjonalność) |
| 6 | `application/.../service/cf/ctx/state` | 286 | Stany Calculated Fields |
| 7 | `application/src/test/.../controller` | 269 | Testy kontrolerów |
| 8 | `ui-ngx/.../widget/lib/settings/common/map` | 229 | Ustawienia widgetów mapowych |
| 9 | `application/.../service/cf` | 222 | Serwisy Calculated Fields |
| 10 | `common/data/.../ai/model/chat` | 219 | Modele AI/LLM (Anthropic, Bedrock, Azure OpenAI…) |

### TOP plików (po odfiltrowaniu szumu)

| # | Plik | Zmiany | Temat |
|---|------|-------:|-------|
| 1 | `application/.../service/cf/ctx/state/CalculatedFieldCtx.java` | 85 | Calculated Fields — rdzeń |
| 2 | `application/.../actors/calculatedField/CalculatedFieldEntityMessageProcessor.java` | 78 | CF — aktor per-encja |
| 3 | `application/.../actors/calculatedField/CalculatedFieldManagerMessageProcessor.java` | 71 | CF — aktor zarządzający |
| 4 | `application/.../config/SwaggerConfiguration.java` | 57 | Dokumentacja API |
| 5 | `application/.../service/cf/AbstractCalculatedFieldProcessingService.java` | 53 | CF — przetwarzanie |
| 6 | `application/.../service/cf/DefaultCalculatedFieldProcessingService.java` | 46 | CF — przetwarzanie |
| 7 | `application/src/test/.../cf/CalculatedFieldIntegrationTest.java` | 41 | CF — test integracyjny |
| 8 | `application/.../service/cf/ctx/state/BaseCalculatedFieldState.java` | 40 | CF — stan bazowy |
| 9 | `ui-ngx/src/app/shared/models/calculated-field.models.ts` | 36 | CF — modele frontendowe |
| 10 | `rule-engine/.../rule/engine/ai/TbAiNode.java` | 32 | Node AI w rule engine |

**Wniosek:** dominujący temat roku to **Calculated Fields** — 8 z 10 najgorętszych
plików to CF, spięte przez backend (aktory + serwisy), modele w `common/data`
i frontend (`calculated-field.models.ts`, dialogi). Drugi wyraźny wątek to
**integracja AI/LLM** (`common/data/.../ai/model/chat`, `TbAiNode`,
`Langchain4jChatModelConfigurerImpl` — 31 zmian).

## 2. Kwartały — jak przesuwał się nacisk pracy

| Kwartał | Commity | Lider aktywności | Dominujące tematy |
|---------|--------:|------------------|-------------------|
| Q3 2025 (06–09) | 456 | `common` / `application` | **AI/LLM** (`ai/model/chat` — 144, `ai/provider` — 52), rozkręcanie **Calculated Fields** (`cf/ctx/state`, `cf/configuration`) |
| Q4 2025 (09–12) | 832 (szczyt) | `application` | Szczyt prac nad **Calculated Fields** (state + serwisy + aktory ~450 dotknięć), masowa aktualizacja **widget_types** (672), start **alarm-rules** w UI |
| Q1 2026 (12–03) | 594 | `ui-ngx` (6 895 — eksplozja) | Wielki front-endowy push: `shared/components` (415), **widgety mapowe** (`settings/common/map` — 184), rule-node'y, kontrolery REST |
| Q2 2026 (03–06) | 441 | `ui-ngx` | Stabilizacja: testy klienckie (`application/src/test/.../client` — 104), filtry UI, device profile, kontrolery |

**Trend:** rok zaczął się backendowo (AI + CF w `common`/`application`),
przeszedł przez szczyt prac CF/alarm-rules w Q4, a od grudnia środek ciężkości
wyraźnie przesunął się na **frontend Angular** i stabilizację/testy.

## 3. Współzmiany — co zmienia się razem

### Najczęstsze pary modułów w tych samych commitach

| Para | Commity razem |
|------|--------------:|
| `application` + `common/data` | 288 |
| `application` + `dao` | 200 |
| `common/data` + `dao` | 127 |
| `common/dao-api` + `dao` | 79 |
| `application` + `ui-ngx` | 79 |
| `application` + `common/dao-api` | 77 |
| `application` + `rule-engine-components` | 64 |
| `application` + `common/proto` | 63 |

### Najczęstsze trójki

| Trójka | Commity razem |
|--------|--------------:|
| `application` + `common/data` + `dao` | 115 |
| `application` + `common/dao-api` + `dao` | 77 |
| `common/dao-api` + `common/data` + `dao` | 57 |
| `application` + `common/data` + `ui-ngx` | 54 |
| `transport/http` + `transport/lwm2m` + `transport/mqtt` (+coap) | 43 |

### Wnioski dla top 3 obszarów z rankingu

1. **`ui-ngx`** — w większości żyje własnym życiem (commity czysto frontendowe),
   ale pełnoprawny feature przechodzi pionowo: `application + common/data + ui-ngx`
   (54 commity). Zmiana modelu w `common/data` zwykle pociąga model TS w `ui-ngx`.
2. **`application`** — najbardziej sprzężony moduł w repo; prawie każda zmiana
   domenowa idzie w konwoju `application + common/data + dao` (115 commitów) —
   to kanoniczna pionowa oś: model → DAO → serwis. Jeśli dotykasz encji,
   spodziewaj się zmian we wszystkich trzech (plus `common/dao-api` przy nowych
   metodach serwisu).
3. **`common/data`** — wspólny słownik repo; sprzęga się ze wszystkim
   (application, dao, ui-ngx, rule-engine). Zmiany tutaj mają najszerszy promień
   rażenia. Osobne, symetryczne sprzężenie: cztery moduły `transport/*`
   (mqtt/coap/http/lwm2m) zmieniają się stadnie — wspólna abstrakcja
   w `common/transport`.

## 4. Wspólne mianowniki — pliki-huby całego repo

Pliki, które zmieniają się razem z wieloma różnymi obszarami naraz
(metryka: średnia liczba odrębnych obszarów w commitach zawierających plik):

| Plik | Commity | Śr. obszarów/commit | Commity ≥3 obszary | Rola |
|------|--------:|--------------------:|-------------------:|------|
| `common/proto/src/main/proto/queue.proto` | 38 | **5.1** | 24/38 | Kontrakt protobuf kolejek — zmiana = konwój przez transporty, application, edqs |
| `application/src/main/resources/thingsboard.yml` | 91 | **4.6** | 43/91 | Centralny config serwera — każdy nowy feature dopisuje sekcję |
| `application/src/main/data/upgrade/basic/schema_update.sql` | 49 | **4.5** | 26/49 | Migracje schematu — każda zmiana encji zostawia tu ślad |
| `ui-ngx/src/assets/locale/locale.constant-en_US.json` | 164 | 1.3 | 13/164 | Najczęściej zmieniany plik repo, ale sprzężony niemal wyłącznie z `ui-ngx` — to "lokalny", nie globalny mianownik |

**Najlepszy pojedynczy sygnał cross-repo:** `queue.proto` i `thingsboard.yml` —
jeśli commit je dotyka, prawie na pewno jest to zmiana wielomodułowa.
`schema_update.sql` + `dao/src/main/resources/sql/schema-entities.sql` to
analogiczny sygnał dla zmian modelu danych.

## 5. Weryfikacja istnienia plików

Wszystkie pliki wymienione w rankingach i sekcji hubów **istnieją w bieżącym
working tree** (zweryfikowano `test -f` dla 19 kluczowych plików oraz katalogów
`ai/model/chat` i `alarm-rules`) — analiza nie opiera się na plikach usuniętych
ani przeniesionych. Uwaga praktyczna: katalog `alarm-rules` w `ui-ngx` i pakiet
`cf/` to świeże obszary (powstały w analizowanym okresie), więc starsze ścieżki
sprzed Q3 2025 mogą nie mieć tu zastosowania.

---
*Wygenerowano: 2026-06-10, na podstawie `git log --since=2025-06-10` (2323 commity, bez merge'y).*
