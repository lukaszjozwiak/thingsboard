# Mapa strukturalna — jak to jest zbudowane (dependency-cruiser + jdeps/Maven)

> Dokument w dwóch częściach: **Część I — frontend** (dependency-cruiser,
> `ui-ngx/src/app`) i **Część II — backend Java** (jdeps + maven-dependency-plugin,
> moduły `common/*`, `dao`, `rule-engine`, `application`).

# Część I — Frontend (dependency-cruiser)

> Narzędzie: dependency-cruiser **17.4.3** na `ui-ngx/src/app` (frontend Angular).
> Przeanalizowano **1881 modułów / 15 966 zależności** (1676 plików w `src/app`,
> po wykluczeniu `*.spec.ts`). Konfiguracja: `ui-ngx/.dependency-cruiser.cjs`.
>
> **Adaptacja szablonu:** prompty serii odnosiły się do ścieżek z innego repo
> (`webapp`, `channels/src/*`, `platform/client`, `platform/types`). Mapowanie na
> ten projekt: `platform/types` → `ui-ngx/src/app/shared/models`,
> `platform/client` → `ui-ngx/src/app/core` (http/api/serwisy),
> `channels/src` → `ui-ngx/src/app/modules/home`. Gorące obszary wzięte
> z `artifact-1-territory.md`: `shared/components`, `alarm-rules`,
> `calculated-fields`, `widget/lib/settings`, `shared/models`, `filter`, `rule-node`.

## 0. Konfiguracja narzędzia

- Instalacja: `npm install --no-save --legacy-peer-deps dependency-cruiser typescript @viz-js/viz`
  w `ui-ngx/` (bez modyfikacji `package.json`; `--legacy-peer-deps` konieczne
  przez konflikty peer-deps Angulara). Brak natywnego Graphviz w systemie —
  render SVG przez `@viz-js/viz` (Graphviz WASM).
- Konfig `ui-ngx/.dependency-cruiser.cjs`: `tsConfig` (aliasy `@app/@core/@shared/@home`),
  `tsPreCompilationDeps: true`, cache, reguły: `no-circular`,
  `models-are-foundation`, `core-below-modules`, `shared-not-from-modules`.
- Przebieg: `npx depcruise src/app -T json -f $TEMP/depcruise-full.json`,
  potem analizy przez `depcruise-fmt` i skrypty Node na JSON-ie.

## 0b. Top 3 pomysły na eksplorację legacy z dependency-cruiser

1. **Walidacja cykli i granic warstw** (`-T err` / `err-html` z regułami
   `no-circular` + custom): natychmiast pokazuje, gdzie repo „gryzie własny ogon"
   i które warstwy są fikcją. Raporty: `err`, `err-long`, `err-html`, `baseline`
   (do mrożenia długu i pilnowania, żeby nie rósł).
2. **Metryki stabilności i sprzężeń** (`--metrics -T metrics`): fan-in/fan-out
   i indeks niestabilności per folder — wskazuje moduły „rdzeniowe" (wysoki
   fan-in → każda zmiana ryzykowna) i wrażliwe na zmiany. Uzupełnienie:
   `--reaches <plik>` („kogo zepsuję zmieniając X") i `--affected <rev>`
   (zasięg zmian z brancha/PR-a).
3. **Wizualizacja wycinków** (`-T dot/ddot/archi/mermaid` + `--focus`,
   `--include-only`, `--collapse`): nie cały graf, tylko podgraf odpowiadający
   na jedno pytanie (np. „jak wygląda pętla spinająca warstwy"). `archi`/`ddot`
   dają widok folderowy dla rozmów architektonicznych, `err-html` — interaktywny
   raport długu.

## 1. Cykle w aktywnych obszarach

### Najważniejsze obserwacje

1. **Frontend ma jedną gigantyczną silnie spójną składową: 265 modułów** (z 1676),
   w której każdy plik zależy (pośrednio) od każdego. Skład: 169 plików
   `shared/models`, 54 `core`, 26 `modules/home`, 11 `shared/components`.
   To nie są lokalne cykle — to jeden splot na poziomie całej aplikacji.
2. **Łącznie 762 unikatowe cykle** (1002 naruszenia `no-circular`); 171 to pętle
   2-elementowe (komponent ↔ table-config — wzorzec lokalny, niegroźny), ale
   ~180 cykli ma długość 20–40 — to ślady przejścia przez wielki splot.
3. **Wszystkie zbadane krawędzie cykli to realne importy runtime** (`import`),
   nie type-only — czyli splot istnieje też w bundlu i w testach, nie tylko
   w kompilatorze.
4. Gorące obszary z mapy terytorium różnią się diametralnie: **`alarm-rules`
   i `calculated-fields` (nowy kod) mają po 3–5 małych, lokalnych cykli**, a
   **`shared/components` (314) i `shared/models` (647)** są wrośnięte w splot.
   `rule-node` — zero cykli.
5. Pliki najczęściej występujące w cyklach to huby całego frontendu:
   `core/utils.ts` (490 cykli), `shared/models/widget.models.ts` (465),
   `core/core.state.ts` (357), `shared/models/telemetry/telemetry.models.ts` (370).

### Tabela

| Obszar | Co znalazłem | Dowód z dependency-cruiser | Dlaczego to ważne przy zmianie | Związek z artifact-1 | Co sprawdzić dalej |
|---|---|---|---|---|---|
| `shared/models` | **647 cykli**; 169 plików modeli siedzi w 265-modułowym splocie | np. `shared/models/widget.models.ts → core/utils.ts → …` (pętla 2-el.); SCC z analizy JSON | „Czyste modele" wcale nie są czyste — zmiana modelu może odbić się echem po core i home; import jednego modelu wciąga ~315 modułów | Najgorętszy obszar wspólny (`common/data` frontendu — sprzęga się ze wszystkim, sekcja 3 artifact-1) | Czy da się wyciąć `core/utils.ts` z modeli (to główna krawędź zwrotna) |
| `shared/components` | **314 cykli**, w tym zwrotne importy z `modules/home` | `shared/components/popover.component.ts → modules/home/models/widget-component.models.ts` | Komponent „wspólny" zależy od warstwy wyżej — refaktor home'a może zepsuć shared, odwrotnie niż sugeruje struktura katalogów | #2 katalog hands-on (572 dotknięcia) | Ile komponentów shared importuje z `@home/*` (23 naruszenia `shared-not-from-modules`) |
| `widget/lib/settings` | 107 cykli, głównie wzorzec `*-settings.component ↔ *-panel.component` + wejścia do splotu przez `widget.models.ts` | `color-settings.component.ts ↔ color-settings-panel.component.ts` | Pary settings↔panel są lokalne (łatwe do przerwania), ale każda zmiana ustawień widgetów dotyka `widget.models.ts` = wejście do splotu | Q1 2026: push na widgety mapowe i settings | Czy pary settings/panel da się rozdzielić przez wydzielenie współdzielonych typów |
| `alarm-rules` | Tylko **3 cykle**, wszystkie 2-elementowe wokół `alarm-rules-table-config.ts` | `alarm-rules.component.ts ↔ alarm-rules-table-config.ts` | Nowy kod (Q4 2025) jest strukturalnie czysty — cykle to lokalny wzorzec table-config, łatwy do utrzymania w ryzach | Nowa funkcjonalność z Q4 2025 (286 dotknięć) | Pilnować, żeby nie wrósł w splot — reguła baseline |
| `calculated-fields` | **5 cykli**, lokalne, wokół `calculated-fields-table-config.ts` | `calculated-field.component.ts ↔ calculated-fields-table-config.ts` | Jak wyżej — dominujący temat roku jest po stronie UI zdrowy | Temat #1 całego roku (CF) | jw.; oraz `calculated-field-form.service.ts` w core importujący z home (inwersja!) |
| `filter` | 4 cykle, ale o długości **21–30** — filter-dialog jest nicią wielkiego splotu | pętla 21-el.: `query.models → alarm.models → core/utils → widget.models → … → filter-dialog.component → query.models` | Zmiana w filtrach może wymagać zrozumienia łańcucha przez pół aplikacji; trudna do przewidzenia propagacja | 172 dotknięcia, aktywny w Q2 2026 | Render podgrafu pętli (sekcja 5) |
| `rule-node` | **0 cykli** | brak naruszeń `no-circular` | Obszar bezpieczny strukturalnie | 189+156 dotknięć (action/external) | — |

## 2. Granice warstw

### Najważniejsze obserwacje

1. **Wszystkie trzy granice są naruszone** — łącznie 86 naruszeń:
   `models-are-foundation` 51, `shared-not-from-modules` 23,
   `core-below-modules` 12. Warstwy istnieją „kulturowo", ale graf ich nie
   potwierdza.
2. **Głównym wektorem inwersji jest `core/utils.ts`**: fundamentowe modele
   (`alarm`, `device`, `base-data`, `alias`, `audit-log`, `ace`…) importują
   helpery z `core` — to pojedyncza krawędź, która wciąga całą warstwę modeli
   do splotu.
3. **`core` importuje z `modules/home`** w 12 miejscach — m.in.
   `core/http/widget.service.ts → @home/models/widget-component.models.ts`
   i `core/services/calculated-field-form.service.ts → @home/components/calculated-fields/…`.
   Plik `modules/home/models/widget-component.models.ts` ma **fan-in 156** —
   de facto pełni rolę warstwy wspólnej, mieszkając w feature-module.
4. **Modele importują UI**: `shared/models/alarm.models.ts →
   modules/home/components/widget/lib/table-widget.models.ts`,
   `device.models.ts → …/lwm2m/lwm2m-profile-config.models.ts` — typy
   konfiguracji widgetów/profili mieszkają przy komponentach, a modele po nie
   sięgają w górę.
5. Dobra wiadomość: naruszenia są **skoncentrowane w kilku hubach** —
   `core/utils.ts`, `widget-component.models.ts`, `modules-map.models.ts`,
   `table-widget.models.ts`, `lwm2m-profile-config.models.ts`. Naprawa ~5 plików
   (przeniesienie typów w dół / wydzielenie utils bez zależności) usunęłaby
   większość inwersji.

### Tabela

| Sprawdzana granica | Wynik | Dowód z dependency-cruiser | Dlaczego to ważne przy zmianie | Związek z artifact-1 | Co sprawdzić dalej |
|---|---|---|---|---|---|
| `shared/models` jako fundament (nie importuje core/UI) | **NARUSZONA — 51 błędów** | `error models-are-foundation: shared/models/alarm.models.ts → core/utils.ts`; `→ modules/home/.../table-widget.models.ts` | Modele to frontendowy odpowiednik `common/data` — najszerszy promień rażenia; przez `core/utils.ts` każdy model ciągnie ~315 modułów | `common/data` = #3 moduł aktywności; `shared/models` 213 dotknięć | Czy `core/utils.ts` da się rozbić na czyste helpery (bez importów) + resztę |
| `core` poniżej `modules` (brak importów z `@modules`) | **NARUSZONA — 12 błędów** | `error core-below-modules: core/http/widget.service.ts → modules/home/models/widget-component.models.ts` | Serwis HTTP (klient API) zależy od typów feature-modułu — zmiana w home może zepsuć warstwę klienta, czego nikt się nie spodziewa | Kontrolery REST + `core/http` aktywne we wszystkich kwartałach | Przenieść `widget-component.models.ts` (fan-in 156!) do `shared` |
| `shared` nie importuje z `modules` | **NARUSZONA — 23 błędy** | `error shared-not-from-modules: shared/components/popover.component.ts → modules/home/models/widget-component.models.ts` | „Wspólny" komponent nie jest wspólny — nie da się go użyć/testować bez feature-modułów | `shared/components` = #2 katalog hands-on (572) | Które z 23 krawędzi to znowu `widget-component.models.ts` / `modules-map.models.ts` |
| Aliasowanie ścieżek (przewidywalność importów) | OK | tsconfig paths (`@core`, `@shared`, `@home`) poprawnie rozwiązywane (`aliased-tsconfig-paths`) | Importy są spójne składniowo — problem jest w kierunku, nie w formie | — | — |

**Interpretacja dla często zmienianych miejsc:** nowe obszary (`alarm-rules`,
`calculated-fields`, `rule-node`) korzystają z warstw przewidywalnie — wyjątek:
`core/services/calculated-field-form.service.ts` sięgający do `@home`. Stare
huby (`widget.models.ts`, `core/utils.ts`, `core.state.ts`) i wszystko wokół
widgetów może „zaskoczyć przy zmianie": modyfikacja modelu potrafi przejść
przez core do komponentów dashboardu i z powrotem.

## 3. Ryzyka testowalności

### Podsumowanie

W gorących obszarach głównym wrogiem testów jednostkowych jest 265-modułowy
splot: każdy plik, który dotyka `widget.models.ts`, `core/utils.ts` albo
`core.state.ts` (NgRx), ma domknięcie tranzytywne ~315 modułów. W praktyce
izolowany unit test bez TestBed/mocków jest możliwy tylko dla czystych funkcji;
komponenty tabelowe (`*-table-config`) ciągną >500 modułów.

### Lista ryzyk testowych

| Miejsce | Domknięcie tranzyt. | Ryzyko | Rekomendowany poziom testu |
|---|---:|---|---|
| `alarm-rules-table-config.ts` | **531** (25 importów bezpośr.) | Konfiguracja tabeli ciągnie dialogi, serwisy, store | Test integracyjny (TestBed + mock http); unit tylko dla czystych helperów |
| `calculated-fields-table-config.ts` | **525** | jw. + serwis w `core` zależny od tego obszaru (pętla przez warstwy) | jw. |
| `filter-dialog.component.ts` | 315 (tylko 6 importów bezpośr.!) | 6 importów wystarcza, by wpaść w splot — mockowanie „po bożemu" niewykonalne | Integracyjny; logika filtrów → wydzielić do czystych funkcji |
| `widget/lib/settings/*` (np. `color-settings`) | ~318 | Wejście do splotu przez `widget.models.ts`; pary settings↔panel | Komponentowy z płytkim renderem; e2e dla edytora dashboardu |
| `shared/components/entity/*` | ~316 | Autocomplete'y sprzężone ze store (NgRx `core.state.ts`, fan-in 622) | Integracyjny z mock store |
| `shared/models/*.models.ts` | ~315 (przez `core/utils.ts`) | Nawet „model" nie jest czysty — import modelu w teście wciąga pół aplikacji | Po przecięciu krawędzi `models → core/utils` wróci do unit |
| Zmiany w dashboard/widget framework | n/d | `widget-component.models.ts` (fan-in 156), `widget.models.ts` (356), `core.state.ts` (622), `core/utils.ts` (541) — zmiana huba dotyka setek modułów | Regresja realnie tylko e2e (dashboard, widget rendering) |

### Najbardziej podejrzane moduły

1. `core/utils.ts` — fan-in **541**, uczestnik 490 cykli; worek helperów
   importowany przez modele = główna przyczyna splotu.
2. `core/core.state.ts` — fan-in **622**; globalny stan NgRx wciągany nawet
   przez `shared/components` (przez `toggle-header.component.ts`).
3. `shared/models/widget.models.ts` — fan-in **356**, 465 cykli; centralny
   typ widgetów spina modele z komponentami.
4. `modules/home/models/widget-component.models.ts` — fan-in **156**, mieszka
   w feature-module, a importują go `core` i `shared` (inwersja warstw).
5. `shared/components/toggle-header.component.ts` — komponent UI w 300 cyklach,
   bo importowany przez… `shared/models/time/time.models.ts` (model → komponent!).

### Co sprawdzić dalej

- Czy krawędzie `models → core/utils` używają tylko kilku funkcji (np.
  `isDefined`, `deepClone`) — jeśli tak, wydzielenie `shared/util/pure.ts`
  przecina splot jednym ruchem.
- Czemu `time.models.ts` importuje `toggle-header.component.ts` (prawdopodobnie
  pojedynczy typ — kandydat na przeniesienie do models).
- `--metrics -T metrics` dla folderów `core`, `shared/models`, `home/components`
  (indeks niestabilności przed/po ewentualnym refaktorze).
- Baseline (`depcruise-baseline`) — zamrozić 1085 naruszeń i pilnować w CI,
  żeby nowy kod (alarm-rules, CF — dziś czysty) nie wrastał w splot.

### Opcjonalny kolejny krok: graf

Wyrenderowany (sekcja 4). Kolejne kandydatury: `--reaches` dla
`widget.models.ts` (kogo psuje zmiana), `--focus` na `core/utils.ts`
z `--focus-depth 1`, `archi` dla widoku folderowego całego `src/app`.

## 4. Render podgrafu (Graphviz)

**Pytanie, na które odpowiada graf:** *jak wygląda pętla, która odwraca warstwy
i spina `shared/models → core → modules/home → shared` w jeden splot?*

- Zakres: 21 plików kanonicznego cyklu (m.in. `query.models.ts`,
  `core/utils.ts`, `widget.models.ts`, `core.state.ts`,
  `widget-component.models.ts`, `dashboard-page.models.ts`,
  `widget-config.component.ts`, `filter-dialog.component.ts`).
- Metoda: `depcruise-fmt -T dot --include-only '<21 plików>'` na pełnym JSON-ie,
  render `@viz-js/viz` (Graphviz WASM, brak natywnego `dot` w systemie).
- Plik: **`context/map/artifact-2-core-knot.svg`** (krawędzie czerwone =
  uczestniczą w cyklu).

## 5. Fakty do dalszej syntezy

- 1881 modułów, 15 966 zależności; 1085 naruszeń reguł (1002 cykle, 83 warstwy).
- 762 unikatowe cykle; największa SCC = **265 modułów** (169 models, 54 core,
  26 home, 11 shared/components); kolejne SCC: 18, 12, 7, 4 — przepaść.
- Krawędzie splotu to importy runtime (nie type-only).
- Nowy kod (alarm-rules, calculated-fields, rule-node) jest strukturalnie
  czysty; dług koncentruje się w warstwie widget/dashboard + `core/utils.ts`.

---

# Część II — Backend Java (jdeps + maven-dependency-plugin)

> Narzędzia: **jdeps 25.0.3** (JDK 25) na skompilowanych klasach + **Maven 3.9.15**
> z `maven-dependency-plugin:3.9.0:analyze`. Zakres: gorące moduły backendu wg
> `artifact-1-territory.md` — `application` (1298 klas), `common/data` (1150),
> `dao` (762), `rule-engine/rule-engine-components` (284), `common/queue` (171)
> — plus graf wszystkich 60 modułów Mavena z pom.xml.

## 0. Konfiguracja narzędzi (Java)

- Kompilacja: `mvn -T1C compile -pl "application,!ui-ngx" -am -DskipTests -Dlicense.skip=true`
  (wykluczenie `ui-ngx` z reaktora — frontend budowałby się przez yarn kilkanaście
  minut; scope `runtime` pozwala na to przy fazie `compile`).
- jdeps: `jdeps -verbose:package <moduł>/target/classes` (graf pakietów) i
  `-verbose:class` (graf klas); SCC liczone Tarjanem skryptem Node na sparsowanym
  wyjściu. Uwaga na format: jdeps wyrównuje kolumny wieloma spacjami (`\s+->\s+`).
- `dependency:analyze`: domyślny plugin **3.7.0 nie czyta bajtkodu JDK 25**
  (major 69) — trzeba jawnie `org.apache.maven.plugins:maven-dependency-plugin:3.9.0:analyze`.
  Dla `application` konieczny stub `ui-ngx-4.4.0-SNAPSHOT.jar` w lokalnym `.m2`
  (`install:install-file` z pustym jarem), bo goal forkuje pełną rezolucję
  zależności łącznie z runtime.
- Zastrzeżenie: klasy skompilowane lokalnie JDK 25 (produkcyjnie projekt celuje
  w starszy bajtkod) — na analizę struktury zależności nie ma to wpływu.

## 1. Graf modułów Mavena — warstwy „na papierze" istnieją

Maven wymusza acykliczność na poziomie modułów, więc tu cykli nie ma z definicji.
Kierunki i sprzężenia (z 60 pom.xml, tylko zależności `org.thingsboard*`):

| Metryka | Top | Interpretacja |
|---|---|---|
| Fan-in (ile modułów zależy) | `data` **20**, `util` 14, `message` 14, `queue` 13, `transport-api` 8 | `common/data` to słownik repo — potwierdza wprost tezę z artifact-1 („sprzęga się ze wszystkim") |
| Fan-out wewnętrzny | `application` **23**, `black-box-tests` 12, `edqs` 8, `dao` 8 | `application` konsumuje praktycznie każdy moduł — każda zmiana gdziekolwiek może go dotknąć |

Warstwy modułowe są zdrowe: `data` → (`util`,`message`,`stats`) → (`proto`,`cache`,
`cluster-api`,`discovery-api`) → (`queue`,`dao-api`) → (`dao`,`rule-engine-api`,
`transport-api`) → (`rule-engine-components`, transporty) → `application`.
Ciekawostka: `common/data` (fundament) zależy od **langchain4j-core** — zewnętrzne
SDK AI weszło do słownika danych repo (ślad tematu AI z Q3 2025, artifact-1).

Render: **`context/map/artifact-2-java-modules.svg`** (24 węzły, 96 krawędzi,
transporty mqtt/coap/http/lwm2m/snmp zwinięte do `transport-impl`; kolory = warstwy).

## 2. Cykle pakietowe wewnątrz modułów (jdeps + Tarjan)

Choć moduły są acykliczne, **wewnątrz** modułów żyją sploty pakietów — ta sama
choroba co na froncie, tylko piętro niżej:

| Moduł | Pakiety / krawędzie | Największa SCC | Co siedzi w splocie | Dowód (przykładowa krawędź) |
|---|---|---|---|---|
| `application` | 393 / 2962 | **107 pakietów** | aktory (`actors.*`, w tym `calculatedField`), **wszystkie kontrolery**, `service.cf.*`, `service.edge.*`, `service.entitiy.*`, install | `service.sync.ie.importing.csv.AbstractBulkImportService → controller.BaseController` — **jedyna** krawędź wciągająca kontrolery do splotu |
| `dao` | 191 / 1153 | **46 pakietów** | `dao.entity`, `dao.model.sql`, wszystkie dao encji, `dao.service.validator` | `dao.entity → dao.{alarm,cf,edge,…}` i z powrotem (AbstractEntityService); walidatory (fan-out 67) ↔ serwisy dao |
| `common/data` | 91 / 268 | **33 pakiety** | pakiet bazowy `common.data`, `id`, `alarm`, `query`, `kv`, `relation`, `widget`… | dwukierunkowo `data ↔ data.id`: każde `*Id` importuje `EntityType` z pakietu bazowego, a encje bazowe importują swoje `*Id` |
| `common/queue` | 41 / 129 | 9 pakietów | `queue`, `discovery`, `kafka`, `memory`, `task` | `queue.discovery` ↔ `queue` |
| `rule-engine-components` | 118 / 370 | **0 — zero cykli** | — | nody zależą tylko w dół: `rule.engine.api`, `common.data`, `common.msg` |

Mniejsze SCC w `common/data` pokrywają się z gorącymi tematami roku:
`alarm.rule.condition*` (3 pakiety) i `cf ↔ cf.configuration` (2) — nowe
obszary mają lokalne, kontrolowane pętle, jak ich odpowiedniki w UI.

### Symetria z frontendem

| Zjawisko | Frontend (Część I) | Backend Java |
|---|---|---|
| Jeden wielki splot na warstwę | SCC 265 modułów w `src/app` | SCC 107 pakietów w `application`, 46 w `dao`, 33 w `common/data` |
| Pojedyncza krawędź odwracająca warstwy | `shared/models → core/utils.ts` | `AbstractBulkImportService → BaseController` (serwis → kontroler) |
| „Czysty słownik", który czysty nie jest | `widget.models.ts` (fan-in 356) | pakiet bazowy `common.data` ↔ `id` (fan-in pakietu: 55/50) |
| Obszar wzorcowo czysty | `rule-node` (0 cykli) | `rule-engine-components` (0 cykli) |
| Nowy kod | `alarm-rules`/`calculated-fields` czyste | **odwrotnie**: CF backendowe siedzi w środku splotu — `actors.calculatedField ↔ service.cf` dwukierunkowo |

Ta ostatnia różnica jest istotna: najgorętsze pliki roku wg artifact-1
(`CalculatedFieldCtx`, `CalculatedFieldEntityMessageProcessor`,
`CalculatedFieldManagerMessageProcessor`) leżą dokładnie na dwukierunkowej
krawędzi aktorzy↔serwisy — aktor importuje stany/serwisy CF
(`…EntityMessageProcessor → service.cf.ctx.state.CalculatedFieldCtx`), a serwisy
importują typy aktorów z powrotem (`DefaultCalculatedFieldProcessingService →
actors.calculatedField.CalculatedFieldTelemetryMsg`, `MultipleTbCallback`,
`ActorSystemContext`).

## 3. Granice warstw (Java)

| Sprawdzana granica | Wynik | Dowód | Dlaczego ważne przy zmianie |
|---|---|---|---|
| Kontrolery na szczycie (nikt nie importuje `controller`) | **NARUSZONA — 1 krawędź** | `AbstractBulkImportService$1/$2 → controller.BaseController` | Przez tę jedną krawędź każdy kontroler (a jest ich ~80, fan-out pakietu 187) jest osiągalny z warstwy serwisów i odwrotnie — splot 107 pakietów |
| Aktory ↔ serwisy jednostronnie | **NARUSZONA — dwukierunkowa** | `actors.calculatedField ↔ service.cf*` (7+ krawędzi klasowych w każdą stronę) | Najgorętszy kod roku (CF) nie ma szwu do testowania/wydzielenia; zmiana stanu CF dotyka aktora i odwrotnie |
| `common/data` jako fundament bez zależności | OK na poziomie modułów (tylko `protobuf-dynamic`, `langchain4j-core`) | pom.xml + jdeps (brak `org.thingsboard.*` spoza `common.data`) | Lepiej niż frontendowe `shared/models` (które importuje `core/utils`); ale wewnętrznie pakiet bazowy = god-package (fan-in 55) |
| `dao` nie zna `application` | OK | moduł `dao` nie zależy od `application` (Maven to wymusza) | Pionowy konwój zmian app+data+dao z artifact-1 to sprzężenie *współzmian*, nie importów — łatwiejsze do utrzymania |

## 4. dependency:analyze — dług deklaracji zależności

`maven-dependency-plugin:3.9.0:analyze` na 4 gorących modułach:

| Moduł | Used-undeclared | Unused-declared | Najciekawsze |
|---|---:|---:|---|
| `common/data` | 9 | 7 | Fundament **używa Springa nie deklarując go** (`spring-core` przez transitive); jackson, guava, gson, protobuf też niezadeklarowane; `antisamy`/`logback` zadeklarowane a nieużyte w bajtkodzie |
| `dao` | 27 | 22 | Cały stos persystencji (`spring-data-jpa`, `hibernate-core`, `spring-jdbc`, `HikariCP`) używany **przez przypadek transitive** ze startera; `rule-engine-api` zadeklarowane, w bajtkodzie nieużywane |
| `rule-engine-components` | 25 | 4 | `org.thingsboard:{data,dao-api,cluster-api,proto}` używane bezpośrednio, ale dziedziczone jako `provided` z innych deklaracji; `langchain4j-core` niezadeklarowany mimo własnego noda AI |
| `application` | **73** | 6 | M.in. `org.thingsboard:{data,message,dao-api,proto,cache}` używane wprost bez deklaracji; „nieużywane" `coap`/`snmp`/`remote-js-client` to **fałszywe pozytywy** — wpinane runtime'owo przez component-scan Springa (bajtkod tego nie widzi) |

**Interpretacja:** wersje i obecność połowy classpathu zależą od przechodnich
deklaracji innych modułów. Podbicie wersji w jednym pom potrafi zmienić zachowanie
modułu, który danej biblioteki nawet nie deklaruje — to samo ryzyko „zaskoczenia
przy zmianie", które na froncie tworzy splot importów. Uwaga: wyników
unused-declared nie czyścić mechanicznie (sterowniki JDBC, transporty
component-scan, natywne netty to runtime).

## 5. Ryzyka testowalności (Java)

Domknięcie tranzytywne klas `org.thingsboard.*` (graf klasowy `application`;
liczby to **dolne ograniczenie** — liście z innych modułów nieekspandowane):

| Klasa (gorąca wg artifact-1) | Domknięcie | Rozbicie | Rekomendowany poziom testu |
|---|---:|---|---|
| `CalculatedFieldManagerMessageProcessor` | **391 klas** | 145 `common/data`, 101 `application`, 49 `dao`, 33 `message`, 21 `rule-engine-api`, 14 `queue` | Tylko integracyjny (jak istniejący `CalculatedFieldIntegrationTest`); unit wyłącznie dla czystych fragmentów stanów |
| `CalculatedFieldEntityMessageProcessor` | 338 | jw. | jw. |
| `CalculatedFieldCtx` | 331 | jw. | jw. — „rdzeń CF" (85 zmian/rok) jest nietestowalny w izolacji |
| `DeviceController` | 277 | przez `BaseController` (209) | Testy kontrolerów w repo słusznie są integracyjne (`application/src/test/.../controller` — 269 dotknięć wg artifact-1) |
| nody `rule-engine-components` | małe | 0 cykli, zależą tylko od `api`/`data`/`msg` | **Unit z mockiem `TbContext`** — strukturalnie najzdrowszy kod backendu |

Huby klasowe (fan-in w `application`): `TenantId` 489, `EntityId` 345,
`TbCoreComponent` 216, `EntityType` 157, `SecurityUser` 132, `JacksonUtil` 132,
`ThingsboardException` 132 — zmiana któregokolwiek to zmiana globalna.

## 6. Co sprawdzić dalej (Java)

- **Przeciąć `AbstractBulkImportService → BaseController`** — jedna krawędź,
  która scala kontrolery z serwisami w SCC 107; wydzielenie używanych helperów
  `BaseController` do serwisu/komponentu rozcina splot jednym ruchem
  (odpowiednik frontendowego „wytnij `core/utils.ts` z modeli").
- **Szew aktorzy↔CF**: typy wiadomości (`CalculatedFieldTelemetryMsg`,
  `MultipleTbCallback`) przenieść do pakietu neutralnego (`common/message`
  lub `service.cf.msg`), żeby `service.cf` nie importował `actors.*`.
- **`dependency:analyze` w CI** z progiem na used-undeclared dla
  `org.thingsboard:*` (zacząć od zamrożenia obecnego stanu, jak baseline
  dependency-cruisera na froncie).
- Pilnować małych SCC nowego kodu w `common/data` (`alarm.rule.condition*`,
  `cf ↔ cf.configuration`), żeby nie wrosły w 33-pakietowy splot bazowy.
- Opcjonalnie: jdeps z pełnym classpathem między-modułowym (zamiast liści
  „not found") dla dokładnych domknięć cross-module.

## 7. Fakty do dalszej syntezy (Java)

- 60 modułów Mavena; fan-in: `data` 20, `util`/`message` 14, `queue` 13;
  fan-out: `application` 23. Moduły acykliczne (wymusza Maven), graf w
  `artifact-2-java-modules.svg`.
- SCC pakietów: `application` **107**/393, `dao` **46**/191, `common/data`
  **33**/91, `queue` 9/41, `rule-engine-components` **0**/118.
- Pojedyncze krawędzie spinające: `AbstractBulkImportService → BaseController`;
  `actors.calculatedField ↔ service.cf` (dwukierunkowa).
- Najgorętszy kod roku (CF backend) siedzi w środku splotu — odwrotnie niż
  jego frontendowy odpowiednik; `rule-engine-components` czyste po obu stronach.
- Dług deklaracji: 134 used-undeclared łącznie w 4 modułach (w tym wewnętrzne
  `org.thingsboard:*`), classpath w dużej mierze przechodni.

---
*Część I: 2026-06-10, dependency-cruiser 17.4.3, `ui-ngx/src/app`, konfig `ui-ngx/.dependency-cruiser.cjs`.*
*Część II: 2026-06-11, jdeps 25.0.3 + maven-dependency-plugin 3.9.0 na `target/classes` (kompilacja `mvn compile -pl "application,!ui-ngx" -am`).*
