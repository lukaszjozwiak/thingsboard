---
title: Refaktor agregatu-strażnika — niezmiennik cyklu życia Alarmu (ThingsBoard)
created: 2026-06-13
type: refactor-plan
---

# Agregat-strażnik dla rdzeniowego niezmiennika: **cykl życia Alarmu**

> **Produkt tej pracy to PLAN refaktoru, nie kod.** Nie modyfikuję kodu
> produkcyjnego. Wszystkie cytaty `plik:linia` zweryfikowane bezpośrednim
> otwarciem plików (branch `master`, commit roboczy `b7abed0664`).
> Krok po kroku: odkrycie → identyfikacja → klasyfikacja → diagnoza → projekt.

---

## KROK 0 — Odkryty kontekst

- **Brak formalnego PRD / `tech-stack.md`** (potwierdzone w `context/foundation/`
  — tylko `README.md` szablonu). Wizja produktu wyłącznie z `README.md` korzenia:
  *„data collection, processing, **visualization**, and device management"* i
  *„define thresholds and **trigger alarms**"* (README:4, README:29). Alarm jest
  punktem, w którym przetwarzanie danych zamienia się w **akcję operatora** —
  rdzeń wartości biznesowej, nie pomocniczy plumbing.
- **Prior:** `context/domain/01-domain-distillation.md` (destylacja domeny). Ten
  dokument świadomie **rozjeżdża się z rankingiem #1 destylacji** (patrz ramka
  „Dlaczego nie Device" niżej) — bo deliverable to *agregat-strażnik* (metody
  domenowe z preconditions, nazwany błąd, klient→serwer), a nie refaktor
  codegenu pól.
- **Stack i warstwy logiki biznesowej** (gdzie żyje reguła Alarmu):

  | Warstwa | Plik(i) | Rola wobec niezmiennika |
  |---|---|---|
  | **Model domenowy** | `common/data/.../alarm/Alarm.java` | Anemiczny `@Data`/`@Builder` — *zero* egzekwowania |
  | **REST** | `application/.../controller/AlarmController.java` | 6 endpointów mutujących; mapowanie błędów rozjechane (TODO w kodzie) |
  | **Orkiestracja** | `application/.../service/entitiy/alarm/DefaultTbAlarmService.java` | Rekonstrukcja przejść z różnicy flag; audyt; połykanie błędów komentarza |
  | **Persystencja + reguła** | `dao/.../alarm/BaseAlarmService.java` (+ `AlarmDao`) | Realne egzekwowanie — w SQL, przez flagi wyniku |
  | **Reguły (2. ścieżka zapisu)** | `rule-engine-components/.../action/TbCreateAlarmNode.java`, `TbClearAlarmNode.java` | Pipeline tworzy/czyści alarmy z pominięciem REST |
  | **UI (strażnik kliencki)** | `ui-ngx/.../components/alarm/alarm-table-config.ts` | Filtruje, które alarmy *wolno* ack/clear — logika maszyny stanów w przeglądarce |

---

## KROK 1 — Zidentyfikowane niezmienniki biznesowe Alarmu

Reguły, które w tej domenie **muszą** być zawsze prawdziwe (z kodu + README):

| # | Niezmiennik (zawsze prawdziwy) | Źródło (plik:linia) |
|---|---|---|
| I1 | **`status` jest czystą funkcją `(cleared, acknowledged)`** — nigdy nie jest osobno przechowywany ani ustawiany wprost | `Alarm.java:154-166` (`getStatus()`→`toStatus(cleared,acknowledged)`) |
| I2 | **Cykl życia jest monotoniczny**: `ack` tylko gdy `!acknowledged`; `clear` tylko gdy `!cleared`; powtórzenie to operacja nielegalna, nie no-op | `DefaultTbAlarmService.java:118` („already acknowledged"), `:140` („already cleared") |
| I3 | **Każde przejście jest atomowe** wraz ze swoimi skutkami ubocznymi (komentarz systemowy, event do rule chain, rekordy propagacji) | `DefaultTbAlarmService.java:113-116, 135-138`; `BaseAlarmService.java:237-253` |
| I4 | **`severity` i `originator` są ustalane przy utworzeniu**; na CLEARED alarmie severity jest zamrożona (eskalacja możliwa tylko na ACTIVE) | model: `Alarm.java:66-69`; deklaracja w `01-domain-distillation.md` KROK 3 (Agregat #2) |
| I5 | **`customerId` alarmu = customer originatora** (spójność własności) | `BaseAlarmService.java:126-132` |
| I6 | **Propagacja alarmu do encji-rodziców jest częścią tej samej transakcji co zapis** (brak alarmu „widocznego u siebie, niewidocznego u rodzica") | `BaseAlarmService.java:237-253, 449-456` |

---

## KROK 2 — Klasyfikacja i wybór #1

Trzy osie oceny (a) rdzeniowość, (b) rozsmarowanie po warstwach, (c) realne
egzekwowanie:

| # | Rdzeniowość (a) | Rozsmarowanie (b) | Egzekwowanie (c) |
|---|---|---|---|
| **I2 — monotoniczny cykl życia ack/clear** | **Najwyższa** — to *sens* alarmu jako bytu operacyjnego (README:29) | **Najszersze** — UI + generic-save + dedykowane REST + DAO/SQL + rule engine = **5 miejsc** | **Najsłabsze jako niezmiennik domenowy**: model `IGNORUJE`, UI jest *strażnikiem*, serwer egzekwuje pośrednio przez flagę DB, błąd kodowany źle (TODO) |
| I1 — status wyliczany | Wysoka | Wąskie (1 metoda) | `EGZEKWUJE` (czysta funkcja) — zdrowe, zostawiamy |
| I4 — severity/originator immutable | Wysoka | Średnie | `DEKLARUJE` + **naruszane** (`merge()` nadpisuje severity) |
| I3/I6 — atomowość + propagacja | Wysoka | Średnie | Częściowo `IGNORUJE` — błędy *połykane* (catch-log) |
| I5 — customerId == originator.customer | Średnia | Wąskie | `EGZEKWUJE` (rzuca w create) |

### ✅ Wybór #1 — **I2: monotoniczny cykl życia Alarmu** (rdzeń agregatu, wraz z I1/I3/I4/I6, które naturalnie pod niego podpadają)

To **jednocześnie najbardziej rdzeniowy I najsłabiej egzekwowany jako reguła
domenowa**: alarm bez wiarygodnego cyklu życia (ktoś „odznacza" ack, severity
zmienia się po wyczyszczeniu, ack „przechodzi" dwa razy w bulku) to alarm, któremu
operator nie może ufać — a zaufanie do alarmów jest całą wartością filaru
*processing → thresholds & alarms*.

> #### Dlaczego nie Device (rozjazd z `01-domain-distillation.md` #1)
> Destylacja słusznie wskazała *kompletność pól Device przez warstwy* jako #1 w osi
> **wartość × silent-drift**. Ale to niezmiennik **serializacji/codegenu**
> (round-trip `toProto→fromProto`, parytet EDQS↔SQL) — jego naprawa to rejestr
> deskryptorów pól i testy round-trip, **nie** agregat z metodami i preconditions.
> Deliverable tego kroku to *agregat-strażnik* (metoda domenowa → precondition →
> nazwany błąd → klient traci rolę strażnika). Najlepszym dopasowaniem do tego
> wzorca jest **maszyna stanów**, a najsilniejsza maszyna stanów w tej domenie to
> Alarm. Stąd świadomy wybór I2.

---

## KROK 3 — Diagnoza: gdzie dziś żyje reguła I2 (i gdzie przecieka)

### 3.1 Model domenowy nie egzekwuje **niczego** (rdzeń problemu)

`Alarm` to anemiczny worek danych: `@Data` + `@Builder` + `@AllArgsConstructor`
generują **publiczny setter dla każdej flagi**.

```text
Alarm.java:46   @Data
Alarm.java:48   @Builder
Alarm.java:49   @AllArgsConstructor
Alarm.java:71   private boolean acknowledged;   // publiczny setAcknowledged(boolean)
Alarm.java:73   private boolean cleared;         // publiczny setCleared(boolean)  → setCleared(false) LEGALNE w pamięci
Alarm.java:69   private AlarmSeverity severity;  // publiczny setSeverity(...)      → mutowalne po utworzeniu
```

Jedyny zdrowy fragment to **I1**: `getStatus()` liczy status z flag i nie ma
settera (`Alarm.java:154-166`). Reszta cyklu życia jest niepilnowana w modelu.

### 3.2 Strażnikiem przejść jest **klient (UI)** — logika maszyny stanów w przeglądarce

```text
alarm-table-config.ts:290   const unacknowledgedAlarms = alarms.filter(alarm => !alarm.acknowledged);
alarm-table-config.ts:325   const activeAlarms      = alarms.filter(alarm => !alarm.cleared);
alarm-table-config.ts:199-200  allowAcknowledgment / allowClear  // gating na uprawnieniu, nie na stanie serwera
```

To **precondition I2 wykonywany w kliencie**: przeglądarka decyduje, które alarmy
*wolno* ack/clear, i tylko dla nich wysyła żądania. Serwer jest tu drugim, nie
pierwszym strażnikiem.

### 3.3 Serwer egzekwuje I2 **pośrednio** — przez flagę z bazy, nie przez domenę

Realna reguła „już ack / już cleared" jest rozstrzygana w SQL (`AlarmDao`), zwracana
jako flaga w `AlarmApiCallResult`, a `DefaultTbAlarmService` dopiero **interpretuje
flagę** i rzuca wyjątek:

```text
AlarmApiCallResult.java:26-36   @Data, pola: modified, cleared, created...   (wynik DB, nie decyzja domeny)
DefaultTbAlarmService.java:113  if (result.isModified()) {...} else
DefaultTbAlarmService.java:118     throw new ThingsboardException("Alarm was already acknowledged!", BAD_REQUEST_PARAMS);
DefaultTbAlarmService.java:135  if (result.isCleared())  {...} else
DefaultTbAlarmService.java:140     throw new ThingsboardException("Alarm was already cleared!", BAD_REQUEST_PARAMS);
```

Enforcement niezmiennika domenowego **mieszka w warstwie persystencji**. Domena nie
wie, że przejście było nielegalne — dowiaduje się o tym z `boolean` z DAO.

### 3.4 Ta sama reguła rekonstruowana po raz trzeci — **różnicowanie flag** w generic-save

Generyczny `POST /api/alarm` przyjmuje **cały obiekt `Alarm`** i serwis zgaduje
intencję użytkownika, porównując pożądany stan z aktualnym:

```text
AlarmController.java:139-148    saveAlarm(@RequestBody Alarm alarm)   // pełny stan, nie komenda
DefaultTbAlarmService.java:77   if (alarm.isAcknowledged() && !resultAlarm.isAcknowledged()) resultAlarm = ack(...);
DefaultTbAlarmService.java:80   if (alarm.isCleared()     && !resultAlarm.isCleared())     resultAlarm = clear(...);
DefaultTbAlarmService.java:83-89  // analogiczne różnicowanie assign/unassign
```

Ten sam niezmiennik I2 jest więc kodowany **trzy razy** (UI filter + flag-diff +
dedykowane endpointy ack/clear), za każdym razem inną mechaniką.

### 3.5 Niezmiennik I4 (severity immutable) jest **aktywnie naruszany** w `merge()`

```text
BaseAlarmService.java:397  private Alarm merge(Alarm existing, Alarm alarm) {
BaseAlarmService.java:413     existing.setAcknowledged(alarm.isAcknowledged());   // może cofnąć ack
BaseAlarmService.java:414     existing.setCleared(alarm.isCleared());             // może cofnąć clear
BaseAlarmService.java:415     existing.setSeverity(alarm.getSeverity());          // nadpisuje severity bez warunku !cleared
```

Brak guarda `!cleared` przy zmianie severity i brak monotoniczności flag w samej
operacji merge — to luka, przez którą stan może się cofnąć.

### 3.6 Błędy I3/I6 są **połykane** zamiast zatrzymywać operację (łamie fail-fast)

```text
BaseAlarmService.java:449-456   createEntityAlarmRecord(...) { try {...} catch (Exception e) { log.warn(...); } }
DefaultTbAlarmService.java:244-248  addSystemAlarmComment(...) { try {...} catch (ThingsboardException e) { log.error(...); } }
```

Rekord propagacji (I6) lub komentarz systemowy (ślad audytowy I3) mogą **cicho
zniknąć**, a alarm i tak „przejdzie". To dokładnie wzorzec *log-i-jedź-dalej*,
który ten refaktor ma wyeliminować.

### 3.7 Mapowanie błędu na odpowiedź jest **przyznanym długiem** (TODO w kodzie)

```text
AlarmController.java:173   //TODO: return correct error code if the alarm is not found or already cleared
AlarmController.java:188   //TODO: return correct error code if the alarm is not found or already cleared
```

Nielegalne przejście wraca dziś jako `BAD_REQUEST_PARAMS` zamiast `409 CONFLICT`;
zespół to wie i zostawił marker.

### 3.8 Druga ścieżka zapisu (rule engine) omija REST

```text
TbCreateAlarmNode.java   // pipeline tworzy/aktualizuje aktywny alarm
TbClearAlarmNode.java     // pipeline czyści alarm
```

Niezmiennik musi obowiązywać identycznie z REST **i** z rule engine — co dziś jest
możliwe tylko dlatego, że obie ścieżki schodzą do tego samego `AlarmService` w DAO.
Każdy strażnik założony *wyżej* (kontroler/UI) tej ścieżki nie chroni — co jest
kolejnym argumentem za umieszczeniem reguły w **agregacie/domenie**, nie w API.

**Wniosek diagnozy:** I2 (z I1/I3/I4/I6) jest *deklarowany* w wielu miejscach,
*egzekwowany* tylko pośrednio w SQL, *pre-walidowany* w przeglądarce i miejscami
*naruszalny* (`merge`) lub *cicho gubiony* (`catch-log`). Klasyczny kandydat na
agregat-strażnik.

---

## KROK 4 — Projekt agregatu-strażnika

Cel: **jedno miejsce** egzekwujące cykl życia. Domena rzuca **nazwany błąd
domenowy**; klient przestaje być strażnikiem; persystencja staje się głupim
repozytorium ładującym/zapisującym cały agregat w jednej transakcji.

### 4.1 Root agregatu — `Alarm` z metodami domenowymi (zamiast `@Data`)

Zachowujemy nazwę bytu `Alarm` (load-bearing w 20+ modułach), ale zamieniamy
anemiczny worek na **strażnika**: zdejmujemy `@Data`/`@Builder`, settery flag stają
się prywatne, a przejścia wyrażamy metodami z preconditions.

```java
// common/data — Alarm jako aggregate root (pseudokod sygnatur)
public final class Alarm extends BaseData<AlarmId> implements HasName, HasTenantId, HasCustomerId {

    // pola jak dziś, ale: brak public setterów dla acknowledged/cleared/severity/originator/ts

    // --- FABRYKA (egzekwuje niezmienniki UTWORZENIA: I4, I5, startTs) ---
    public static Alarm raise(TenantId t, EntityId originator, String type,
                              AlarmSeverity severity, long startTs, AlarmDetails details) {
        require(originator != null, MISSING_ORIGINATOR);
        require(severity   != null, MISSING_SEVERITY);
        // status startowy ZAWSZE ACTIVE_UNACK — nie da się utworzyć od razu cleared/ack
        return new Alarm(/* acknowledged=false, cleared=false, ackTs=0, clearTs=0, ... */);
    }

    // --- REKONSTYTUCJA (z bazy; BEZ preconditions — stan już był legalny) ---
    public static Alarm reconstitute(AlarmRecord row) { /* mapowanie 1:1 */ }

    // --- PRZEJŚCIA (egzekwują I2, I3, I4) ---
    public AlarmTransition acknowledge(UserId by, long ackTs) {
        if (acknowledged) throw new IllegalAlarmTransitionException(ALREADY_ACKNOWLEDGED, id);
        this.acknowledged = true; this.ackTs = positive(ackTs);
        return AlarmTransition.acked(this, by);     // niesie skutki: komentarz + event
    }

    public AlarmTransition clear(UserId by, long clearTs, AlarmDetails patch) {
        if (cleared) throw new IllegalAlarmTransitionException(ALREADY_CLEARED, id);
        this.cleared = true; this.clearTs = positive(clearTs);
        if (patch != null) this.details = this.details.merge(patch);
        return AlarmTransition.cleared(this, by);
    }

    public AlarmTransition reraise(AlarmSeverity newSeverity, long ts) {
        if (cleared) throw new IllegalAlarmTransitionException(SEVERITY_FROZEN_WHEN_CLEARED, id); // I4
        this.severity = newSeverity; this.endTs = max(endTs, ts);
        return AlarmTransition.severityChanged(this, oldSeverity);
    }

    public AlarmTransition assign(UserId assignee, long ts) { /* I2: idempotencję ROZSTRZYGA domena */ }
    public AlarmTransition unassign(long ts) { ... }

    // I1 pozostaje: status nadal czysto wyliczany, bez settera
    public AlarmStatus getStatus() { return toStatus(cleared, acknowledged); }
}
```

Kluczowe: **nielegalna operacja rzuca `IllegalAlarmTransitionException`** (nazwany
błąd domenowy z kodem), a nie cicho aktualizuje stan i nie zwraca `boolean`.

### 4.2 Nazwane błędy domenowe

```java
// common/data/alarm/exception
public final class IllegalAlarmTransitionException extends RuntimeException {
    public enum Reason { ALREADY_ACKNOWLEDGED, ALREADY_CLEARED,
                         SEVERITY_FROZEN_WHEN_CLEARED, MISSING_ORIGINATOR, MISSING_SEVERITY,
                         CUSTOMER_MISMATCH /* I5 */ }
    private final Reason reason; private final AlarmId alarmId;
}
```

### 4.3 Repozytorium ładujące/zapisujące **cały agregat** (zamiast rozsianych mutatorów DAO)

Dziś DAO ma osobne czasowniki (`acknowledgeAlarm`, `clearAlarm`, `updateAlarm`,
`assignAlarm`, `createOrUpdateActiveAlarm` — `BaseAlarmService.java:108-171,268-285`).
Każdy z nich to fragment maszyny stanów w SQL. Zastępujemy je repozytorium
zorientowanym na agregat:

```java
public interface AlarmRepository {
    Optional<Alarm> loadForUpdate(TenantId t, AlarmId id);   // SELECT ... FOR UPDATE (lock pesymistyczny)
    PersistedAlarm save(Alarm aggregate);                    // UPSERT alarmu + rekordy propagacji + typy alarmu
}
```

`loadForUpdate` + mutacja w pamięci (przez metody §4.1) + `save` zastępują parę
„zgadnij-deltę-i-zaaplikuj-w-SQL". Reguła monotoniczności (I2) jest egzekwowana
**zanim** cokolwiek dotknie bazy.

### 4.4 Atomowość (I3/I6) — wszystko w **jednej** transakcji, fail-fast

```java
@Transactional
public AlarmInfo acknowledge(AcknowledgeAlarmCommand cmd) {
    Alarm a = repo.loadForUpdate(cmd.tenantId(), cmd.alarmId())
                  .orElseThrow(() -> new AlarmNotFoundException(cmd.alarmId()));   // I2: 404, osobny od 409
    AlarmTransition tx = a.acknowledge(cmd.user().getId(), cmd.ts());              // rzuca 409 gdy nielegalne
    PersistedAlarm saved = repo.save(a);                                          // alarm + propagacja (I6) atomowo
    domainEvents.publishInTx(tx.events());     // komentarz systemowy + ALARM_ACK — w TEJ transakcji (I3)
    return saved.info();
}
```

`createEntityAlarmRecord` (`BaseAlarmService.java:449-456`) i
`addSystemAlarmComment` (`DefaultTbAlarmService.java:244-248`) **przestają łapać i
logować** — wyjątek propaguje i wywołuje rollback. *Log-i-jedź-dalej → zatrzymaj.*

### 4.5 Cienkie API — parse → metoda agregatu → mapowanie błędu

```java
// AlarmController — dedykowane endpointy stają się cienkie
@PostMapping("/alarm/{alarmId}/ack")
public AlarmInfo ackAlarm(@PathVariable String alarmId) {
    var cmd = AcknowledgeAlarmCommand.of(getTenantId(), toAlarmId(alarmId), getCurrentUser());
    return tbAlarmService.acknowledge(cmd);     // koniec TODO z :173
}

// @RestControllerAdvice — mapowanie nazwanego błędu domenowego na HTTP
@ExceptionHandler(IllegalAlarmTransitionException.class)
ResponseEntity<?> onIllegalTransition(IllegalAlarmTransitionException e) {
    return status(409 /* CONFLICT */).body(error(e.getReason()));   // realizuje TODO :173/:188
}
```

Generyczny `POST /api/alarm` (`AlarmController.java:139`) przestaje różnicować flagi
w serwisie: parsuje wejście, woła `Alarm.raise(...)` (create) **albo** ładuje agregat
i woła odpowiednie metody przejść; nielegalne kombinacje odrzuca **domena**, nie
heurystyka `if (flag != flag)`.

### 4.6 Egzekucja przenosi się z klienta na serwer

UI **może zachować** filtry `!acknowledged`/`!cleared` (`alarm-table-config.ts:290,325`)
jako *UX* (wyszarzenie przycisku), ale przestają być **strażnikiem** — autorytatywne
odrzucenie nielegalnego przejścia daje serwer (409). Operacje bulk dostają jawny
kontrakt częściowego sukcesu zamiast polegać na pre-filtrze przeglądarki.

---

## KROK 5 — Before/after, plan faz, testy

### 5.1 Before / after (każde dzisiejsze miejsce reguły)

| Miejsce dziś | Before | After |
|---|---|---|
| `Alarm.java:46,71,73` | `@Data` + publiczne settery flag | Aggregate root; settery flag prywatne; przejścia jako metody z preconditions |
| `alarm-table-config.ts:290,325` | Klient **decyduje**, co wolno ack/clear (strażnik) | Klient tylko podpowiada UX; strażnikiem jest serwer (409) |
| `DefaultTbAlarmService.java:77-89` | Różnicowanie flag (rekonstrukcja intencji) | Jawne komendy; create→`raise`, zmiana→metoda agregatu |
| `DefaultTbAlarmService.java:118,140` | `throw "...already..."` na podstawie flagi z DB | `IllegalAlarmTransitionException(reason)` z domeny, przed dotknięciem DB |
| `BaseAlarmService.java:397-415 (merge)` | `setSeverity`/`setCleared` bez warunku | Brak `merge`; zmiana severity tylko `reraise()` z guardem `!cleared` (I4) |
| `BaseAlarmService.java:449-456` | `catch (Exception) log.warn` (połknięcie I6) | Propagacja w tej samej transakcji; wyjątek → rollback |
| `DefaultTbAlarmService.java:244-248` | `catch log.error` (połknięcie śladu I3) | Komentarz w transakcji przejścia; błąd zatrzymuje |
| `AlarmController.java:173,188` | TODO: zły kod błędu | `@ExceptionHandler` → `409 CONFLICT` |
| `TbCreateAlarmNode/TbClearAlarmNode` | Druga ścieżka schodzi do DAO | Druga ścieżka woła te same metody agregatu (jeden strażnik dla REST i rule engine) |

### 5.2 Plan faz refaktoru (guard-first, odwracalnie)

> **Mnożnik ryzyka:** brak buildu/testów Javy na PR (z `01-domain-distillation.md`
> KROK 5). Dlatego **Faza 0 to czysty dodatek testów** — zamienia „cicho/źle
> egzekwowane" na „czerwone" **zanim** ruszy jakakolwiek zmiana produkcyjna.

- **Faza 0 (test-first, zero zmian produkcyjnych):** charakteryzujące testy
  bieżącego zachowania maszyny stanów na `AlarmService` (legalne i nielegalne
  przejścia) + test pokazujący lukę `merge()` severity i połknięcie błędu
  propagacji. Czerwone = dokumentacja długu.
- **Faza 1 (model):** wprowadź `Alarm.raise/reconstitute/acknowledge/clear/reraise`
  + `IllegalAlarmTransitionException`; settery flag → prywatne. **Domena testowana
  w izolacji** (bez Springa/DB). *Test-first.*
- **Faza 2 (repozytorium):** `AlarmRepository.loadForUpdate/save`; przełóż
  `BaseAlarmService` na ładowanie-mutację-zapis agregatu; usuń `merge()`. Stare
  metody DAO stają się prywatnymi detalami `save`.
- **Faza 3 (orkiestracja):** `DefaultTbAlarmService` chudnie — usuń różnicowanie
  flag (`:77-89`); przejścia + skutki uboczne w jednej `@Transactional`; usuń
  `catch-log` (`:244-248`, `BaseAlarmService:449-456`).
- **Faza 4 (API):** `@ExceptionHandler` dla `IllegalAlarmTransitionException` →
  409; usuń TODO (`:173,188`); generyczny `saveAlarm` → jawne create/transition.
- **Faza 5 (rule engine):** `TbCreateAlarmNode/TbClearAlarmNode` na metody agregatu.
- **Faza 6 (UI):** zdejmij rolę *strażnika* z filtrów; obsłuż 409 w bulku
  (raport częściowego sukcesu). Filtry zostają jako UX.

### 5.3 Przypadki testowe niezmiennika (legalne / nielegalne)

**Legalne (mają przejść):**
- `raise` → status `ACTIVE_UNACK`; `acknowledge` → `ACTIVE_ACK`; `clear` → `CLEARED_ACK`.
- `raise` → `clear` (bez ack) → `CLEARED_UNACK`; potem `acknowledge` → `CLEARED_ACK`.
- `reraise(MAJOR→CRITICAL)` na ACTIVE → severity zmienia się; event severityChanged.
- propagacja: `save` alarmu z `propagate=true` tworzy rekordy encji **w tej samej
  transakcji**; rollback alarmu = brak rekordów propagacji (I6).

**Nielegalne (muszą rzucić nazwany błąd, nie zmienić stanu):**
- `acknowledge` na już-ack → `ALREADY_ACKNOWLEDGED` (→ 409).
- `clear` na już-cleared → `ALREADY_CLEARED` (→ 409).
- `reraise` na CLEARED → `SEVERITY_FROZEN_WHEN_CLEARED` (I4).
- `raise` bez originatora/severity → `MISSING_ORIGINATOR` / `MISSING_SEVERITY`.
- próba `setCleared(false)` — **nie kompiluje się** (setter prywatny) = niezmiennik
  egzekwowany przez typ.
- alarm dla customera ≠ customer originatora → `CUSTOMER_MISMATCH` (I5).
- błąd zapisu rekordu propagacji → rollback całości (brak częściowego zapisu, I6).

### 5.4 Nowe „load-bearing" nazwy do zarejestrowania

`Alarm.raise`, `Alarm.reconstitute`, `Alarm#acknowledge`, `Alarm#clear`,
`Alarm#reraise`, `AlarmTransition`, `IllegalAlarmTransitionException`
(+ `Reason`: `ALREADY_ACKNOWLEDGED`, `ALREADY_CLEARED`, `SEVERITY_FROZEN_WHEN_CLEARED`,
`MISSING_ORIGINATOR`, `MISSING_SEVERITY`, `CUSTOMER_MISMATCH`),
`AlarmNotFoundException`, `AlarmRepository#loadForUpdate`, `AlarmRepository#save`,
`AcknowledgeAlarmCommand` / `ClearAlarmCommand` / `AssignAlarmCommand`.

---

## Ograniczenia tego planu

- **Brak PRD** — intencja biznesowa I4 (czy severity *wolno* eskalować na ACTIVE)
  inferowana z zachowania `merge()` (`BaseAlarmService.java:415`) i deklaracji
  destylacji; do potwierdzenia z produktem. Projekt wybiera wariant zachowawczy
  (severity zamrożona po CLEARED).
- **Cytaty zweryfikowane** bezpośrednio: `Alarm.java`, `AlarmApiCallResult.java`,
  `DefaultTbAlarmService.java`, `BaseAlarmService.java`, `AlarmController.java`,
  `alarm-table-config.ts`. Istnienie 2. ścieżki zapisu (`TbCreateAlarmNode`,
  `TbClearAlarmNode`) potwierdzone listowaniem; ich wewnętrzne linie nie cytowane.
- Plan **nie** dotyka I1 (status wyliczany) — jest zdrowy i pozostaje wzorcem.
```
