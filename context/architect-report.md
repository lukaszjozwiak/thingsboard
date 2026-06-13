---
title: Raport architektoniczny — Moduł 4 (10xArchitect)
created: 2026-06-13
type: architect-summary
sources: [context/map/repo-map.md, context/changes/os-encji/research.md, context/changes/refactor-opportunities/plan.md, context/domain/01-domain-distillation.md, context/domain/02-invariant-aggregate-refactor.md, context/domain/03-anti-corruption-layer.md]
---

# Raport architektoniczny — ThingsBoard (Moduł 4)

> Synteza wyłącznie z artefaktów L2–L5. Każde twierdzenie strukturalne pochodzi z
> wskazanego artefaktu (nie z pamięci o kodzie). Braki oznaczono **BRAK artefaktu**.

## 1. Opisane projekty

Wszystkie artefakty L2–L5 powstały na **jednym** repozytorium: **ThingsBoard**
(commity robocze `5f6f69dd` → `b7abed0664`, branch `master`).

| Repo | Stack | Skala (orientacyjnie, z L2) | Artefakty |
|---|---|---|---|
| **ThingsBoard** | Monolit Java/Spring (`application`) + Angular (`ui-ngx`) + moduły wspólne (`common/*`, `dao`, `rule-engine`, `transport/*`); Kafka (`queue.proto`) + PostgreSQL | 60 modułów Maven; `ui-ngx` ~1676 plików TS; `application` ~1298 klas; okno analizy gita 12 mies. (2323 commity) | L2, L3, L4, L5 |

Tylko jeden projekt — nie ma rozjazdu między repozytoriami między wejściami.

## 2. Mapa projektu (z L2 — `repo-map.md`)

1. **Jeden splot zależności na warstwę po obu stronach stacku:** silnie spójna
   składowa 265 modułów w `ui-ngx` (762 cykle) i SCC 107 pakietów w `application`
   (kontrolery + aktory + serwisy), 46 w `dao`, 33 w `common/data` *(graf importów + jdeps)*.
2. **Splot spina pojedyncza krawędź inwersji warstw:** `shared/models → core/utils.ts`
   na froncie, `AbstractBulkImportService → BaseController` na backendzie.
3. **Strefa ryzyka #1 — Calculated Fields backend:** najgorętszy kod roku siedzi
   *w środku* splotu (aktorzy ↔ serwisy CF dwukierunkowo, domknięcie ~391 klas),
   a dwoje głównych autorów nie commituje od Q1 2026 — ryzyko strukturalne × ryzyko wiedzy.
4. **Centra lokalne / huby:** `common/data` (fan-in 20 modułów; `TenantId` fan-in 489),
   `widget.models.ts` (fan-in 356), `BaseController`, `core/utils.ts` — pojedyncze
   pliki, przez które przechodzą setki zależności.
5. **Najważniejszy unknown:** jdeps objął tylko 5 gorących modułów Javy; `msa/*`,
   implementacje transportów, `edqs`, docker — bez grafu, sprzężenia wnioskowane
   wyłącznie ze współzmian gita (bywa mylące). Mapa nie mówi nic o jakości kodu,
   pokryciu testami ani planach produktowych.

## 3. Analiza ficzera (z L3 — `os-encji/research.md`)

**Co i dlaczego:** zbadano **oś encji** (kanoniczny CRUD encji domenowej,
reprezentant **Device**) — bo L2 wskazała ją jako **strefę ryzyka #4** (konwój
`common/data`+`dao`+`application`, 115 wspólnych commitów, przechodni classpath).

**Feature overview:** input wchodzi z `ui-ngx` przez REST (`POST /api/device` →
`DeviceController`); stan zmienia się w pięciowarstwowym konwoju
`Controller → Tb*Service → *ServiceImpl (dao) → Jpa*Dao → PostgreSQL`; side-effecty
(cache, Kafka/klaster, edge, EDQS, audyt, version control) rozchodzą się **wyłącznie
eventami Springa po commicie** (3 listenery `SaveEntityEvent`); wraca zapisany
`Device` (z `version`, znormalizowanym `type`) jako JSON.

**Technical debt (3 najważniejsze ryzyka):**
- **Silent-drift mapowania pól (blast radius):** dodanie jednego pola do encji wymaga
  **~10 ręcznych punktów mapowania**, z których kompilator wymusza tylko 3–4; reszta
  (copy-ctor, JPA, `ProtoUtils`/proto, EDQS, `EntityKeyMapping`, `schema_update.sql`,
  lustro TS) zawodzi **cicho**. Historia gita to potwierdza: `version` (2024) →
  2 fix-commity (zapomniane proto, potem zły typ `int32→int64`), `displayName` (2025)
  → rozjazd SQL↔EDQS. **Potwierdzone ast-grepem** (T6: 42 call-site'y
  `SaveEntityEvent.builder()`, 100% w `dao`; T3: 21× `@Query`).
- **Luka testowa o profilu bezpieczeństwa:** `findDevicesByQuery`/`getDevicesByIds`
  (logika per-device `checkPermission`) — **zero pokrycia**; `DefaultTbDeviceService`
  — 0 testów bezpośrednich (T9, ast-grep). Regresja = wyciek urządzeń między customerami.
- **Kruche sprzężenie / inwersja warstw:** ast-grep **skorygował** L2 — krawędzie
  serwis→kontroler są **dwie**, nie jedna (`AbstractBulkImportService:60` +
  `AccessValidator:63→HttpValidationCallback`), a kontrolerów jest **60**, nie „~80".

## 4. Plan refaktoryzacji (z L4 — `refactor-opportunities/plan.md`)

**Co refaktoryzowane:** **C1 — silent-drift mapowania pól Device**, i tylko jego
krok **guard-first**. Docelowy kształt: dwa najsłabiej chronione punkty (proto
round-trip, projekcja EDQS `FieldsUtil`) stają się **czerwonymi testami sterowanymi
refleksją** — nowe, niezmapowane pole `Device` wywala test automatycznie, bez edycji
testu. Świadome pominięcia EDQS kodowane jako jawny, komentowany allowlist.

**Czego świadomie NIE robimy:** brak codegenu/MapStruct/rejestru deskryptorów; nie
zwijamy 7 punktów mapowania w jedno źródło; brak guardów dla copy-ctor/`updateDevice`/
`AbstractDeviceEntity` i lustra TS; adopcja CI nie jest blokerem faz 1–2.

**Fazy + weryfikacja:**
- **Faza 1 — guardy Device (proto + EDQS) + triage czerwieni** → auto (`mvn -pl
  common/proto,common/data ... test`) + ręcznie (usunięcie mapowania → czerwony test).
- **Faza 2 — uogólnienie wzorca na pozostałe encje HasVersion/EDQS** → auto
  (parametryzowane `*ParityTest`) + ręcznie (spot-check 2–3 allowlist).
- **Faza 3 (zalecana, nieblokująca) — CI: backend compile + `@DaoSqlTest`** → auto
  (workflow na PR) + ręcznie (PR psujący guard → czerwony check).

## 5. Domena wg DDD (z L5 — `01`/`02`/`03`)

**Ubiquitous language (z `01-domain-distillation.md`):** **Tenant** (korzeń izolacji
wielodzierżawnej, `SYS_TENANT_ID`), **Device** (+ Profile + Credentials — filar
„device management"), **Telemetry/Attribute (KV)**, **Alarm** (status **wyliczany**
z flag, nie pole), **Calculated Field** (najgorętszy temat roku). `EntityType` to
44 typy encji. Najważniejsze rozjazdy model-vs-kod: (#1) `Device.type` deklarowany
jako wolne pole, a **zawsze nadpisywany** nazwą Device Profile; (#2) `AlarmStatus`
sugeruje stan przechowywany, a jest **funkcją** (cleared, acknowledged); (#3) jedno
pole domenowe = ~10 ręcznych punktów kopiowania.

**Niezmiennik #1 i agregat:** L5 ma dwa, świadomie rozbieżne wybory #1:
- `01` (oś wartość × silent-drift) → **Device / „komplet pól przeżywa każdą
  serializację"** (status **IGNORUJE**) — agregat **Device**.
- `02` (oś agregat-strażnik z metodami i preconditions) → **I2: monotoniczny cykl
  życia Alarmu** (`ack` tylko gdy `!acknowledged`, `clear` tylko gdy `!cleared`) —
  agregat **Alarm**. Świadomy rozjazd uzasadniony: deliverable `02` to agregat z
  metodami domenowymi, a najsilniejsza maszyna stanów w domenie to Alarm, nie Device.

**Anti-Corruption Layer (z `03`):** przecieka **langchain4j** (fat-SDK AI, własny
fork `1.16.1-TB1`) — przez **5 warstw** (model `common/data` → kontrakt
`rule-engine-api` → nody `rule-engine-components` → serwis `application/service/ai`
→ REST `controller`), 16 plików produkcyjnych. Najgroźniej: `langchain4j-core` w
**compile scope w `common/data`** wciąga SDK na classpath **20 modułów**, które z AI
nie mają nic wspólnego. Kod *deklaruje* wymienialność (enum `AiProvider` z 9 dostawcami,
warstwa `Tb*`), ale jej nie dotrzymuje; ACL zwija 3 tory konwersji w jeden adapter
`service/ai/langchain4j/**` i domyka regułą ArchUnit.

## 6. Decyzje, które należą do mnie

- **Wybór ficzera (L3):** AI podpowiedziało strefy ryzyka z mapy; ja rozstrzygnąłem,
  że oś encji (#4, Device) daje najlepszą dźwignię nauki — kanoniczny wzorzec
  kopiowany na wszystkie encje, więc analiza skaluje się poza jeden przypadek.
- **Zakres refaktoru (L4):** AI wskazało 6 kandydatów C1–C6; ja zdecydowałem
  ograniczyć plan do **guard-first kroku C1** (czysty dodatek testów, near-zero
  ryzyko, odwracalny) zamiast strukturalnego codegenu — bo **brak CI Javy** czyni
  każdą zmianę produkcyjną nieubezpieczoną.
- **Rozjazd #1 w L5:** to **moja** świadoma decyzja, że `02` wybierze Alarm zamiast
  Device — Device pasuje do osi „silent-drift/serializacja", ale nie do wzorca
  „agregat-strażnik z preconditions"; rozbieżność udokumentowałem, nie ukryłem.
- **Wariant zachowawczy I4 (L5/`02`):** wobec **BRAK PRD** sam rozstrzygnąłem, że
  severity jest zamrożona po CLEARED (inferencja z `merge()`), oznaczając to jako
  „do potwierdzenia z produktem".
- **Granica ACL (L5/`03`):** AI zaproponowało pakiet *lub* osobny moduł Maven; ja
  zostawiłem decyzję pakiet-vs-moduł jako otwartą dla sesji planowania, wybierając
  pakiet jako minimalny pierwszy ruch.

---
*Ograniczenia: BRAK formalnego PRD/`tech-stack.md` (L5, KROK 0) — Ubiquitous Language
wyprowadzony z README + kodu. Twierdzenia o SCC/fan-in/silent-drift przejęte jako
priory z L2 (jdeps/dependency-cruiser) z ich ograniczeniami; twierdzenia osi encji
zweryfikowane ast-grepem (L3).*
