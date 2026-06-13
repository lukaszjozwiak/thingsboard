---
date: 2026-06-13T05:56:46+02:00
researcher: Claude (Fable 5)
git_commit: 6360aabb4af27e6332af1bdaf3b228d5bfb66076
branch: master
repository: thingsboard
topic: "Refactor opportunities — które problemy z analizy osi encji warto naprawić, w jakim kształcie i kolejności"
tags: [research, refactor-opportunities, tech-debt, entity-axis, edqs, basecontroller, silent-drift, schema, ci, verified]
status: complete
last_updated: 2026-06-13
last_updated_by: Claude (Fable 5)
last_updated_note: "Weryfikacja twierdzeń strukturalnych ast-grep 0.43.0 — 3 korekty (BaseController 1083 nie ~930, schema-entities.sql 982 nie 910, check*Id 103 nie 106) + 3 doprecyzowania (case'y 26, schema_update 27/25, C6 56 plików/72 importerów); reszta potwierdzona. Ranking i werdykty intencjonalności bez zmian."
verification_commit: 6360aabb4af27e6332af1bdaf3b228d5bfb66076
---

# Research: Refactor opportunities (na bazie `os-encji/research.md`)

**Date**: 2026-06-13T05:56:46+02:00
**Researcher**: Claude (Fable 5)
**Git Commit**: 6360aabb4af27e6332af1bdaf3b228d5bfb66076
**Branch**: master
**Repository**: thingsboard

## Research Question

Analiza `context/changes/os-encji/research.md` udokumentowała dług techniczny i ryzyka strukturalne osi encji, ale celowo zostawiła otwarte pytanie: **które** z tych problemów warto naprawić, **w jakim docelowym kształcie** i **w jakiej kolejności**. Ta eksploracja wypisuje każdy zapisany problem, klasyfikuje go (kandydat na refaktor strukturalny vs nie-kandydat), a następnie bada każdego kandydata w trzech wymiarach — obecny kształt, intencjonalność (historia), wykonalność migracji — i kończy rankingiem refactor opportunities z trade-offami. **Żaden refaktor nie został wykonany; żadna decyzja nie zapadła.** Ranking to propozycja dla osobnej sesji planowania.

Dowody zebrane w `os-encji/research.md` i `context/map/*` traktowane jako priory (nie wyprowadzane na nowo). Wszystkie twierdzenia poniżej potwierdzone w bieżącym kodzie (commit `6360aabb4a`) przez cztery równoległe sub-agenty w trybie wyłącznie eksploracyjnym.

---

## Lista kandydatów i klasyfikacja (do audytu)

**KANDYDACI** — naprawa zmieniłaby strukturę kodu:

| ID | Problem (sekcja raportu os-encji) | Czemu strukturalny |
|----|-----------------------------------|--------------------|
| **C1** | Silent-drift: ~10 ręcznych punktów mapowania pola; tylko 3–4 wymuszane kompilatorem (§2.1). 7 punktów cichych: copy-ctor `Device(Device)`, `AbstractDeviceEntity` DTO↔entity, `ProtoUtils` toProto/fromProto, EDQS `DeviceFields`/`FieldsUtil`/`BaseEntityData`, `EntityKeyMapping`, lustro TS | Naprawa = zwinięcie mapowania do jednego źródła / codegen / szew round-trip |
| **C2** | Inwersja warstw serwis→kontroler, **dwie** krawędzie: `AbstractBulkImportService:60→BaseController`, `AccessValidator:63→HttpValidationCallback` (§2.2); kotwica splotu SCC-107 | Naprawa = wyniesienie utility z kontrolera do neutralnego pakietu, rozcięcie cyklu |
| **C3** | `BaseController` god-class: 1083 (raport: ~930) linii, switch po `EntityType`, 54 wstrzykiwane serwisy, logika create/update + ustawianie tenanta przeciekająca do kontrolerów (§2.2, §1.8) | Naprawa = dekompozycja odpowiedzialności / zejście logiki o warstwę niżej |
| **C4** | Duplikacja semantyki zapytań EDQS: `EntityKeyMapping` (SQL) vs `BaseEntityData`/`FieldsUtil` (EDQS) (§2.4) | Naprawa = unifikacja rejestracji pól zapytań za jednym rejestrem |
| **C5** | Podwójne źródło schematu: `schema-entities.sql` vs `schema_update.sql`, brak diffu/generatora (§2.4) | Naprawa = jedno źródło / generowane migracje |
| **C6** | Utrwalona literówka pakietu `service.entitiy` (§2.2) | Naprawa = rename pakietu (trywialnie-strukturalny) |

**NIE-KANDYDACI** — zachowane jako wejście do oceny wykonalności i kosztu, ale nie zmieniają struktury kodu:

- **Luki w testach** — brak szybkiej pętli, `DefaultTbDeviceService` zero testów jednostkowych, dziury per metoda/gałąź, `@Ignore` na teście transakcyjnego delete (§2.3). Brakujące testy, nie struktura — **ale bramkowane przez C1/C3** (testowalność blokowana przez ciche mapowanie i god-class).
- **Przechodni classpath / used-undeclared deps** (§2.4) — higiena deklaracji pom + bramka CI, nie struktura kodu.
- **„Fetch then authorize" przy READ** (§2.2) — kolejność autoryzacji; już złagodzone filtrem `tenant_id` w SQL.
- **Świadome mechaniki** — ręczny optimistic-lock, cache put-on-write, per-tenant lock na create, unikalność nazwy w DB, guardy delete, autoCommit do VC (§1). Zapisane jako znaleziska, nie dług.
- **Huby fan-in** `TenantId`/`EntityId`/`EntityType` — nieodłączny słownik domenowy; brak sensownego celu refaktoru.

---

## Ustalenie przekrojowe (dotyczy WSZYSTKICH kandydatów): brak sieci bezpieczeństwa w CI

To najważniejszy fakt wykonalności, więc na początku. **Na PR nie uruchamia się żaden build ani test Javy.** [evidence]

- Istnieją dokładnie **dwa** workflow GitHub: `.github/workflows/check-configuration-files.yml` (skrypt Pythona walidujący składnię 7 nazwanych plików `*.yml`, odpalany tylko gdy któryś się zmieni) i `license-header-format.yml` (`mvn license:format` + auto-commit nagłówków). Żaden nie kompiluje ani nie testuje modułów Javy. [evidence: oba pliki przeczytane w całości]
- **Brak bramki strukturalnej na backendzie:** zero ArchUnit w jakimkolwiek `pom.xml`; `maven-dependency-plugin` używany do `copy`/pakowania, **nigdy `analyze`**; `maven-enforcer-plugin` pilnuje wersji/istnienia plików, nie architektury; brak jdeps/checkstyle-architektura. [evidence: grep po `**/pom.xml`]
- `ui-ngx/.dependency-cruiser.cjs` (z mapy strukturalnej) jest (a) **tylko frontendowy**, (b) **nieśledzony w gicie** (`?? ui-ngx/.dependency-cruiser.cjs` w git status), (c) **niewpięty w żaden npm-script ani CI**. Nie chroni żadnego kandydata Javy. [evidence]
- Twierdzenie raportu os-encji („brak diffu schematu w CI, brak bramki zależności") — **POTWIERDZONE**.

**Konsekwencja dla rankingu:** każdy refaktor traci automatyczną sieć regresji w chwili wejścia; de-facto bramką jest to, co autor odpali lokalnie (`mvn test`, ~272 testy `@DaoSqlTest`) + review. To podnosi wartość kandydatów, których pierwszy krok da się zabezpieczyć **dodaniem testu** (C1, C5), i obniża pewność przy kandydatach o szerokim blast radius i słabej osłonie (C2, C3). Najwyższej dźwigni prerekwizyt dla CAŁEGO programu — niezależny od kandydatów — to dodanie joba kompilacji+testów backendu na PR. [evidence + inference]

---

## Per kandydat: obecny kształt, intencjonalność, wykonalność

### C1 — Silent-drift: fan-out ręcznego mapowania pól (Device)

**Obecny kształt** [evidence]: kanoniczny zbiór pól `Device` (`Device.java:49-72`) jest ręcznie kopiowany w **7 odrębnych punktach Java** + proto + UI, każdy to osobne, ręcznie pisane ciało:

| # | Punkt | Lokalizacja | Egzekwowanie |
|---|-------|-------------|--------------|
| 1 | copy-ctor `Device(Device)` | `Device.java:82-95` | **ciche** |
| 2 | `updateDevice(Device)` | `Device.java:97-111` | **ciche** |
| 3a–c | DTO→entity / entity→entity / `toDevice()` | `AbstractDeviceEntity.java:86-111`, `:113-126`, `:128-156` | **ciche** |
| 4a–c | `toProto`/`fromProto` + `DeviceProto` | `ProtoUtils.java:809-879`, `queue.proto:264-286` | proto3 tolerancyjny → **ciche** |
| 5a–b | EDQS `DeviceFields` / `FieldsUtil.toFields` | `DeviceFields.java:31-57`, `FieldsUtil.java:130-142` | **ciche** |
| 6 | lustro TS | `device.models.ts:714-725` (+ DeviceInfo `:727-732`) | inny język → **ciche** |

Brak MapStruct, refleksyjnego kopiera, wspólnej bazy mappera. `AbstractDeviceEntity` deduplikuje tylko 3 z 7 punktów (DeviceEntity vs DeviceInfoEntity). `FieldsUtil.toFields(Object)` to centralny dispatch instanceof (17 gałęzi, `:44-82`) — jedyny istniejący szew skupiający konwersje encja→Fields. **Niuans:** EDQS `DeviceFields` celowo pomija `firmwareId/softwareId/externalId/deviceData` — to projekcja zapytaniowa (podzbiór), nie wierna replika; docelowy kształt musi zachować tę intencję. [evidence]

**Werdykt intencjonalności: PRZYPADKOWA ZŁOŻONOŚĆ.** [evidence] Wszystkie 9 commitów z raportu potwierdzone co do treści: `version` rollout `67b8ded9f4` nie dotknął żadnego `.proto` (0 plików), naprawiony tego samego dnia `4b7b69313f` (+`ProtoUtils`), 4 dni później `1dcb64d298` (`int32`→`int64`), UI dogoniło `55e33d7f3d` ~5 tygodni później; `displayName` `3680cbdc03` (tylko SQL) → `e3967e31fc` następnego dnia („Sync edqs and sql logics"); dryf EDQS `506ec363eb`→`b4898568d1` (~2 tygodnie). **Codegen/MapStruct nigdy nie był rozważany** (zero trafień w historii). To dryf z wygody, nie decyzja nośna.

**Wykonalność** [evidence]: **proponowany szew-guard JUŻ ISTNIEJE.** `ProtoUtilsTest.protoSerializationDeserializationEntities()` (`ProtoUtilsTest.java:253-293`) robi `toProto→fromProto→equals` na obiektach losowanych `EasyRandom` dla Device i 7 innych encji — to szybki test (bez Springa/DB) i rozszerzalny wzorzec. ~272 testy `@DaoSqlTest` osłaniają mapowanie persystencji; najsłabiej osłonięta jest ścieżka EDQS (`FieldsUtil`). **Pierwszy krok-prerekwizyt:** rozszerzyć istniejący round-trip ProtoUtils o pozostałe ciche punkty + dodać test parytetu `FieldsUtil` — czysty dodatek testu, zero zmian produkcyjnych, odwracalny; zamienia „cicho gubione" na „czerwone w CI" zanim ruszy jakikolwiek ruch strukturalny.

### C2 — Inwersja warstw serwis→kontroler (dwie krawędzie)

**Obecny kształt** [evidence]: przeszukanie całego drzewa `service/` znalazło **dokładnie dwie** krawędzie `import org.thingsboard.server.controller.` — i żadnej więcej:
- `AbstractBulkImportService.java:60` → `BaseController.toException(t)` (`:233`, `:262`). `toException` to publiczna **statyczna** jednolinijka (`BaseController.java:886-888`) opakowująca `Throwable` w `Exception` — **sprzężenie tylko przez utility statyczne, nie realne**. Podklasy: `DeviceBulkImportService`, `AssetBulkImportService`, `EdgeBulkImportService`.
- `AccessValidator.java:63` → `new HttpValidationCallback(...)` (`:189`). `HttpValidationCallback` to 32-liniowa podklasa `service.security.ValidationCallback<DeferredResult<ResponseEntity>>` bez własnej logiki — **artefakt** odwrotnej zależności: klasa o naturze service-security mieszkająca w pakiecie `controller`.

**Niuans (z agenta obecnego kształtu):** głębszy problem przy krawędzi 2 to to, że `AccessValidator` zwraca `DeferredResult<ResponseEntity>` (typ transportu HTTP) z serwisu bezpieczeństwa (`:166-207`) — pełne naprawienie warstw to przeprojektowanie kontraktu `AccessValidator` na transport-agnostyczny, co jest szersze niż przeniesienie jednej klasy. **Ale rozcięcie krawędzi (cel SCC) tego nie wymaga.**

**Werdykt intencjonalności: PRZYPADKOWA (legacy).** [evidence] Krawędź `AccessValidator` istnieje od `5e2667d3de` (**2018-04-01**, „Implementation of Ts Websocket Service") — poprzedza jakąkolwiek konwencję warstw. Krawędź bulk-import weszła przez przetasowanie modułów `7ec45fd5f1` (2022-05-18), reużywając statyków `BaseController` zamiast wynieść je do util. Żaden commit nie wskazuje intencji odwrócenia warstw.

**Wykonalność** [evidence]: **osłona SŁABA** — brak `AbstractBulkImportServiceTest` i `AccessValidatorTest`; bulk-import osłaniany pośrednio tylko przez `DeviceControllerTest`. Oba ruchy są jednak wychwytywane przez kompilator. Blast radius: 2 krawędzie; rozcięcie rozpuszcza część splotu SCC-107 (dokładnego rozmiaru SCC nie przeliczono ponownie — [unknown]). **Pierwszy krok:** przenieść `HttpValidationCallback` z `controller` → `service.security` (obok rodzica `ValidationCallback`; zmienia się jedna linia importu) i wynieść `BaseController.toException` do wspólnego util. Mechaniczne, odwracalne; przy cienkiej sieci testów robić po jednym i opierać się na lokalnym `mvn test`.

### C3 — `BaseController` god-class

**Obecny kształt** [evidence]: `BaseController.java` ma **1083 (raport: ~930) linii** (weryfikacja `wc -l` potwierdza liczbę z os-encji, nie korektę agenta), **54 `@Autowired`** (zgodne z raportem), switch po `EntityType` w `checkEntityId` (**26 (raport: ~25–28)** case'ów w samym switchu `:633-661`, każdy deleguje do per-encyjnej metody `checkXxxId`; **103 (raport: 106) referencji `check*Id(`**), `default ->` do `entitiesService::findEntityByTenantIdAndId`. Klasa łączy ≥7 odrębnych odpowiedzialności: hub DI (~40 serwisów domenowych), autoryzacja/sprawdzanie istnienia, dispatch EntityType→serwis (switch zamiast polimorfizmu), parsowanie paginacji, dostęp do kontekstu bezpieczeństwa, logowanie audytu/akcji, tłumaczenie wyjątków. Przeciek do kontrolerów potwierdzony: `DeviceController.java:196-201` i powtórka `:236-237` (`setTenantId` + check per endpoint).

**Werdykt intencjonalności: PRZYPADKOWA akrecja; ale switch to świadoma, ŚWIEŻA decyzja.** [evidence] Klasa narastała przez **325 commitów** od commitu fundamentowego `5fdbb51cd7`. Dekompozycja **była próbowana i NIE wycofana**: `377f59400d` (2022, „remove boilerplate permission checks"). Switch po `EntityType` w `checkEntityId` został **wprowadzony/zmodernizowany w 2025** (`d54e7a0e78` „Refactor BaseController.checkEntityId") — to nie stary wzorzec, lecz utrzymywany dispatch. God-class jest przypadkowy (akrecja), ale switch w nim — celowy i aktualny.

**Wykonalność** [evidence]: **najwyższy blast radius** ze wszystkich — ~60 kontrolerów dziedziczy, 54 wstrzykiwane kolaboratory; brak CI. Zachowanie osłonięte tranzytywnie przez integracyjny suite kontrolerów (np. `DeviceControllerTest`), który nie biega na CI. **Istniejący szew:** switch już deleguje do rodziny `checkXxxId` — to realny punkt inkrementalnej dekompozycji; rejestr walidatorów po `EntityType` mógłby zastąpić switch bez ruszania call-site'ów. **Pierwszy krok:** wyodrębnić JEDEN spójny klaster — rodzinę `checkXxxId` — do osobnego kolaboratora (np. `EntityIdChecker` wstrzykiwany do BaseController), delegując z istniejących metod (sygnatury bez zmian, tylko relokacja ciał, niezależnie testowalne). Nowa abstrakcja architektoniczna nie jest potrzebna, by zacząć.

### C4 — Duplikacja semantyki zapytań EDQS (SQL vs in-memory)

**Obecny kształt** [evidence]: dwa niezależne rejestry odpowiadają „które pola są zapytywalne i jak". SQL: `EntityKeyMapping.java` (statyczny blok `:64-196`) — `allowedEntityFieldMap`, `entityFieldColumnMap` (pole→kolumna SQL), aliasy (`title`→`name`), `propertiesFunctions` (np. `displayName` → `COALESCE(NULLIF(TRIM(e.label),''), e.name)` `:88`). EDQS: `BaseEntityData.getField()` (switch po stringach `:134-147`) + `getDisplayName()` reimplementujący regułę displayName **w Javie** (`:149-167`) + `EntityFields.getAsString()` (drugi switch pole→getter `:146-177`). Brak wspólnego kontraktu/interfejsu; jedyne incydentalne współdzielenie to stałe nazw kolumn z `ModelConstants`. Reguła displayName/ownerName zduplikowana jako **dane (SQL)** vs **control-flow (Java)**.

**Flaga przeprojektowania pojęć biznesowych:** zduplikowane reguły pól wyliczanych (`displayName`, `ownerName`, `ownerType`) to **semantyka biznesowa** wyrażona dwukrotnie w dwóch językach. Unifikacja wymaga wspólnej definicji pól wyliczanych, bliżej poziomu pojęć niż C1. **Zgodnie z twardą granicą tej eksploracji — sygnalizuję i zatrzymuję się: realne ryzyko C4 to dryf reguły biznesowej między silnikiem SQL a in-memory, nie struktura tabeli.**

**Werdykt intencjonalności: DECYZJA NOŚNA.** [evidence — mocny] EDQS wprowadzony jako jeden duży PR `86b5378d59` (Klimov, 2025-01-27, „EDQS (#3196)", 358 plików, +18 294/−675, 132 pliki pod `edqs/`, nowy moduł). Treść squasha opisuje budowę silnika in-memory („CSV Loader", „query processors", „in-memory queue", „inmemory/grafana stats"). Dwa modele pól żyją w celowo osobnych modułach. **Duplikacja jest świadomym ograniczeniem; tax dryfu pól (commity „sync edqs and sql") to jej koszt uboczny.**

**Wykonalność** [evidence]: **dryf domyślnie NIE gryzie produkcji** — `queue.edqs.sync.enabled` domyślnie **`false`** (`thingsboard.yml:1990`) i `queue.edqs.api.supported` domyślnie **`false`** (`:1997`). Ścieżka EDQS jest opt-in; domyślną jest SQL `EntityKeyMapping`. To **materialnie obniża pilność C4.** Osłona: `EntityQueryControllerTest`, edqs `RepositoryUtilsTest` (istnieją, integracyjne, poza CI). **Pierwszy krok:** test charakteryzujący parytet zbioru pól (`EntityKeyMapping` kolumny vs `*Fields` builder) dla jednej encji — uwidacznia przyszły dryf, bez próby unifikacji.

### C5 — Podwójne źródło schematu

**Obecny kształt** [evidence]: `schema-entities.sql` (**982 (raport: 910) linii**, pełny autorytatywny `CREATE TABLE ... device` `:330-350`, ładowany przez `SqlEntityDatabaseSchemaService`) vs `application/src/main/data/upgrade/{basic,lts}/schema_update.sql` (**27 i 25 (raport: ~21) linii**, delty `ALTER` per release, ładowane przez `SqlDatabaseUpgradeService` + `SystemPatchApplier`). Trzecie ręczne źródło: zbiór `@Column` w `AbstractDeviceEntity` przy `spring.jpa.hibernate.ddl-auto=none` (brak generacji/walidacji schematu przez Hibernate). **Korekta framingu raportu:** pliki NIE są lustrami — literalny text-diff byłby bez sensu. „47% współzmian" odzwierciedla to, że nowa kolumna musi trafić i do pełnego CREATE, i do delty ALTER (nie że mają być identyczne).

**Werdykt intencjonalności: DECYZJA NOŚNA (konwencja, nieudokumentowana).** [evidence + inference] Flyway/Liquibase **nigdy nie były obecne** (zero w historii i w `pom.xml` — nie dodane i usunięte, lecz nigdy nie wprowadzone). Katalog `upgrade/` ustrukturyzowany per linia wersji (`basic`/`lts`) wskazuje celową konwencję ręcznych delt; brak README ją dokumentującego.

**Wykonalność** [evidence]: brak jakiegokolwiek schema-diffu; jedyny test schematu `EntitiesSchemaSqlTest` sprawdza wyłącznie **idempotencję** `schema-entities.sql` (re-run), nie parytet install vs upgrade. **Najtańsza realna osłona** (sprostowanie Open-Q1 raportu): NIE diff plików, lecz test `@DaoSqlTest` budujący bazę dwiema drogami — (1) świeży `schema-entities.sql`, (2) schemat poprzedniego release + `schema_update.sql` — i porównujący zbiory kolumn z `information_schema`. Infra już istnieje (`EntitiesSchemaSqlTest` ładuje i wykonuje skrypty przez `JdbcTemplate` `:36-56`). Adopcja Flyway/Liquibase jest **NIE-inkrementalna** wobec bespoke `SystemPatchApplier`/`SqlDatabaseUpgradeService`. **Pierwszy krok:** test parytetu install-vs-upgrade rozszerzający infra `EntitiesSchemaSqlTest`.

### C6 — Literówka pakietu `service.entitiy`

**Obecny kształt** [evidence]: pakiet `org.thingsboard.server.service.entitiy` (zauważ „entitiy"); **56 (raport: ~56–62) plików** pod nim, 72 (raport: ~76) importerów (77 (raport: ~82) wystąpień importu), m.in. ~24 kontrolery. Legalny duży podsystem (`Tb*Service`/`DefaultTb*Service`). Nie spleciony z innymi kandydatami.

**Werdykt intencjonalności: PRZYPADKOWA (literówka, która skostniała).** [evidence] Utworzony z literówką `7c7b3eade0` (2022-04-26, „created TbDeviceService"). **Nigdy nie próbowano renamu** (brak commitów `--diff-filter=R`); literówka propagowała się dalej (nawet do komunikatów commitów UI).

**Wykonalność** [evidence]: **najniższe ryzyko z sześciu.** Brak referencji string/refleksja/`@ComponentScan` do pakietu z literówką (grep po `*.properties`/`*.xml`/`*.factories`/`@ComponentScan` — zero; scany używają szerokiego rodzica `org.thingsboard.server`). Rename zostaje wewnątrz już skanowanego rodzica → component-scan Springa nietknięty. Blast radius płytki i **w 100% łapany przez kompilator** (tylko importy). **Pierwszy krok:** atomowy `git mv` katalogu + przepis importów w ~76 plikach w jednym commicie (kompilacyjnie weryfikowalny, trywialnie odwracalny przez `git revert`).

---

## Code References

- `common/data/src/main/java/org/thingsboard/server/common/data/Device.java:49-72,82-95,97-111` — pola + cichy copy-ctor/update (C1)
- `dao/src/main/java/org/thingsboard/server/dao/model/sql/AbstractDeviceEntity.java:86-156` — ręczne mapowania DTO↔entity (C1)
- `common/proto/src/main/java/org/thingsboard/server/common/util/ProtoUtils.java:809-879` + `common/proto/src/main/proto/queue.proto:264-286` — ręczna serializacja (C1)
- `common/proto/src/test/java/.../ProtoUtilsTest.java:253-297` — **istniejący szew round-trip** (guard C1)
- `common/data/.../edqs/fields/DeviceFields.java:31-57`, `FieldsUtil.java:44-82,130-142` — EDQS, projekcja-podzbiór (C1/C4)
- `dao/src/main/java/.../sql/query/EntityKeyMapping.java:64-196` vs `common/edqs/.../data/BaseEntityData.java:134-167` + `EntityFields.java:146-177` — duplikacja semantyki zapytań (C4)
- `application/.../service/sync/ie/importing/csv/AbstractBulkImportService.java:60,233,262` + `BaseController.java:886-888` (`toException`) — inwersja krawędź 1 (C2)
- `application/.../service/security/AccessValidator.java:63,189` + `controller/HttpValidationCallback.java` — inwersja krawędź 2 (C2)
- `application/.../controller/BaseController.java` (1083 (raport: ~930) linii, 54 `@Autowired`, switch `checkEntityId`) + `DeviceController.java:196-201,236-237` — god-class + przeciek (C3)
- `dao/src/main/resources/sql/schema-entities.sql:330-350` vs `application/src/main/data/upgrade/{basic,lts}/schema_update.sql` + `EntitiesSchemaSqlTest.java:36-60` — podwójny schemat (C5)
- `application/src/main/java/org/thingsboard/server/service/entitiy/**` — pakiet z literówką (C6)
- `.github/workflows/check-configuration-files.yml`, `license-header-format.yml` — całość CI (brak buildu/testów Javy)
- `application/src/main/resources/thingsboard.yml:1990,1997` — `edqs.sync.enabled=false`, `edqs.api.supported=false` (C4 default-off)

## Architecture Insights

1. **Asymetria egzekwowania to rdzeń długu osi.** Kompilator pilnuje twardych ogniw (ModelConstants, JPQL na boot, graf Mavena), ale **dodanie pola** — najczęstsza operacja — jest najmniej chronione (C1). Naprawa nie musi zaczynać się od struktury: szew-guard (round-trip/parytet) zamienia „ciche" na „czerwone" przy zerowym ryzyku.
2. **Część długu jest świadoma i nie powinna być „naprawiana" jako struktura.** EDQS (C4) to celowy silnik in-memory; podwójny schemat (C5) to celowa konwencja migracji. Dla obu właściwy ruch to **osłona dryfu (test parytetu)**, nie unifikacja/przepisanie.
3. **Jedna tania krawędź ma nieproporcjonalną dźwignię strukturalną.** C2 (2 krawędzie, ruchy mechaniczne) rozpuszcza część splotu SCC-107 — odpowiednik frontendowego „wytnij `core/utils.ts` z modeli".
4. **Brak CI Javy jest mnożnikiem ryzyka dla każdego ruchu** — i sam w sobie najwyższy quick-win programu (job kompilacji+testów na PR), choć formalnie poza listą kandydatów z raportu.

## Historical Context (from prior changes)

- `context/changes/os-encji/research.md` — źródło wszystkich kandydatów; dowody zweryfikowane i pogłębione (commity, linie, default-off EDQS, framing C5).
- `context/map/repo-map.md` §4 (oś encji), §6 (BaseController jako „czemu serwisy nie powinny go importować") — kotwica C2/C3.
- `context/map/artifact-2-structure.md` §II.2–3 — SCC 107 pakietów, krawędź `AbstractBulkImportService→BaseController`; §II.6 „przeciąć tę krawędź" zgodne z C2.
- `context/map/artifact-1-territory.md` §4 — `schema_update.sql` jako hub (C5); `queue.proto` hub (C1).
- `context/map/artifact-3-contributors.md` §5 — oś danych: dashevchenko, Viacheslav Klimov, Andrii Landiak (Kafka/proto); EDQS autor: Klimov (#3196).

---

## Weryfikacja twierdzeń (ast-grep)

Twierdzenia STRUKTURALNE, na których stoi ranking (liczby metod/pól, „nadpisuje X, ale nie Y", liczność call-site'ów/importów, pary lustrzanych typów), zweryfikowane `ast-grep 0.43.0` na commicie `6360aabb4a`. Każde zero z ast-grep potwierdzone klasycznym grepem (kolumna Metoda notuje, gdzie wzorzec AST nie wystarczył). **Werdykt:** potwierdzone / doprecyzowane / obalone.

| # | Twierdzenie | Werdykt | Dowód (plik:linia) | Metoda (wzorzec/reguła) |
|---|-------------|---------|--------------------|--------------------------|
| V1 | C2: dokładnie **2** krawędzie serwis→kontroler w drzewie `service/` | **potwierdzone** | `AccessValidator.java:63`, `AbstractBulkImportService.java:60` | ast-grep `import org.thingsboard.server.controller.$A;` → 2 |
| V2 | C2: `BaseController.toException` — **2** call-site'y, def. statyczna | **potwierdzone** | użycia `AbstractBulkImportService.java:233,262`; def. `BaseController.java:886` | ast-grep `BaseController.toException($$$)`; grep `static .* toException(` |
| V3 | C2: `HttpValidationCallback` = podklasa `ValidationCallback<DeferredResult<ResponseEntity>>`, 32 linie (typ-alias) | **potwierdzone** | `HttpValidationCallback.java:26` (32 linie) | ast-grep `class HttpValidationCallback extends $P` → **0** (modyfikator `public`+generyki) → grep potwierdza `:26` |
| V4 | C2: **3** podklasy `AbstractBulkImportService` (Device/Asset/Edge) | **potwierdzone** | `AssetBulkImportService.java:42`, `DeviceBulkImportService.java:76`, `EdgeBulkImportService.java:41` | ast-grep `class $C extends AbstractBulkImportService<$$$>` → **0** (modyfikatory) → grep `extends AbstractBulkImportService` → 3 |
| V5 | C3: `BaseController` ~930 linii | **obalone → 1083** | `BaseController.java` (1083) | `wc -l` (poza ast-grep — liczba linii); liczba z os-encji była trafna |
| V6 | C3: **54** `@Autowired` | **potwierdzone** | `BaseController.java:236–395` (54) | ast-grep `@Autowired` → 53 (pominął `@Autowired(required = false)` `:359`) → grep potwierdza 54 |
| V7 | C3: switch po `EntityType` w `checkEntityId` `:633-661` | **potwierdzone** | `BaseController.java:633-661` | ast-grep `switch ($X.getEntityType()) { $$$ }` |
| V8 | C3: liczba case'ów switcha ~25–28 | **doprecyzowane → 26** (w switchu; 28 w całym pliku) | `BaseController.java:633-661` | `sed 633,661` + grep `^\s*case ` |
| V9 | C3: **106** referencji `check*Id(` | **doprecyzowane → 103** | `BaseController.java` | grep `check[A-Za-z]*Id\(` (ast-grep nie matchuje fragmentu identyfikatora) |
| V10 | C3: ~60 kontrolerów `extends BaseController` (57 bezpośr.+3 pośr.) | **potwierdzone** | 57 plików `extends BaseController` | ast-grep `class $C extends BaseController` → **0** (modyfikatory/`implements`) → grep `-rl` → 57; +3 pośrednie per os-encji |
| V11 | C3: przeciek `setTenantId(getCurrentUser().getTenantId())` w kontrolerze `:196,:236` | **potwierdzone** | `DeviceController.java:196,236` | ast-grep `device.setTenantId(getCurrentUser().getTenantId())` → 2 |
| V12 | C1: cichy copy-ctor `Device(Device)` `:82` + `updateDevice` `:97` | **potwierdzone** | `Device.java:82`, `:97` (`updateDevice` zwraca `Device`, nie `void`) | grep deklaracji |
| V13 | C1: `AbstractDeviceEntity` deduplikuje **3** punkty mapowania | **potwierdzone** | `AbstractDeviceEntity.java:86` (DTO→entity), `:113` (copy), `:128` (`toDevice`, `protected`) | grep deklaracji |
| V14 | C1: `ProtoUtils` para lustrzana `toProto(Device)`/`fromProto` | **potwierdzone** | `ProtoUtils.java:809` (toProto), `:852` (fromProto) | grep sygnatur |
| V15 | C1: `FieldsUtil.toFields` = **17** gałęzi `instanceof` | **potwierdzone** | `FieldsUtil.java:44-82` | ast-grep `$X instanceof $T` → 17 = grep 17 |
| V16 | C1: `DeviceFields` nadpisuje label/type/deviceProfileId/additionalInfo, ale **NIE** firmwareId/softwareId/externalId/deviceData („X ale nie Y") | **potwierdzone** | `DeviceFields.java:31-57` | grep tych 4 nazw w pliku → **0** wystąpień (potwierdza pominięcie) |
| V17 | C1: round-trip `ProtoUtilsTest` pokrywa **8** encji (Device + 7) | **potwierdzone** | `ProtoUtilsTest.java:253-293` | grep `assertEqualDeserializedEntity` → 8 (Device, DeviceCredentials, DeviceProfile, Tenant, TenantProfile, TbResource, ApiUsageState, RepositorySettings) |
| V18 | C4: reguła `displayName` jako `COALESCE(...)` (SQL) `:88` | **potwierdzone** | `EntityKeyMapping.java:88` | grep `COALESCE` |
| V19 | C4: para lustrzana — EDQS reimplementuje displayName w Javie (`getField` switch `:135`, `getDisplayName` `:149`, `case "displayName"` `:143`) | **potwierdzone** | `BaseEntityData.java:135,143,149` | grep `switch`/`getDisplayName`/`case "displayName"` |
| V20 | C4: `edqs.sync.enabled=false` `:1990`, `edqs.api.supported=false` `:1997` (default-off) | **potwierdzone** | `thingsboard.yml:1990,1997` | grep `TB_EDQS_SYNC_ENABLED`/`TB_EDQS_API_SUPPORTED` |
| V21 | C5: `schema-entities.sql` 910 linii; tabela `device` `:330-350` | **obalone (linie) → 982**; lokalizacja `:330` potwierdzona | `schema-entities.sql` (982), `CREATE TABLE ... device` `:330` | `wc -l` + grep `CREATE TABLE IF NOT EXISTS device` |
| V22 | C5: `schema_update.sql` ~21 linii każdy | **doprecyzowane → basic 27, lts 25** | `upgrade/basic/...:27`, `upgrade/lts/...:25` | `wc -l` |
| V23 | C6: pakiet `service.entitiy` ~56–62 plików | **doprecyzowane → 56** | `application/.../service/entitiy/**` (56 `.java`) | `find -name '*.java'` |
| V24 | C6: ~76 importerów (~82 wystąpienia) | **doprecyzowane → 72 importerów / 77 wystąpień** | grep importu `service.entitiy` | grep `-rl` (72 plików) / `grep -r` (77 linii) |

**Wpływ na ranking:** żadna korekta NIE zmienia pozycji kandydata. Trzy obalenia/doprecyzowania (V5 1083, V9 103, V21/V22 schemat) dotyczą **rozmiaru/skali**, nie istnienia ani charakteru długu. V5 (BaseController 1083, nie ~930) jeśli już **wzmacnia** tezę o god-class i wysokim blast radius — spójne z pozycją C3 (#3). V21/V22 dotyczą C5, który i tak jest odrzucony z topu (decyzja nośna). Brak adnotacji „do decyzji na etapie planowania" — żaden werdykt strukturalny nie podważa rankingu. Sekcja „Refactor opportunities (ranked)" i werdykty intencjonalności pozostają bez zmian (zgodnie z poleceniem weryfikacji).

---

# Refactor opportunities (ranking — propozycja do sesji planowania)

> Ranking po stosunku **koszt długu / koszt zmiany**, z premią za inkrementalny, odwracalny pierwszy krok przy braku CI. To propozycja, nie decyzja.

## #1 — C1: Zamknąć silent-drift mapowania pól szwem round-trip (guard-first)

- **Obecny → docelowy kształt:** 7 cichych ręcznych punktów mapowania na pole → najpierw **szew-guard** (round-trip `toProto/fromProto/equals` + parytet `FieldsUtil`/EntityKeyMapping) zamieniający „ciche gubienie" w „czerwone w CI"; dopiero potem opcjonalnie codegen/rejestr deskryptorów pól z jednego manifestu (z zachowaniem celowej projekcji-podzbioru EDQS).
- **Czemu #1 (koszt długu vs zmiany):** najwyższy udokumentowany koszt długu — realne wpadki produkcyjne (`version`, `displayName`, dryf EDQS), tax recurujący na każde pole każdej encji. Jednocześnie **najtańsze de-ryzykowanie**: guard JUŻ ISTNIEJE (`ProtoUtilsTest:253`), pierwszy krok to czysty dodatek testu. Intencjonalność: przypadkowa — naprawa nie walczy z decyzją.
- **Blast radius:** pierwszy krok ~zero (tylko test). Pełny codegen szerszy, ale guard-first odcina ryzyko zanim cokolwiek strukturalnego ruszy.
- **Szkic ścieżki:** (1) rozszerzyć round-trip ProtoUtils o brakujące ciche pola; (2) dodać test parytetu `FieldsUtil` (EDQS) ↔ zbiór pól; (3) [dopiero gdy guardy zielone i utrwalone] rozważyć rejestr/codegen mapowań.
- **Pierwszy krok-prerekwizyt:** rozszerzyć `ProtoUtilsTest.protoSerializationDeserializationEntities()` i dodać test parytetu pól EDQS — bez zmian produkcyjnych, odwracalny.

## #2 — C2: Rozciąć inwersję serwis→kontroler (2 krawędzie)

- **Obecny → docelowy kształt:** `AbstractBulkImportService→BaseController.toException` i `AccessValidator→HttpValidationCallback` → `toException` w neutralnym util; `HttpValidationCallback` przeniesiony do `service.security` obok rodzica `ValidationCallback`. Cel: zero importów `controller` w drzewie `service/`.
- **Czemu #2:** **najniższy koszt zmiany przy wysokiej dźwigni strukturalnej** — dwa mechaniczne, kompilacyjnie-weryfikowalne ruchy rozpuszczają część splotu SCC-107. Intencjonalność: przypadkowa (legacy 2018/2022).
- **Blast radius:** 2 krawędzie + importerzy `toException`/`HttpValidationCallback` (łapane przez kompilator). Uwaga: pełna transport-agnostyczność `AccessValidator` (`DeferredResult<ResponseEntity>`) jest szersza — **poza zakresem tego ruchu**, rozcięcie krawędzi jej nie wymaga.
- **Szkic ścieżki:** (1) przenieść `HttpValidationCallback` → `service.security` (1 linia importu); (2) wynieść `toException` do util + przepiąć bulk-import; (3) (opcjonalnie później) bramka strukturalna w CI utrwalająca „service nie importuje controller".
- **Pierwszy krok-prerekwizyt:** przeniesienie `HttpValidationCallback` do `service.security` (najmniejszy, w pełni odwracalny ruch). Bo brak dedykowanych testów — robić po jednym i opierać się na lokalnym `mvn test`.

## #3 — C3: Wyodrębnić rodzinę `checkXxxId` z `BaseController` (pierwszy plasterek)

- **Obecny → docelowy kształt:** god-class (~930 linii, 54 serwisy, switch po EntityType, 7 odpowiedzialności) → wyodrębnić jeden spójny klaster (rodzina `checkXxxId` + dispatch switch) do kolaboratora `EntityIdChecker`; docelowo rejestr walidatorów po `EntityType` zamiast switcha. `BaseController` zostaje przy plumbingu HTTP.
- **Czemu #3 (a nie wyżej):** wysoki koszt długu strukturalnego, ale **najwyższy blast radius** (~60 podklas, 54 kolaboratory) i **najsłabsza osłona przy braku CI** — stosunek dźwigni do ryzyka gorszy niż C1/C2. Akrecja przypadkowa, ale switch świeży/celowy → dekompozycja powinna go zachować jako delegację. Istnieje realny szew inkrementalny (switch już deleguje do `checkXxxId`), więc start nie wymaga nowej abstrakcji.
- **Blast radius:** relokacja ciał metod bez zmiany sygnatur ⇒ wąski przy pierwszym plasterku; rośnie jeśli ruszać hub DI 54 serwisów (NIE w pierwszym kroku).
- **Szkic ścieżki:** (1) wyciąć `checkXxxId` + switch do `EntityIdChecker` wstrzykiwanego do `BaseController`, delegacja z istniejących metod; (2) testy jednostkowe checkera (wreszcie szybka pętla); (3) dopiero potem rozważać rozbicie DI/audytu.
- **Pierwszy krok-prerekwizyt:** ze względu na brak CI — najpierw zabezpieczyć zachowanie autoryzacji testem charakteryzującym (choćby przez istniejące `*ControllerTest`), potem relokować.

---

## Kandydaci rozważeni i odrzuceni (z listy refactor opportunities)

- **C4 (duplikacja EDQS) — ODRZUCONY z topu.** Trzy niezależne powody: (a) **decyzja nośna** (celowy silnik in-memory, PR #3196); (b) **default-off** (`edqs.sync.enabled=false`) — dryf nie gryzie produkcji domyślnie, niska pilność; (c) **realny fix dotyka pojęć biznesowych** (reguły pól wyliczanych w dwóch językach) — poza granicą tej eksploracji. Właściwy ruch: **test parytetu pól** (osłona dryfu), nie unifikacja. Zachować jako wejście do późniejszej, osobnej analizy pojęciowej.
- **C5 (podwójny schemat) — ODRZUCONY jako refaktor strukturalny; ZACHOWANY jako tani guard.** Decyzja nośna (konwencja migracji); adopcja Flyway/Liquibase **nie-inkrementalna**. Ale **test parytetu install-vs-upgrade** (`@DaoSqlTest`, infra już istnieje w `EntitiesSchemaSqlTest`) to świetny tani quick-win — to guard, nie zmiana struktury, więc poza ścisłym rankingiem refaktorów, ale wart zrobienia obok #1 (ta sama filozofia guard-first).
- **C6 (literówka `service.entitiy`) — ODRZUCONY z topu, najniższe ryzyko.** Czysto kosmetyczny; wysoka wartość poznawcza znikoma, dźwignia strukturalna zerowa. Idealny „przy okazji" (atomowy rename, 100% kompilacyjny), ale nie uzasadnia własnego miejsca w rankingu dźwigni.
- **Luki w testach (§2.3) — NIE-KANDYDAT**, lecz bezpośrednio adresowane przez pierwsze kroki #1 i #3 (guardy i szybka pętla to dokładnie to, czego brakuje).
- **Przechodni classpath / used-undeclared (§2.4) — NIE-KANDYDAT** (higiena pom + bramka CI).

## Open Questions (do sesji planowania, nie do tej eksploracji)

1. Czy program refaktorów poprzedzić **dodaniem joba kompilacji+testów backendu na PR**? To mnożnik bezpieczeństwa dla #1–#3, choć formalnie poza listą kandydatów raportu.
2. Czy guardy z #1 i C5 rozszerzyć generycznie na **wszystkie encje `HasVersion`** (Asset, Customer, EntityView), skoro wzorce są kopiowane między `*ServiceImpl` (Open-Q2 raportu os-encji)?
3. C4: jeśli kiedyś `edqs.sync.enabled` ma być domyślnie włączone na produkcji — to przesuwa C4 z „latentny" do „pilny" i wymaga osobnej analizy pojęciowej reguł pól wyliczanych.
4. [unknown] Czy istnieją ciche punkty mapowania poza 7 potwierdzonymi (edge sync, import/export DTO) — agent obecnego kształtu nie wyczerpał enumeracji konsumentów pól `Device`.
