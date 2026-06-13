---
title: Destylacja domeny — ThingsBoard (mapa domeny IoT)
created: 2026-06-13
type: domain-distillation
---

# Destylacja domeny: ThingsBoard

> **Produkt tej pracy to MAPA domeny, nie kod.** Trzy kroki: odkrycie → analiza →
> klasyfikacja. Wszystkie cytaty `plik:linia` zweryfikowane bezpośrednio w kodzie
> (commit roboczy `b7abed0664`, branch `master`) — patrz adnotacja przy każdej
> tabeli, które linie potwierdzono otwarciem pliku, a które przejęto z artefaktów
> mapy (`context/map/*`, `context/changes/*`) jako prior.

---

## KROK 0 — Kontekst projektu

### Dokumenty wizji / wymagań

**BRAK formalnego PRD ani `tech-stack.md`.** Przeszukano `context/foundation/`
(tylko `README.md` szablonu), korzeń repo i `context/`. Jedyne źródło wizji
produktu to **`README.md`** (korzeń) — sekcja „Features" i nagłówek. Z konieczności
**Ubiquitous Language wyprowadzam z README + kodu**, nie z dokumentu wymagań —
to świadome ograniczenie tej destylacji (oznaczone niżej).

Cytaty wizji ze źródła (`README.md`):
- `README.md:4` — *„Open-source IoT platform for data collection, processing,
  visualization, and device management."* — **cztery filary produktu.**
- `README.md:25-30` — getting-started wylicza: *connect devices*, *push data*,
  *build real-time dashboards*, *Customer + assign dashboard*, *define thresholds
  and trigger alarms*, *notifications via email, SMS, mobile apps, third-party*.
- `README.md:46` — *„Provision, monitor and control your IoT entities… Define
  relations between your devices, assets, customers or any other entities."*
- `README.md:54` — *„Collect and store telemetry data in scalable and
  fault-tolerant way. Visualize… with built-in or custom widgets and flexible
  dashboards."*

### Materiał wtórny (priory, nie wyprowadzane na nowo)

Repo niesie trzy gotowe artefakty z wcześniejszych analiz — traktuję je jako
zweryfikowane priory (ich własne twierdzenia strukturalne przeszły weryfikację
`ast-grep`):
- `context/map/repo-map.md` — mapa terytorium/struktury/kontrybutorów.
- `context/changes/os-encji/research.md` — trasa e2e osi encji (reprezentant
  **Device**), luki testowe, blast radius (zweryfikowane `ast-grep`).
- `context/changes/refactor-opportunities/research.md` — kandydaci na refaktor
  C1–C6 + ranking (zweryfikowane `ast-grep`).

### Stack i gdzie żyje logika biznesowa

Monolit **Java/Spring** (`application`) + frontend **Angular** (`ui-ngx`) +
moduły wspólne (`common/*`, `dao`, `rule-engine`, `transport/*`), spięte
**Kafką** (kontrakt `common/proto/.../queue.proto`) i **PostgreSQL**
(`dao/.../sql/schema-entities.sql`). Warstwy domeny (z `repo-map.md` §1, graf
modułów Mavena):

| Warstwa | Moduł | Rola |
|---|---|---|
| **Model domenowy** | `common/data` | Słownik repo: encje, ID, enumy, niezmienniki typów (fan-in 20 modułów) |
| **Persystencja + walidacja + cache** | `dao` | `*ServiceImpl`, walidatory, transakcje, eventy po-commit |
| **Orkiestracja** | `application/service` | `Tb*Service` (audyt, version control, powiadomienia), aktory CF |
| **REST** | `application/controller` | ~60 kontrolerów na bazie `BaseController` |
| **UI** | `ui-ngx` | Lustro TS modeli, dashboardy, widgety |
| **Reguły** | `rule-engine-components` | Nody przetwarzania komunikatów (0 cykli — wzorcowo czyste) |

**Kanoniczna taksonomia domeny:** `EntityType` (`common/data/.../EntityType.java:26-75`)
— **44 typy encji** z numerami proto, od `TENANT(1)`, `CUSTOMER(2)`, `DEVICE(6)`,
`ALARM(7)`, przez `RULE_CHAIN(11)`, `DEVICE_PROFILE(21)`, po najnowsze
`CALCULATED_FIELD(39)`, `JOB(41)`, `AI_MODEL(43)`, `API_KEY(44)`. Ślad ewolucji:
`// CALCULATED_FIELD_LINK(40), - was removed in 4.3` (`EntityType.java:66`).
*(zweryfikowano otwarciem pliku)*

---

## KROK 1 — Ubiquitous Language

Pojęcia wyciągnięte z README (wizja) **oraz** z kodu (nazwy encji/stanów/operacji).
Kolumna „Gdzie w kodzie" zweryfikowana bezpośrednio dla pozycji oznaczonych ✓;
pozycje oznaczone ≈ przejęto z harvestingu (plik istnieje, linia przybliżona).

| Pojęcie | Definicja (z kodu/README) | Cytat źródłowy | Gdzie żyje w kodzie |
|---|---|---|---|
| **Tenant** | Korzeń izolacji wielodzierżawnej; `SYS_TENANT_ID` zarezerwowany dla encji systemowych | README:46 „IoT entities" | ✓ `id/TenantId.java:30`, `SYS_TENANT_ID` `:36`, `isSystem()` `:55`; marker `HasTenantId.java` |
| **Customer** | Podmiot w obrębie tenanta (model resellera); encja bez `customerId` widoczna dla całego tenanta | README:28 „Create a Customer" | ✓ `Customer.java:32` (extends `ContactBased`); marker `HasCustomerId.java` |
| **Device** | Podstawowa encja IoT; nosi telemetrię, atrybuty, profil, credentiale | README:46 „Provision… devices" | ✓ `Device.java:45` (6 interfejsów: `HasLabel,HasTenantId,HasCustomerId,HasOtaPackage,HasVersion,ExportableEntity`) |
| **Asset** | Logiczny obiekt grupujący/reprezentujący byty fizyczne | README:46 „devices, assets" | ✓ `EntityType.ASSET(5)`; `asset/` |
| **Device Profile** | Szablon urządzenia: transport, schemat telemetrii, reguły alarmów, domyślny rule chain, OTA | (kod) | ✓ `DeviceProfile.java:45`; `transportType` `:64`, `defaultRuleChainId` `:70`, `profileData` `:80` |
| **Device Transport Type** | Protokół połączenia | (kod) | ✓ `DeviceTransportType.java:19-23` — `DEFAULT, MQTT, COAP, LWM2M, SNMP` |
| **Device Credentials** | Sekret uwierzytelniający urządzenie; 1 aktywny zestaw na urządzenie, auto-tworzony przy create | (kod) | ✓ `security/DeviceCredentialsType.java:20-23` — `ACCESS_TOKEN, X509_CERTIFICATE, MQTT_BASIC, LWM2M_CREDENTIALS` |
| **Telemetry / Attribute (KV)** | Para klucz-wartość; szereg czasowy (telemetria) lub atrybut; rdzeń zbierania danych IoT | README:54 „Collect… telemetry data" | ✓ `kv/KvEntry.java:26` (`getKey`/`getDataType`); `AttributeScope.java:26-28` |
| **Attribute Scope** | Trójpodział widoczności atrybutu | (kod) | ✓ `AttributeScope.java:26-28` — `CLIENT_SCOPE(1), SERVER_SCOPE(2), SHARED_SCOPE(3)` |
| **Calculated Field** | Pole wyliczane w czasie rzeczywistym z telemetrii/atrybutów; dominujący temat roku | (kod) | ✓ `cf/CalculatedField.java:50`; typy `cf/CalculatedFieldType.java:24-30`: `SIMPLE, SCRIPT, GEOFENCING, ALARM, PROPAGATION, RELATED_ENTITIES_AGGREGATION, ENTITY_AGGREGATION` |
| **Alarm** | Zdarzenie domenowe wyzwalane warunkiem na encji; cykl życia active/cleared × ack/unack | README:29 „trigger alarms" | ✓ `alarm/Alarm.java:51`; `originator` `:67`, `severity` `:69`; flagi `acknowledged` `:71`, `cleared` `:73` |
| **Alarm Severity** | Poziom istotności | README:29 „Define thresholds" | ✓ `alarm/AlarmSeverity.java:20` — `CRITICAL, MAJOR, MINOR, WARNING, INDETERMINATE` |
| **Alarm Status** | Stan **wyliczany** z (cleared, acknowledged) — nie pole | (kod) | ✓ `alarm/AlarmStatus.java:20`; `Alarm.getStatus()` `:156`, `toStatus()` `:160-164` |
| **Rule Chain / Rule Node** | DAG procesorów komunikatów per tenant; CORE (chmura) lub EDGE | README:54 „processing" | ✓ `rule/RuleChain.java:40`; `type` `:51`, `firstRuleNodeId` `:53`, `root` `:55`; `RuleChainType.java:19` — `CORE, EDGE` |
| **TbMsg / TbMsgType** | Komunikat płynący przez rule engine; typ steruje domyślnym połączeniem | (kod) | ≈ `common/message`/`msg/TbMsgType.java` (60+ typów: POST_TELEMETRY, ALARM_CREATED, ENTITY_UPDATED…) |
| **Entity Relation** | Skierowana, typowana krawędź grafu encji (from→to:type) | README:46 „Define relations" | ✓ `relation/EntityRelation.java:43`; `from` `:55`, `to` `:59`, `type` `:65`; `CONTAINS_TYPE="Contains"` `:49`, `MANAGES_TYPE` `:50` |
| **Entity View** | Fasada eksponująca podzbiór telemetrii/atrybutów Device/Asset w oknie czasowym dla Customera | (kod) | ≈ `EntityView.java` (`entityId`, `keys`, `startTimeMs/endTimeMs`) |
| **OTA Package** | Paczka firmware/software z checksumą, przypięta do profilu | (kod) | ≈ `ota/`, `HasOtaPackage.java`; `Device.firmwareId/softwareId` `Device.java:66-67` ✓ |
| **Edge** | Brama fog/edge uruchamiająca podzbiór reguł (chainy EDGE), routująca do chmury | (kod) | ≈ `edge/Edge.java` (`routingKey`, `secret`, `rootRuleChainId`); `EntityType.EDGE(26)` ✓ |
| **Dashboard / Widget** | Kontener wizualizacji; `WidgetsBundle`/`WidgetType` definicje widgetów | README:54 „dashboards" | ≈ `Dashboard.java`, `widget/WidgetType.java`, `widget/WidgetsBundle.java`; `EntityType.DASHBOARD(4)` ✓ |
| **Notification** | Powiadomienie wielokanałowe (web/email/SMS/Slack/Teams/mobile) | README:30 „notifications via email, SMS, mobile apps" | ≈ `notification/` (Target/Template/Rule/Request); `EntityType.NOTIFICATION*(29-33)` ✓ |
| **AI Model** | Konfiguracja LLM per tenant (młody, multi-provider, langchain4j) | (kod) | ≈ `ai/AiModel.java`; `EntityType.AI_MODEL(43)` ✓; langchain4j w `common/data` (repo-map §4.6) |
| **Queue** | Alias topicu Kafki ze strategią partycji/batcha per tenant | (kod) | ≈ `queue/Queue.java`; `EntityType.QUEUE(28)` ✓ |
| **EDQS** | Replika in-memory danych encji pod entity-queries (Entity Data Query Service), opt-in | (kod) | ✓ default-off: `thingsboard.yml:1990,1997` (os-encji §C4); `common/data/edqs/`, `common/edqs/` |
| **HasVersion** | Optimistic locking — monotoniczny `version` na encji | (kod) | ✓ `HasVersion.java:18-22` (`getVersion`/`setVersion`); `Device.version` `:72` |
| **EntityId (polimorficzny)** | Dyskryminowana unia 40+ `*Id`; wszystkie referencje krzyżowe | (kod) | ✓ `EntityType.java:26-75` (44 typy); `id/TenantId.java`, `id/CustomerId.java` |

---

## KROK 2 — Subdomeny: Core / Supporting / Generic

Kryterium: **Rdzeń = przewaga konkurencyjna i sens produktu** (cztery filary
`README.md:4`: *collection, processing, visualization, device management*),
skrzyżowane z faktyczną intensywnością rozwoju (najgorętszy kod roku wg
`repo-map.md` §2: Calculated Fields backend + alarm-rules + widgety).

| Subdomena | Kategoria | Uzasadnienie (cel produktu / aktywność) |
|---|---|---|
| **Telemetria/Atrybuty (KV)** | **CORE** | Filar #1 „data collection"; bez niezawodnej ingestii KV produkt nie istnieje (README:54). Skalowalność = obietnica marketingowa. |
| **Rule Engine** (`rule-engine-components`) | **CORE** | Filar #2 „processing"; różnicownik vs proste IoT-gateway. Strukturalnie wzorcowy (0 cykli) — celowo pielęgnowany rdzeń. |
| **Calculated Fields** | **CORE** | **Najintensywniej rozwijany kod roku** (repo-map §4.1, „85 zmian/rok" CalculatedFieldCtx). Świeża przewaga: real-time computed properties. |
| **Alarm + Alarm Rules** | **CORE** | Filar przetwarzania → wartość biznesowa (README:29 „thresholds & alarms"); alarm-rules to drugi-najgorętszy obszar UI (Q4 2025). |
| **Device + Device Profile + Credentials** | **CORE** | Filar #4 „device management"; bezpieczne provisioning IoT-entities (README:46) to fundament całej platformy; oś encji = strefa ryzyka #4. |
| **Dashboard / Widget framework** | **Supporting** | Filar #3 „visualization" — istotny, ale realizowany generycznymi wzorcami UI; gigantyczny, lecz to *kanał prezentacji*, nie rdzeniowa logika domenowa. |
| **Entity Relation** | **Supporting** | Spaja encje w graf (README:46), ale jest infrastrukturą modelu, nie celem samym w sobie. |
| **Entity View** | **Supporting** | Kontrola dostępu Customera do podzbioru danych — pochodna device-managementu. |
| **Edge computing** | **Supporting** | Rozszerza rdzeń (reguły) na brzeg sieci; ważne, lecz wtórne wobec chmurowego rdzenia. |
| **OTA Package** | **Supporting** | Operacyjna obsługa floty; uzupełnia device-management. |
| **Notification** | **Supporting** | Kanał wyjścia z alarmów/reguł (README:30); wartość przez integrację, nie unikalny algorytm. |
| **AI/LLM (AiModel)** | **Supporting (emerging)** | Młody, multi-provider, bus-factor 1 (repo-map §4.6); potencjalny przyszły rdzeń, dziś dodatek. |
| **EDQS** | **Supporting** | Świadoma optymalizacja zapytań (replika in-memory); domyślnie wyłączona — wsparcie wydajności, nie domena. |
| **Multi-tenancy / Security / RBAC** | **Generic** | Standard SaaS (Tenant/Customer/User/permission); konieczny, niewyróżniający. |
| **Queue / Kafka transport** | **Generic** | Infrastruktura komunikatów; kontrakt `queue.proto`, wymienialny mechanizm. |
| **Audit log / Version control (Git sync)** | **Generic** | Wymóg enterprise; nie różnicuje produktu. |
| **OAuth2 / Mobile / 2FA / Mail / SMS** | **Generic** | Integracje tożsamości i dostarczania — gotowe wzorce. |
| **Admin settings / Resources / Images / API usage** | **Generic** | Plumbing platformy. |

**Wniosek klasyfikacji:** rdzeń to **pipeline danych** (KV → Rule Engine →
Calculated Fields → Alarm) osadzony na **zarządzaniu urządzeniami** (Device +
Profile + Credentials). To dokładnie tam koncentruje się ryzyko strukturalne
(CF w środku splotu 107 pakietów — repo-map §4.1) i dług osi encji.

---

## KROK 3 — Kandydaci na agregaty i ich niezmienniki

Status egzekwowania: **EGZEKWUJE** (kompilator/DB/runtime wymusza) /
**DEKLARUJE** (model/interfejs mówi, ale brak twardego pilnowania) /
**IGNORUJE** (cichy dryf, gubione bez sygnału).

### Agregat #1 — **Device** (root: Device + DeviceCredentials + atrybuty)

| Niezmiennik (musi być zawsze prawdziwy) | Cytat źródłowy | Status |
|---|---|---|
| Urządzenie żyje wyłącznie w kontekście swojego tenanta (brak widoczności cross-tenant) | odczyt filtrowany `tenant_id` w SQL, tenant nadpisany z JWT (os-encji §1: `DeviceController.java:196`, `BaseController.java:578-585`) | **EGZEKWUJE** (filtr SQL + nadpis JWT) |
| Nazwa unikalna w obrębie tenanta | `device_name_unq_key UNIQUE(tenant_id,name)` (`schema-entities.sql:345`, os-encji §1) | **EGZEKWUJE** — ale przez **constraint DB**, nie walidator (łapane w catch, `DeviceServiceImpl.java:279-281`) |
| `device.type` zawsze równe nazwie Device Profile | rezolucja profilu, `device.type` nadpisany (os-encji §1 krok 8: `DeviceServiceImpl.java:249-266`) | **EGZEKWUJE** w serwisie — choć model deklaruje wolne pole (→ KROK 4, rozjazd #1) |
| `version` monotonicznie rośnie (optimistic lock) | ręczny `version++`+flush+detach (`JpaAbstractDao.java:99-112`, os-encji §1) | **EGZEKWUJE** (ręcznie, obejście Hibernate) |
| Każde urządzenie ma dokładnie jeden zestaw credentials, auto-tworzony przy create w tej samej transakcji | `DeviceServiceImpl.java:226-236` (os-encji §1) | **EGZEKWUJE** (transakcyjnie) |
| **Komplet pól encji przeżywa każdą serializację/kopię** (model→DAO→proto→EDQS→UI) | kanoniczny zestaw `Device.java:49-72`; 7 cichych punktów kopiowania (`Device(Device)` `:82`, `updateDevice` `:97`…) | **IGNORUJE** — silent-drift, ~10 punktów, 3–4 wymuszane (refactor C1) |

### Agregat #2 — **Alarm**

| Niezmiennik | Cytat źródłowy | Status |
|---|---|---|
| `status` jest **funkcją** (cleared, acknowledged), nigdy nie ustawiany wprost | `Alarm.getStatus()` `:156`, `toStatus(cleared,acknowledged)` `:160-164` (zweryfikowano) | **EGZEKWUJE** (pole wyliczane; źródłem prawdy są dwie flagi `:71,:73`) |
| `severity` ustalona przy utworzeniu; dozwolone tylko przejścia stanu ack/clear | `AlarmSeverity.java:20`; flagi/timestampy `Alarm.java:71-85` | **DEKLARUJE** (brak twardego guarda niezmienności severity w modelu) |
| `originator` wskazuje istniejącą encję | `Alarm.originator` (EntityId) `:67` | **DEKLARUJE** |

### Agregat #3 — **RuleChain** (root: RuleChain + RuleNode + connections)

| Niezmiennik | Cytat źródłowy | Status |
|---|---|---|
| Dokładnie jeden **root** chain typu CORE per tenant | `RuleChain.root` `:55`, `type` `:51` (zweryfikowano) | **DEKLARUJE** (pilnowane w serwisie, nie w modelu) |
| `firstRuleNodeId` wskazuje istniejący `RuleNode` | `RuleChain.firstRuleNodeId` `:53` | **DEKLARUJE** |
| `RuleNode.type` to instancjonowalna klasa na classpath | `rule/RuleNode.java` (`type` = FQN) | **EGZEKWUJE** (runtime przy starcie chaina) |

### Agregat #4 — **CalculatedField**

| Niezmiennik | Cytat źródłowy | Status |
|---|---|---|
| Typ CF należy do zamkniętego zbioru 7 wartości | `CalculatedFieldType.java:24-30` + `all` set `:32` (zweryfikowano) | **EGZEKWUJE** (enum) |
| CF jest wersjonowany (optimistic lock) | `CalculatedField.java:50` (`implements HasVersion`) | **EGZEKWUJE** |
| Referencje encji w konfiguracji są ważne | `cf/configuration/CalculatedFieldConfiguration` (repo-map §6.4) | **DEKLARUJE** |

### Agregat #5 — **EntityRelation**

| Niezmiennik | Cytat źródłowy | Status |
|---|---|---|
| Krotka (from, to, type) jest unikalna | `EntityRelation.java:55-65` (zweryfikowano) | **EGZEKWUJE** (PK/constraint DB) |
| `type` niezmienne; grupowane przez `RelationTypeGroup` | `EntityRelation.type` `:65`, stałe `CONTAINS_TYPE` `:49` | **DEKLARUJE** |

### Niezmiennik przekrojowy — **izolacja tenanta**

Każda encja `HasTenantId` jest zawężona do tenanta; `SYS_TENANT_ID` zarezerwowany.
`TenantId.java:36,55` (zweryfikowano). **EGZEKWUJE** na ścieżce odczytu (filtr SQL),
ale rozproszony po ~60 kontrolerach (nadpis JWT powtarzany per endpoint —
`DeviceController.java:196,236`, refactor C3) — **niezmiennik bez jednego strażnika**.

---

## KROK 4 — Rozjazdy MODEL vs KOD

Najcenniejsza część: gdzie wiedza domenowa (model/dokument) mówi X, a kod robi Y.

| # | Dokument/model mówi X | Kod robi Y | Dowód (plik:linia) |
|---|---|---|---|
| 1 | `Device.type` to wolne, walidowane pole typu (`@Length`, `@NoXss`) — atrybut urządzenia | `device.type` jest **zawsze nadpisywany nazwą Device Profile**; użytkownik go nie kontroluje | model: `Device.java:55-56` ✓; zachowanie: `DeviceServiceImpl.java:249-266` (os-encji §1 krok 8) |
| 2 | `AlarmStatus` to 4-wartościowy enum (sugeruje przechowywany stan) | Brak pola `status`; persystowane są **dwie flagi** `acknowledged`/`cleared`, status **wyliczany** | `AlarmStatus.java:20` ✓; `Alarm.java:71,73` + `getStatus()` `:156-164` ✓ |
| 3 | Dodanie jednego pola domenowego = jedna zmiana modelu | Pole trzeba ręcznie skopiować w **~10 punktach** (copy-ctor, JPA, proto, EDQS, EntityKeyMapping, SQL-migracja, lustro TS); 3–4 wymuszane, reszta **cicha** | `Device.java:82,97` ✓; pełna tabela: os-encji §2.1 / refactor C1; realne wpadki: `version` (2 fix-commity), `displayName` (rozjazd EDQS↔SQL) |
| 4 | EDQS to **replika** danych encji (te same wyniki zapytań co SQL) | EDQS **reimplementuje** semantykę pól osobno (`EntityKeyMapping` SQL vs `BaseEntityData`/`FieldsUtil` Java); `DeviceFields` celowo **pomija** `firmwareId/softwareId/externalId/deviceData` — to projekcja-podzbiór, nie wierna kopia | refactor C4: `EntityKeyMapping.java:88` (COALESCE) vs `BaseEntityData.java:149`; pominięcia: `DeviceFields.java:31-57` |
| 5 | Architektura warstwowa: kontroler na szczycie, serwis pod nim | Serwis **importuje kontroler** (2 krawędzie) — inwersja warstw spinająca SCC 107 pakietów | `AbstractBulkImportService.java:60→BaseController`; `AccessValidator.java:63→HttpValidationCallback` (refactor C2, zweryf. ast-grep) |
| 6 | Jedno źródło schematu bazy | **Dwa** ręcznie utrzymywane źródła: `schema-entities.sql` (fresh) vs `schema_update.sql` (delty); współzmiana tylko 47% — okno na zapomnienie migracji | refactor C5: `schema-entities.sql:330-350` vs `application/.../upgrade/{basic,lts}/schema_update.sql` |
| 7 | `EntityType` to spójna, ciągła taksonomia | Dziura w numeracji: `CALCULATED_FIELD_LINK(40)` usunięty w 4.3 — numer proto zarezerwowany, model nosi bliznę po refaktorze | `EntityType.java:66` ✓ (komentarz w kodzie) |
| 8 | `DeviceProfileType` sugeruje wiele rodzajów profili | Istnieje wyłącznie `DEFAULT` — enum-placeholder (model deklaruje wariantowość, której kod nie ma) | `DeviceProfileType.java` (jedna wartość `DEFAULT`) |
| 9 | Pakiet orkiestracji to `service.entity` | Utrwalona literówka `service.entitiy` (56 plików, 72 importerów) — nazwa w kodzie rozjeżdża się z pojęciem | refactor C6: `application/.../service/entitiy/**` |

---

## KROK 5 — Ranking refaktoru (wartość niezmiennika × słabość egzekwowania)

Szereguję kandydatów na agregaty wg **wartości** (jak rdzeniowy jest niezmiennik
dla sensu produktu) i **ryzyka** (jak słabo jest dziś egzekwowany). Spójne
z rankingiem `refactor-opportunities/research.md`, ale przez soczewkę DDD
(niezmienniki, nie tylko struktura).

| # | Agregat / niezmiennik | Wartość (rdzeniowość) | Ryzyko (słabość egzekw.) | Werdykt |
|---|---|---|---|---|
| **1** | **Device — komplet pól przez wszystkie warstwy** | **Najwyższa** — Device to filar #4, a pole = telemetria/konfiguracja rdzenia | **Najwyższe** — `IGNORUJE` (silent-drift, realne wpadki produkcyjne) | **#1 do refaktoru** |
| 2 | Alarm — integralność stanu (cleared/ack) | Wysoka (filar przetwarzania) | Niskie–średnie — status wyliczany `EGZEKWUJE`; severity tylko `DEKLARUJE` | Guard severity/przejść |
| 3 | Izolacja tenanta — jeden strażnik | Najwyższa (bezpieczeństwo SaaS) | Średnie — `EGZEKWUJE`, ale rozproszone po 60 kontrolerach (C3) | Konsolidacja w `BaseController`/checker |
| 4 | RuleChain — spójność DAG | Wysoka (rdzeń processing) | Średnie — `DEKLARUJE` (pilnowane w serwisie) | Niżej — kod czysty (0 cykli) |
| 5 | EntityRelation — unikalność/typ | Średnia (infrastruktura grafu) | Niskie — `EGZEKWUJE` (DB) | Bez pilnej potrzeby |

### #1 — dlaczego Device / silent-drift pól

- **Najbardziej rdzeniowy niezmiennik:** „pole urządzenia nie znika po drodze"
  dotyka bezpośrednio filaru *data collection* i *device management*. Każda encja
  osi (Asset, Customer, EntityView) dziedziczy ten sam wzorzec — naprawa skaluje
  się poza Device.
- **Najsłabiej egzekwowany:** jedyny niezmiennik ze statusem **IGNORUJE**.
  Kompilator pilnuje 3–4 z ~10 punktów; reszta gubi pole **cicho**. Historia
  gita dokumentuje realne wpadki (`version` — 2 fix-commity i typ `int32→int64`;
  `displayName` — rozjazd EDQS↔SQL na produkcji).
- **Najtańsze de-ryzykowanie (guard-first):** szew round-trip **już istnieje** —
  `ProtoUtilsTest.protoSerializationDeserializationEntities()` (`ProtoUtilsTest.java:253`,
  refactor C1) robi `toProto→fromProto→equals` na 8 encjach. Pierwszy krok to
  **czysty dodatek testu** (rozszerzyć round-trip + test parytetu `FieldsUtil`/EDQS),
  zamieniający „cicho gubione" na „czerwone" — zero ryzyka, w pełni odwracalne,
  zanim ruszy jakikolwiek codegen/rejestr deskryptorów pól.
- **Mnożnik ryzyka (przekrojowy):** **brak buildu/testów Javy na PR** (refactor
  §„brak sieci bezpieczeństwa w CI") — to podnosi wartość kandydatów, których
  pierwszy krok da się zabezpieczyć dodaniem testu (czyli #1).

---

## Ograniczenia tej destylacji

- **Brak PRD/wizji formalnej** — Ubiquitous Language wyprowadzony z `README.md` +
  kodu; niektóre intencje biznesowe (np. *czemu* `type` lustruje profil) są
  inferowane z zachowania, nie z deklaracji produktowej.
- **Cytaty zweryfikowane bezpośrednio** dla rdzenia (EntityType, Device, Alarm,
  CalculatedField, RuleChain, EntityRelation, TenantId, DeviceProfile,
  AttributeScope, DeviceCredentialsType, HasVersion). Pozycje ≈ (TbMsgType,
  EntityView, OTA, Edge, Dashboard, Notification, AiModel, Queue) — plik
  potwierdzony przez `EntityType`/listing katalogów, dokładne linie z harvestingu.
- **Twierdzenia o blast-radius, SCC, co-change, silent-drift** przejęte jako
  priory z `context/map/*` i `context/changes/*` (zweryfikowane tam `ast-grep`);
  nie wyprowadzane na nowo.
- Mapa **nie** ocenia jakości kodu ani planów produktowych — destyluje *gdzie żyje
  wiedza domenowa i gdzie kod jej nie odwzorowuje* (KROK 4).
