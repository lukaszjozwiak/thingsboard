---
title: Anti-Corruption Layer — izolacja langchain4j od domeny AI (ThingsBoard)
created: 2026-06-13
type: refactor-plan
---

# Anti-Corruption Layer dla przeciekającej zależności: **langchain4j**

> **Produkt tej pracy to PLAN refaktoru, nie kod.** Nie modyfikuję kodu
> produkcyjnego. Wszystkie cytaty `plik:linia` zweryfikowane bezpośrednim
> otwarciem plików (branch `master`, commit roboczy `b7abed0664`).
> Krok po kroku: odkrycie → identyfikacja → klasyfikacja → diagnoza → projekt.

---

## KROK 0 — Odkryty kontekst

- **Brak formalnego PRD / `tech-stack.md`** (potwierdzone w `context/foundation/` —
  tylko `README.md` szablonu; spójne z `01-domain-distillation.md` KROK 0). Wizja
  z `README.md` korzenia nie wymienia AI — podsystem AI/LLM jest **świeżym
  dodatkiem** (KROK 2 destylacji: *Supporting (emerging)*, bus-factor 1).
- **Deklaracja wymienialności — nie w dokumencie, lecz w samej architekturze
  kodu.** Cały podsystem jest zbudowany wokół **wymienialnego providera**:
  `AiProvider` to enum **9 dostawców** (`AiProvider.java:20-28` — OPENAI,
  AZURE_OPENAI, GOOGLE_AI_GEMINI, GOOGLE_VERTEX_AI_GEMINI, MISTRAL_AI, ANTHROPIC,
  AMAZON_BEDROCK, GITHUB_MODELS, OLLAMA), a warstwa typów `Tb*`
  (`TbChatRequest`, `TbContent`, `TbChatResponse`, `TbResponseFormat`) + port
  `Langchain4jChatModelConfigurer` to **wyraźna próba odseparowania domeny od
  konkretnego SDK**. Intencja izolacji jest więc zadeklarowana *w kodzie* —
  refaktor ją **dokańcza**, nie wymyśla.
- **Sygnał ryzyka biblioteki — fork.** ThingsBoard nie używa upstreamowego
  langchain4j, lecz **własnego forka**: groupId `org.thingsboard.langchain4j`,
  wersja `1.16.1-TB1` (`pom.xml:147`, BOM `pom.xml:1056-1057`). Utrzymywanie
  patcha `-TB1` to twardy dowód, że ta zależność jest **niestabilna i kosztowna**
  — dokładnie profil biblioteki, którą ACL ma okiełznać.
- **Prior:** `repo-map.md` §4 (strefa ryzyka #6): *„langchain4j wszedł nawet do
  fundamentowego `common/data`"* oraz §2 (graf modułów): *„`common/data` …
  zależy tylko od protobuf i langchain4j-core"*. Ten plan weryfikuje i rozwija
  ten sygnał do pełnej inwentaryzacji przecieku.

### Stack i warstwy, przez które przecieka langchain4j

| Warstwa | Moduł Mavena | Co dziś zależy od `dev.langchain4j` |
|---|---|---|
| **Model domenowy** | `common/data` (fan-in **20** modułów) | port `Langchain4jChatModelConfigurer`, `AiChatModelConfig.configure()`, 9× `*ChatModelConfig`, DTO `TbChatRequest`/`TbContent` |
| **Kontrakt rule-engine** | `rule-engine-api` | `RuleEngineAiChatModelService` (sygnatura w `ChatRequest`/`ChatResponse`) |
| **Nody reguł** | `rule-engine-components` | `TbAiNode`, `TbResponseFormat`, `Langchain4jJsonSchemaAdapter` |
| **Orkiestracja** | `application/service/ai` | `AiChatModelService(Impl)`, `AiRequestsExecutor`, `DefaultAiRequestsExecutor`, `Langchain4jChatModelConfigurerImpl` |
| **REST** | `application/controller` | `AiModelController.sendChatRequest` (buduje `ChatRequest` na granicy HTTP) |
| **UI** | `ui-ngx` | **NIC** — `grep langchain4j **/*.ts` → 0 trafień. UI mówi wyłącznie JSON-em `Tb*`/`AiModelConfig`. |

---

## KROK 1 — Identyfikacja: gdzie langchain4j przecieka

Przeszukanie `dev.langchain4j` po całym drzewie: **24 pliki Java** (16
produkcyjnych + 8 testów) w **4 modułach** + **5 manifestów `pom.xml`**.
Pełna lista „kto dziś zna langchain4j" (zweryfikowane otwarciem plików):

### 1.1 Model domenowy — `common/data` (przeciek najgroźniejszy: fan-in 20)

| Plik:linia | Co przecieka |
|---|---|
| `ai/model/chat/AiChatModelConfig.java:19,29` | domenowy interfejs zwraca `dev.langchain4j.model.chat.ChatModel` z metody `configure(...)` |
| `ai/model/chat/Langchain4jChatModelConfigurer.java:18,20-39` | **cały interfejs** typowany w `ChatModel` (9 przeciążeń), nazwany od biblioteki, mieszka w warstwie modelu |
| `ai/model/chat/OpenAiChatModelConfig.java:18,52-54` | `configure()` → `ChatModel` (record domenowy) |
| + 8 sióstr: `AmazonBedrock`, `Anthropic`, `AzureOpenAi`, `GitHubModels`, `GoogleAiGemini`, `GoogleVertexAiGemini`, `MistralAi`, `Ollama` `ChatModelConfig.java` | każdy importuje `ChatModel` i implementuje `configure()` (double-dispatch) |
| `ai/dto/TbChatRequest.java:18-22,57-77` | DTO wire'owe importuje `ChatMessage`, `Content`, `SystemMessage`, `UserMessage`, `ChatRequest`; metoda `toLangChainChatRequest()` + `getLangChainMessages()` budują obiekty biblioteki |
| `ai/dto/TbContent.java:20-21,47,75-77` | DTO importuje `Content`, `TextContent`; metoda `toLangChainContent()` |
| `common/data/pom.xml:116-117` | **`langchain4j-core` w `compile` scope** → biblioteka na classpath 20 modułów |

### 1.2 Kontrakt rule-engine — `rule-engine-api`

| Plik:linia | Co przecieka |
|---|---|
| `RuleEngineAiChatModelService.java:19-20,25` | port reguł: `FluentFuture<ChatResponse> sendChatRequestAsync(AiChatModelConfig<C>, ChatRequest)` — **oba typy z biblioteki** |
| `rule-engine-api/pom.xml:102-103` | zależność `langchain4j` (pełna) |

### 1.3 Nody reguł — `rule-engine-components`

| Plik:linia | Co przecieka |
|---|---|
| `ai/TbAiNode.java:24-33,100,131,152-208,266-299` | importuje 8 typów langchain4j; buduje `UserMessage`/`SystemMessage`/`TextContent`/`PdfFileContent`/`ImageContent`/`ChatRequest`, konsumuje `ChatResponse` |
| `ai/TbResponseFormat.java:21-22,47,70,89,108-112` | domenowy format odpowiedzi zna `ResponseFormat`/`ResponseFormatType`; `toLangChainResponseFormat()` |
| `ai/Langchain4jJsonSchemaAdapter.java:20-29,49` | konwerter Jackson `ObjectNode` → langchain4j `JsonSchema` (10 typów `request.json.*`) |
| `rule-engine-components/pom.xml:157-158` | zależność `langchain4j` (pełna) |

### 1.4 Orkiestracja — `application/service/ai`

| Plik:linia | Co przecieka |
|---|---|
| `AiChatModelService.java:18,20` | `extends RuleEngineAiChatModelService` → dziedziczy sygnaturę w `ChatRequest`/`ChatResponse` |
| `AiChatModelServiceImpl.java:20-22,36-39` | przyjmuje/zwraca typy biblioteki; woła `config.configure(configurer)` |
| `AiRequestsExecutor.java:19-21,25` | port: `(ChatModel, ChatRequest) → FluentFuture<ChatResponse>` |
| `DefaultAiRequestsExecutor.java:21-23,74-75` | **`chatModel.chat(chatRequest)`** — JEDYNE realne wywołanie biblioteki w runtime |
| `Langchain4jChatModelConfigurerImpl.java:20-29,60-253` | **realny adapter** — 9 metod budujących provider-specyficzne modele langchain4j (+ AWS SDK, Google creds) |

### 1.5 REST — `application/controller`

| Plik:linia | Co przecieka |
|---|---|
| `AiModelController.java:19,167,170-171` | import `ChatRequest`; na granicy HTTP woła `toLangChainChatRequest()`, przekazuje `ChatRequest` do serwisu, rozpakowuje `ChatResponse.aiMessage().text()` |
| `application/pom.xml:389-418` | **8 artefaktów provider-specyficznych**: open-ai, azure-open-ai, google-genai, mistral-ai, anthropic, bedrock, open-ai-official, ollama |

---

## KROK 2 — Klasyfikacja i wybór #1

Trzy osie oceny każdego kandydata na przeciek (z manifestów: zewnętrzne
zależności, które *mogłyby* przeciekać): (a) liczba warstw/plików dotkniętych,
(b) ryzyko/koszt wymiany dziś, (c) deklarowana wymienialność (rozjazd
intencja↔kod).

| Zależność | (a) Warstwy / pliki | (b) Ryzyko wymiany | (c) Deklarowana wymienialność | Werdykt |
|---|---|---|---|---|
| **langchain4j** | **5 warstw**, 16 plików prod + classpath 20 modułów (przez `common/data`) | **Wysokie** — fat-SDK, 8 provider-artefaktów, ciągnie AWS/Google/Azure SDK; **już forkowany `-TB1`** | **Tak, wprost** — enum `AiProvider` (9), warstwa `Tb*`, port-configurer = jawna próba ACL, której typy biblioteki *nie dotrzymują* | **#1 — najgorszy przeciek** |
| protobuf | wiele modułów (`queue.proto`) | n/d — to **kontrakt wire**, ma być znany | nie (świadomy kontrakt) | nie-kandydat (intencjonalny) |
| Jackson (`JsonNode`) | wszędzie | bardzo wysokie, ale to format-de-facto | nie | generyczny, nie-wymienialny |
| Hibernate/JPA | tylko `dao` | wysokie | nie | zamknięty w jednej warstwie |
| Kafka | za `common/queue` | — | częściowo (za portem) | już za abstrakcją |

### ✅ Wybór #1 — **langchain4j**

Wygrywa na **wszystkich trzech osiach naraz**: najszerszy zasięg warstw,
najwyższy koszt wymiany (fat-SDK + fork), i — najmocniejszy sygnał — **kod
deklaruje wymienialność, której sam nie dotrzymuje**. Pozostali kandydaci albo
są intencjonalnymi kontraktami (protobuf), albo generycznym de-facto standardem
(Jackson), albo już zamknięci w jednej warstwie (JPA) lub za portem (Kafka).

> #### Najgroźniejszy aspekt: SDK serwerowy w fundamentowym module
> Analog „biblioteka serwerowa wciągana do bundla klienta" w tej domenie to:
> **`common/data` — słownik repo z fan-in 20 modułów — ma `langchain4j-core` w
> `compile` scope** (`common/data/pom.xml:116-117`). To wciąga fat-SDK AI na
> classpath transportów (mqtt/coap/lwm2m), `dao`, `edqs` i całej reszty, która
> z AI nie ma nic wspólnego. UI jest tu **wzorem do naśladowania**: zna wyłącznie
> JSON `Tb*`/`AiModelConfig`, zero langchain4j (`grep` → 0). To, co osiągnął
> frontend (mówić kontraktem domenowym), backend ma osiągnąć w typach Javy.

---

## KROK 3 — Diagnoza: duplikacja i przecieki przez granice

### 3.1 Port jest **odwrócony** — nazwany i typowany od biblioteki, w warstwie modelu

`Langchain4jChatModelConfigurer` (`:20-39`) wygląda jak port (interfejs +
osobny impl), ale:
1. **zwraca typ biblioteki** `ChatModel` z każdej z 9 metod,
2. **jest nazwany od biblioteki** (`Langchain4j...`),
3. **mieszka w `common/data`** (warstwa modelu), nie w warstwie adaptera.

To nie ACL — to biblioteka przebrana za port. Reszta kodu *nie* zna „tylko portu";
zna `ChatModel`, bo port go zwraca.

### 3.2 Typ biblioteki płynie przez **całą** wysokość stosu — aż do REST

```text
AiModelController.java:167   ChatRequest langChainChatRequest = tbChatRequest.toLangChainChatRequest();  // REST buduje typ biblioteki
AiModelController.java:170   aiChatModelService.sendChatRequestAsync(chatModelConfig, langChainChatRequest);
AiChatModelService.java:20   extends RuleEngineAiChatModelService                                         // port dziedziczy sygnaturę langchain4j
RuleEngineAiChatModelService.java:25   FluentFuture<ChatResponse> sendChatRequestAsync(..., ChatRequest)  // kontrakt = typy biblioteki
DefaultAiRequestsExecutor.java:74      chatModel.chat(chatRequest)                                         // dopiero TU realne wywołanie
```

`ChatRequest`/`ChatResponse` to **typ przelotowy** przez 4 warstwy. Jedyne
miejsce, które *naprawdę* potrzebuje biblioteki, to `:74`
(`DefaultAiRequestsExecutor`). Wszystko powyżej tylko **przenosi** typ biblioteki.

### 3.3 Konwersja domena→langchain4j jest **zduplikowana** (dwa niezależne tory)

Ten sam akt „zbuduj `UserMessage`/`Content`/`ChatRequest` z promptów" istnieje
**dwa razy**, w dwóch modułach, inną mechaniką:

```text
# Tor REST (common/data):
TbChatRequest.java:63-77   getLangChainMessages(): SystemMessage.from(...), UserMessage.from(contents)
TbContent.java:75-77       toLangChainContent(): TextContent.from(text)

# Tor rule-engine (rule-engine-components), NIEZALEŻNIE:
TbAiNode.java:178-190      buildAndSendRequest(): SystemMessage.from(...), ChatRequest.builder().messages(...).responseFormat(...)
TbAiNode.java:266-299      buildContents()/toContent(): new TextContent(...), new PdfFileContent(...), new ImageContent(...)
```

**Rozjazd kompletności:** `TbContent` (model) zna **tylko TEXT**
(`TbContent.java:43,49-53`), ale `TbAiNode` poza modelem buduje też
`PdfFileContent`/`ImageContent` (`:291-296`). Czyli rule-engine **obchodzi
niekompletny model domenowy** wstrzykując typy biblioteki wprost — klasyczny
objaw braku ACL: gdy domena nie modeluje pojęcia, kod sięga po typ biblioteki.

### 3.4 Trzeci tor konwersji — format odpowiedzi i JSON-schema

```text
TbResponseFormat.java:108-112       toLangChainResponseFormat(): ResponseFormat.builder().jsonSchema(Langchain4jJsonSchemaAdapter.fromObjectNode(schema))
Langchain4jJsonSchemaAdapter.java:49  fromObjectNode(ObjectNode) → JsonSchema   // 10 typów request.json.*
```

Mapowanie `ResponseFormat`/`JsonSchema` żyje w `rule-engine-components`, choć jest
czystą translacją do biblioteki — należy do adaptera, nie do nodu reguły.

### 3.5 Dowód deklaracji wymienialności, której kod nie dotrzymuje

`AiProvider.java:20-28` (9 dostawców) + `AiModelConfig.java:43-101` (sealed
dyspozytor po `provider`) + warstwa `Tb*` = jawna inwestycja w **wymienialność
dostawcy**. A jednak `grep dev.langchain4j` trafia w **5 warstw** — wymiana
biblioteki (a nie dostawcy!) dotknęłaby modelu, kontraktu reguł, nodów, serwisu
i kontrolera. **Intencja izolacji jest, implementacja jej nie realizuje.**

### 3.6 Asymetria, która wskazuje wzorzec docelowy

`TbChatResponse` (`:41-76`) jest **całkowicie wolny od langchain4j** — to czysty
sealed record (`Success(String)` / `Failure(String)`). Konwersja
`ChatResponse → TbChatResponse` żyje już dziś w kontrolerze
(`AiModelController.java:171`: `chatResponse.aiMessage().text()`). Strona
**odpowiedzi pokazuje docelowy kształt**; strona **żądania** (i model, i nod)
go łamie. ACL = rozszerzyć wzorzec `TbChatResponse` na całą granicę.

**Wniosek diagnozy:** istnieje *zalążek* ACL (typy `Tb*`, enum providerów, port),
ale przeciek jest poczwórny: (1) port zwraca typ biblioteki i siedzi w modelu,
(2) typ biblioteki jest przelotowy do REST, (3) konwersja zduplikowana w 3 torach,
(4) `langchain4j-core` na classpath 20 modułów.

---

## KROK 4 — Projekt ACL

Cel: **jedno miejsce** (pakiet-adapter) zna `dev.langchain4j`. Reszta kodu —
model, kontrakt reguł, nody, serwis, kontroler — zna wyłącznie typy domenowe.
Wymiana biblioteki = zmiana tylko adaptera.

### 4.1 Domenowe value objects (jedyne źródło kształtu zapytania/odpowiedzi)

Wszystkie w `common/data`, **bez importu `dev.langchain4j`**. Zdejmujemy metody
`toLangChain*` (przenoszą się do mappera w adapterze) i **dokańczamy model**
(TbContent zyskuje warianty, których dziś brak):

```java
// common/data/ai/dto — czyste dane, zero langchain4j
public record TbChatRequest(String systemMessage, TbUserMessage userMessage,
                            AiChatModelConfig<?> chatModelConfig,
                            TbResponseFormat responseFormat) { }   // + responseFormat wniesiony do modelu

public sealed interface TbContent permits TbTextContent, TbImageContent, TbPdfContent {
    TbContentType contentType();          // TEXT | IMAGE | PDF  — domyka rozjazd §3.3
    // BRAK toLangChainContent()
}

// TbResponseFormat: PRZENIESIONY z rule-engine-components do common/data; czyste dane
public sealed interface TbResponseFormat permits TbTextResponseFormat, TbJsonResponseFormat, TbJsonSchemaResponseFormat {
    TbResponseFormatType type();
    boolean isSupportedBy(AiChatModelConfig<?> cfg);   // zostaje — to reguła domenowa (capability)
    // BRAK toLangChainResponseFormat()
}

// TbChatResponse — JUŻ czysty, wzorzec; bez zmian (Success/Failure)
```

`AiChatModelConfig` traci wiedzę o bibliotece:

```java
public sealed interface AiChatModelConfig<C extends AiChatModelConfig<C>> extends AiModelConfig {
    // USUNIĘTE: ChatModel configure(Langchain4jChatModelConfigurer configurer);
    AiProvider provider();
    Integer timeoutSeconds(); Integer maxRetries();
    C withTimeoutSeconds(Integer s); C withMaxRetries(Integer r);
    boolean supportsSchemalessJsonOutput(); boolean supportsJsonSchemaOutput();
}
```

`Langchain4jChatModelConfigurer` (interfejs) **znika z `common/data`** —
dyspozycja po providerze przenosi się do adaptera (§4.3).

### 4.2 Wąski port domenowy — jedyny kontrakt, jaki zna reszta kodu

Jeden czasownik, **wyłącznie typy domenowe**, w module widocznym dla reguł i
aplikacji (`rule-engine-api`, library-free po refaktorze):

```java
// rule-engine-api — BEZ importu dev.langchain4j
public interface RuleEngineAiChatModelService {
    FluentFuture<TbChatResponse> sendChatRequest(AiChatModelConfig<?> config, TbChatRequest request);
}
// application: interface AiChatModelService extends RuleEngineAiChatModelService {}  (bez zmian struktury, zmiana typów)
```

Wszystko powyżej portu (kontroler, `TbAiNode`) buduje `TbChatRequest` i czyta
`TbChatResponse` — **nigdy `ChatRequest`/`ChatResponse`/`ChatModel`**.

### 4.3 Adapter — JEDYNE miejsce wiedzy o langchain4j

Nowy pakiet `application/service/ai/langchain4j/**` (kandydat na osobny moduł
`application` → `ai-adapter`, jeśli zespół zechce twardej granicy Mavena).
Skupia **wszystkie** trzy tory konwersji z §3.3–3.4 + wykonanie:

```java
package org.thingsboard.server.service.ai.langchain4j;   // jedyny pakiet z importem dev.langchain4j

@Component
class Langchain4jAiChatModelService implements AiChatModelService {

    private final Langchain4jModelFactory modelFactory;     // dziś: Langchain4jChatModelConfigurerImpl (bez zmian logiki, 9 buildów)
    private final Langchain4jRequestMapper requestMapper;   // konsoliduje TbChatRequest.toLangChain* + TbAiNode.buildContents + TbResponseFormat + JsonSchemaAdapter
    private final AiRequestsExecutor executor;              // dziś: DefaultAiRequestsExecutor (chatModel.chat — :74)

    @Override
    public FluentFuture<TbChatResponse> sendChatRequest(AiChatModelConfig<?> config, TbChatRequest request) {
        ChatModel model = modelFactory.create(config);                 // domena → ChatModel  (provider dispatch TU)
        ChatRequest llReq = requestMapper.toChatRequest(request);      // domena → ChatRequest (jedyny tor konwersji)
        return executor.send(model, llReq)                             // chatModel.chat(...)
                       .transform(requestMapper::toTbResponse, ...)    // ChatResponse → TbChatResponse (wzorzec z §3.6)
                       .catching(Throwable.class, t -> new TbChatResponse.Failure(t.getMessage()), ...);
    }
}
```

`Langchain4jRequestMapper` wchłania: `TbContent→Content` (TEXT/IMAGE/PDF),
`TbResponseFormat→ResponseFormat`, `Langchain4jJsonSchemaAdapter` (przenoszony
1:1 z `rule-engine-components`), budowę `UserMessage`/`SystemMessage`/`ChatRequest`.
**Trzy tory z §3.3–3.4 zwijają się w jeden.**

### 4.4 Rozstrzygnięcie pytania zależnego od kontraktu biblioteki

`langchain4j ChatModel.chat(ChatRequest)` jest **blokujące** (synchroniczny zwrot
`ChatResponse`) — stąd dzisiejszy `DefaultAiRequestsExecutor` owija je w pulę
wątków (`:66-76`). **Decyzja:** most „blokujące→`FluentFuture`" to **detal
adaptera**, nie portu. Port deklaruje `FluentFuture<TbChatResponse>` (kontrakt
domenowy: „asynchronicznie"), a *jak* osiąga asynchroniczność (pula wątków,
timeout, retry=0 z `TbAiNode.java:227`) koduje adapter — w `AiRequestsExecutor`,
nie w API. To zakotwicza wiedzę o blokującej naturze SDK w ACL.

---

## KROK 5 — Dowód izolacji + before/after

### 5.1 Before / after (każde dzisiejsze miejsce przecieku)

| Miejsce dziś (plik:linia) | Before | After |
|---|---|---|
| `Langchain4jChatModelConfigurer.java:20-39` (common/data) | port w modelu, zwraca `ChatModel` | **usunięty z common/data**; dyspozycja providera → adapter `Langchain4jModelFactory` |
| `AiChatModelConfig.java:29` | `ChatModel configure(...)` w interfejsie domeny | metoda **usunięta**; config to czyste dane |
| `OpenAiChatModelConfig.java:52` + 8 sióstr | `configure()` → `ChatModel` | metoda **usunięta** z 9 recordów |
| `TbChatRequest.java:57-77`, `TbContent.java:47,75` | DTO buduje obiekty langchain4j | czyste dane; konwersja → `Langchain4jRequestMapper` |
| `TbResponseFormat.java` (rule-engine) | format + `toLangChainResponseFormat()` w nodzie reguł | **przeniesiony do common/data** jako dane; mapowanie → adapter |
| `Langchain4jJsonSchemaAdapter.java` (rule-engine) | konwerter w `rule-engine-components` | **przeniesiony** do pakietu-adaptera |
| `TbAiNode.java:152-208,266-299` | buduje `UserMessage`/`Content`/`ChatRequest`, czyta `ChatResponse` | buduje `TbChatRequest` (z `TbImageContent`/`TbPdfContent`), czyta `TbChatResponse`; **zero langchain4j** |
| `RuleEngineAiChatModelService.java:25` | sygnatura w `ChatRequest`/`ChatResponse` | sygnatura w `TbChatRequest`/`TbChatResponse` |
| `AiRequestsExecutor.java:25`, `DefaultAiRequestsExecutor.java:74` | port + impl w typach biblioteki | **zostają w adapterze** (detal); `chatModel.chat` nadal jedyny call-site |
| `AiModelController.java:167,170-171` | buduje `ChatRequest` na granicy HTTP | przekazuje `TbChatRequest` do portu; **zero langchain4j** |
| `common/data/pom.xml:116-117` | `langchain4j-core` (compile, classpath 20 modułów) | **usunięty** |
| `rule-engine-api/pom.xml:102-103`, `rule-engine-components/pom.xml:157-158` | `langchain4j` | **usunięte** |
| `application/pom.xml:389-418` | 8 artefaktów provider | **bez zmian** — adapter ich potrzebuje (właściwe miejsce) |

### 5.2 Dowód izolacji (kryterium sukcesu)

**Po refaktorze `grep -rl "dev.langchain4j" --include=*.java` zwraca wyłącznie
pliki w `application/.../service/ai/langchain4j/**`** (+ ich testy). Konkretnie:

- **Przestają znać langchain4j** (16 plików prod): wszystkie w `common/data/ai/**`
  (11), `RuleEngineAiChatModelService`, `TbAiNode`, `TbResponseFormat`,
  `Langchain4jJsonSchemaAdapter`, `AiModelController`.
- **Nadal znają (i tylko one — celowo):** pakiet-adapter
  `service/ai/langchain4j/**` — `Langchain4jModelFactory` (=dawny
  `Langchain4jChatModelConfigurerImpl`), `Langchain4jRequestMapper`
  (=skonsolidowane tory), `AiRequestsExecutor`+`Default...`, `Langchain4jJsonSchemaAdapter`.
- **Manifesty:** langchain4j znika z `common/data/pom.xml`,
  `rule-engine-api/pom.xml`, `rule-engine-components/pom.xml`; zostaje w
  `application/pom.xml` (8 provider-artefaktów + core). **`common/data` przestaje
  ciągnąć fat-SDK na classpath 20 modułów** — to główna wygrana.
- **UI bez zmian** — już dziś nie zna langchain4j; kontrakt JSON `Tb*` niezmieniony
  (warstwa UI dostaje gotowe dane domenowe, nie surowy obiekt biblioteki).

### 5.3 Bramka utrwalająca izolację

Z `refactor-opportunities/research.md` (§„brak sieci bezpieczeństwa w CI"):
**na PR nie biega żaden build/test Javy, zero ArchUnit**. Dlatego ostatnia faza
dodaje **regułę ArchUnit/enforcer**: „`dev.langchain4j` może być importowany
wyłącznie z `..service.ai.langchain4j..`". To zamienia przyszły nawrót przecieku
z „cichego" na „czerwony", spójnie z filozofią guard-first z `01`/`02`.

---

## KROK 6 — Weryfikacja i plan faz

> **Mnożnik ryzyka:** brak CI Javy (prior). Stąd **Faza 0 = czysty dodatek
> testów charakteryzujących** — zanim ruszy jakakolwiek zmiana produkcyjna.
> Istniejące szwy: `Langchain4jChatModelConfigurerImplTest` (9 provider-buildów)
> i `TbAiNodeTest` (budowa żądania) — do rozszerzenia o round-trip mappera.

- **Faza 0 (test-first, zero zmian prod):** testy charakteryzujące mapowania —
  domena→langchain4j→domena dla request/response/content(TEXT/IMAGE/PDF)/json-schema.
  Oprzeć o `Langchain4jChatModelConfigurerImplTest` + `TbAiNodeTest`. Czerwone =
  dokumentacja kontraktu, który mapper musi zachować.
- **Faza 1 (model, common/data):** wnieś `TbResponseFormat` do `common/data`;
  dodaj `TbImageContent`/`TbPdfContent` (domknięcie §3.3); zdejmij metody
  `toLangChain*` z `TbChatRequest`/`TbContent` (ciała przejmie mapper w Fazie 3).
- **Faza 2 (port):** przetypuj `RuleEngineAiChatModelService` na
  `TbChatRequest`/`TbChatResponse`; usuń `configure(...)` z `AiChatModelConfig`
  i 9 recordów; usuń interfejs `Langchain4jChatModelConfigurer` z `common/data`.
- **Faza 3 (adapter):** utwórz `service/ai/langchain4j/**`; przenieś
  `Langchain4jJsonSchemaAdapter` z rule-engine; skonsoliduj 3 tory konwersji w
  `Langchain4jRequestMapper`; `Langchain4jChatModelConfigurerImpl` →
  `Langchain4jModelFactory` (logika 9 buildów bez zmian); `AiRequestsExecutor`
  zostaje (most blokujące→future, §4.4).
- **Faza 4 (konsumenci):** przepisz `TbAiNode` (buduje `TbChatRequest`, czyta
  `TbChatResponse`) i `AiModelController.sendChatRequest` (port zamiast
  `toLangChainChatRequest`). Zero importów langchain4j w obu.
- **Faza 5 (manifesty + bramka):** usuń langchain4j z poma `common/data`,
  `rule-engine-api`, `rule-engine-components`; dodaj regułę ArchUnit (§5.3).
  Weryfikacja końcowa: `grep` izolacji (§5.2).

### Nowe „load-bearing" nazwy do zarejestrowania

`TbImageContent`, `TbPdfContent` (warianty `TbContent`), `TbResponseFormat`
(relokowany do `common/data`), `Langchain4jModelFactory`,
`Langchain4jRequestMapper`, pakiet `org.thingsboard.server.service.ai.langchain4j`,
port `RuleEngineAiChatModelService#sendChatRequest(AiChatModelConfig, TbChatRequest)`.

---

## Ograniczenia tego planu

- **Brak PRD** — „wymienialność dostawcy/biblioteki" inferowana z architektury
  kodu (`AiProvider` 9 wartości, warstwa `Tb*`, fork `-TB1`), nie z deklaracji
  produktowej; do potwierdzenia z zespołem AI (kontakt: Dmytro Skarzhynets,
  bus-factor 1 — `repo-map.md` §5).
- **Cytaty zweryfikowane** bezpośrednio: wszystkie pliki `common/data/ai/**`
  cytowane z linii, `RuleEngineAiChatModelService`, `TbAiNode`, `TbResponseFormat`,
  `Langchain4jJsonSchemaAdapter`, `AiChatModelService(Impl)`, `AiRequestsExecutor`,
  `DefaultAiRequestsExecutor`, `Langchain4jChatModelConfigurerImpl`,
  `AiModelController`, oraz 5 manifestów `pom.xml`. 8 sióstr `*ChatModelConfig`
  (poza `OpenAiChatModelConfig`) potwierdzone listowaniem katalogu + grep
  `dev.langchain4j`; dokładne linie `configure()` nie cytowane indywidualnie.
- **Plan nie dotyka** `TbChatResponse` (już czysty — pozostaje wzorcem) ani logiki
  9 provider-buildów (`Langchain4jChatModelConfigurerImpl` — relokowana 1:1, nie
  przepisywana). To refaktor **granic typów i zależności**, nie zachowania.
- **Granica adaptera jako pakiet vs moduł:** plan proponuje pakiet
  (`service/ai/langchain4j`) jako minimalny ruch; twarda granica Mavena (osobny
  moduł) jest mocniejsza, ale szersza — decyzja do sesji planowania.

---

## Podsumowanie

Najgorszym przeciekiem zależności w ThingsBoard jest **langchain4j**: fat-SDK AI,
który przecieka przez **5 warstw** (model `common/data`, kontrakt `rule-engine-api`,
nody `rule-engine-components`, serwis `application/service`, REST `controller`)
i — najgroźniej — siedzi w `compile` scope w `common/data`, wciągając bibliotekę
na classpath 20 modułów, które z AI nie mają nic wspólnego. Kod paradoksalnie
*deklaruje* wymienialność (enum `AiProvider` z 9 dostawcami, warstwa typów `Tb*`,
port-configurer), ale typy `dev.langchain4j` przelatują przez każdą granicę aż do
kontrolera REST, a konwersja domena→biblioteka jest zduplikowana w trzech
niezależnych torach (`TbChatRequest`/`TbContent`, `TbAiNode`, `TbResponseFormat`
+`Langchain4jJsonSchemaAdapter`); dowodem niestabilności biblioteki jest własny
fork `1.16.1-TB1`. Projekt ACL zwija wszystkie tory w jeden adapter
(`service/ai/langchain4j/**`), wprowadza wąski port domenowy
(`sendChatRequest(AiChatModelConfig, TbChatRequest) → TbChatResponse`) i czyste
value objects (dokańczając model o `TbImageContent`/`TbPdfContent`, których dziś
brak, przez co rule-engine obchodzi domenę). Kryterium sukcesu jest jednoznaczne:
po refaktorze `grep dev.langchain4j` trafia wyłącznie w pakiet-adapter, a
langchain4j znika z manifestów `common/data`/`rule-engine-*`. Plan jest
guard-first (Faza 0 = testy charakteryzujące na istniejących szwach
`Langchain4jChatModelConfigurerImplTest`/`TbAiNodeTest`) i domyka się bramką
ArchUnit, bo backend nie ma CI — spójnie z `01`/`02`.
