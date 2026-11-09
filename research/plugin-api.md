# Cosa può fare un plugin nel branch `feature/plugin-architecture` (API v1)

Ticket: [#4](https://github.com/Omixxx/learnharness/issues/4) (parte di #1).
Fonte: `upstream/feature/plugin-architecture` @ `685c110` (2026-11-07), letto in sola lettura
con `git show` / `git diff main...upstream/feature/plugin-architecture`. Tutti i riferimenti
`file:riga` sono **su quel branch**, salvo dove indicato `main:`.

## Sintesi

| # | Capacità | Verdetto | Come / perché |
|---|---|---|---|
| 1 | UI dentro i messaggi del teacher / azione su selezione | **Parziale** | Solo come blocco: il teacher emette un code fence ` ```lingua ` e il plugin lo renderizza (`registerCodeBlock`). Nessuno span inline, nessun hook sulla selezione. |
| 2 | Testo nel prompt del teacher e/o tool per il teacher | **Sì** | `registerInstructions` / `registerTools` con `agents: ["teacher"]`. |
| 3 | Righe nelle Impostazioni (profilo lingua, velocità) | **Sì, con limiti** | `registerSettings({rows})` con toggle/text/dropdown/number/slider, ma solo nella pagina Settings **del plugin**; la persistenza è a carico del plugin; non tocca le impostazioni core. |
| 4 | Route backend che legge `DEEPGRAM_API_KEY` per mintare un token / proxy | **Parziale** | Mint di token: possibile via `ctx.handle` + lettura di `process.env`, ma **fuori contratto** (l'API non espone credenziali). Proxy di streaming/WebSocket: **no** (bridge solo JSON). |
| 5 | Estendere/sostituire la dettatura (provider/parametri STT) | **No** | Il plugin può *usare* la dettatura esistente (`ctx.ui.mountDictation`) nella propria UI, ma provider (Deepgram), URL, token e lingua sono cablati. Il diff di `dictation.tsx` serve proprio a `mountDictation`; quello di `settings.ts` non riguarda la dettatura. |

## Contesto: modello del sistema plugin

- I plugin sono **codice locale fidato**, non sandbox: il backend gira nel server Node di
  LearnHarness, il frontend nella pagina dell'app (`docs/plugin-architecture.md:10-13`,
  `docs/plugin-authoring.md:3-6`).
- Host = identità/ciclo di vita/contratti nativi/trasporto credenziali; plugin = dati,
  prompt, tool, presentazione (`docs/plugin-architecture.md:15-19`).
- Contratti pubblici: `src/core/plugins.ts` (contesti backend `:106-132`, frontend `:194-232`),
  `src/core/plugin-ui.ts`, ecc.; solo import *type-only* (`docs/plugin-authoring.md:55-63`).
  I file `src/server/*` e `components/*` sono seam di implementazione, **non** import
  supportati (`docs/plugin-architecture.md:45-46`).
- Pacchetto: `<learning data>/plugins/<id>/dist/manifest.json` con `main.js` (ESM browser),
  `backend.cjs` (Node), `styles.css`, `assets/`; dati persistenti in `<id>/data/`
  (`docs/plugin-authoring.md:65-98`). Niente hot reload: le modifiche al codice richiedono
  un riavvio (`docs/plugin-authoring.md:17-18`, `docs/plugin-architecture.md:73-75`).

## 1. UI dentro i messaggi del teacher / azione sulla selezione — PARZIALE

**Cosa c'è.**
- `registerCodeBlock({language, mount})` (`src/core/plugins.ts:163-174`, `:210`;
  `docs/plugin-authoring.md:366-375`): il plugin riceve `source`, `meta`, `incomplete`
  e monta un proprio `HTMLElement` al posto del code fence di quella lingua.
- Il dispatch avviene nella pipeline Markdown condivisa dei messaggi:
  `components/md-blocks.tsx:242-278` → `PluginCodeBlock` (`components/plugin-renderers.tsx:43-49`).
- Pattern combinato già usato in produzione da `plugins/rich-code-blocks`:
  il backend aggiunge istruzioni al teacher per emettere certi fence
  (`plugins/rich-code-blocks/src/backend.ts:16-21`), il frontend registra il renderer
  (`plugins/rich-code-blocks/src/main.ts:37-39`). Stesso schema in
  `examples/plugins/teaching-extension/dist/backend.cjs:13-18` + `dist/main.js:9`.

Quindi un 🔊 è realizzabile così: istruzioni al teacher "racchiudi il passaggio in lingua di
studio in ` ```speak lang=de ` … ` ``` `" + renderer che mostra il testo con un pulsante
play. È un **blocco a sé** nel messaggio, non un pulsante accanto a uno span inline.

**Cosa manca.**
- **Inline code non passa dai plugin**: il ramo inline di `MarkdownCode` ritorna subito un
  `<code>` (`components/md-blocks.tsx:247-253`); solo i fence con `data-block` arrivano a
  `PluginCodeBlock` (`:278`). Anche il Markdown `displayOnly` (`:278`) e `ctx.ui.mountMarkdown`
  saltano i renderer plugin (`docs/plugin-authoring.md:238-241`, `:372`).
- **Nessun hook di selezione testo / menu contestuale**: nessuna occorrenza di
  `selection`/`getSelection`/`contextmenu` rilevante in `src/core/plugin*.ts` o
  `docs/plugin-*.md` (le uniche "selection" riguardano risposte a quiz e checkbox dell'albero).
  I placement di azioni sono solo `sidebar`, `graph`, `top-bar`
  (`src/core/plugins.ts:159-162`, `docs/plugin-authoring.md:153-163`).
- Workaround tecnicamente possibile ma **fuori contratto**: il frontend gira nella pagina,
  quindi potrebbe ascoltare `selectionchange` sul DOM; la doc vieta di dedurre autorità dal
  "private DOM" (`docs/plugin-authoring.md:400-402`) e non c'è stabilità garantita del markup.

**Alternative supportate.**
- Azione nella top bar della sessione con `context.view` (snapshot congelato dell'intera
  sessione, inclusi `messages` e `transcript`): es. "leggi l'ultimo messaggio del teacher"
  (`src/core/plugins.ts:143-149`, `docs/plugin-view-context.md:8-35`, `docs/plugin-authoring.md:165-172`).
- `registerToolRenderer` per una card legata a una chiamata di tool del teacher
  (`src/core/plugins.ts:175-193`, `:211`; `docs/plugin-authoring.md:366-370`).

## 2. Testo nel prompt del teacher e tool — SÌ

- `registerInstructions({agents, text})` e `registerTools({agents, create(context)})` sul
  contesto backend (`src/core/plugins.ts:93-105`, `:120-121`); `PluginAgentKind =
  "prober" | "teacher" | "practice"` (`:91`).
- Additivi: non sostituiscono prompt, modello o tool built-in; il testo è composto nel
  system prompt; un loadout Pi nativo fresco per turno con `{chatId, agent}` congelati
  (`docs/plugin-authoring.md:347-353`). Admission atomica per pacchetto, nomi core riservati,
  errori → warning senza rompere il resto (`docs/plugin-architecture.md:125-130`,
  `docs/plugin-authoring.md:355-364`).
- Regole nomi tool: 1–64 caratteri provider-safe (`docs/plugin-authoring.md:355-356`).
- Esempi: `plugins/coding-questions/src/backend.ts:32-33` (istruzioni teacher e practice),
  `plugins/rich-code-blocks/src/backend.ts:19`, `examples/plugins/teaching-extension/dist/backend.cjs:13-18`
  (istruzioni + tool nativi). Per tool asincroni: `ctx.createBackgroundTool` (`src/core/plugins.ts:119`,
  `docs/plugin-authoring.md:465-470`).

## 3. Righe nelle Impostazioni — SÌ, con limiti

- `registerSettings({rows})` o `registerSettings(mount)` (`src/core/plugins.ts:206-208`).
  Tipi di riga: `toggle`, `text`, `dropdown`, `number`/`slider` con `min/max/step`
  (`src/core/plugin-ui.ts:126-140`); validazione sincrona, Promise pendente disabilita la riga
  (`docs/plugin-authoring.md:282-295`). Profilo lingua = `dropdown`; velocità = `slider`.
- **Dove**: ogni plugin abilitato ha la *sua* pagina Settings; Settings → Plugins gestisce i
  pacchetti (`docs/plugin-authoring.md:276-280`, `docs/plugin-architecture.md:94-95`).
  Non si possono aggiungere righe alle sezioni core (es. accanto alla lingua di dettatura).
- **Persistenza a carico del plugin**: "The host does not load/save plugin settings"
  (`docs/plugin-authoring.md:294-295`); tipicamente `ctx.call` → `ctx.handle` → file in
  `ctx.dataDirectory` (es. `plugins/drawing-pad/src/backend.ts:21-46`,
  `plugins/rich-code-blocks/src/main.ts:13-27`).
- Nessuna scrittura sulle impostazioni core (es. `dictation.language`):
  "programmable core-data writes" è un non-goal (`docs/plugin-architecture.md:251-254`).

## 4. Backend che legge una chiave server-side — PARZIALE

**Cosa c'è.**
- Il backend può registrare operazioni JSON con `ctx.handle(name, handler)`; il frontend le
  chiama con `ctx.call(name, input)` (`src/core/plugins.ts:126-127`, `:215`;
  `docs/plugin-authoring.md:329-332`). Trasporto: `POST /api/plugins/[id]/call`
  (`app/api/plugins/[id]/call/route.ts`) → `pluginTransport.call`.
- Il backend gira nel processo Node con accesso a rete e filesystem
  (`docs/plugin-authoring.md:3-5`), quindi può fare `fetch("https://api.deepgram.com/v1/auth/grant")`.

**Cosa manca / rischi.**
- **Nessun accesso sancito alle chiavi.** `keyValue` non è su `PluginBackendContext`
  (`src/core/plugins.ts:106-132`); la doc: "The API does not expose provider credentials"
  (`docs/plugin-authoring.md:5-6`) e "no credential DTO, custom transport or silent key
  fallback is exposed" (`docs/plugin-architecture.md:185-187`). Importare `src/server/keys.ts`
  non è supportato (`docs/plugin-architecture.md:45-46`).
- **`process.env.DEEPGRAM_API_KEY` funziona in pratica, ma fuori contratto.** `keyValue` legge
  env-first, poi lo store `auth.json` (`src/server/keys.ts:184-191`); `initAuthEnv` copia le chiavi
  di `auth.json` in `process.env` (fill-gap, `src/server/keys.ts:93-125`) e `saveKey` aggiorna
  `process.env` al volo (`:215-226`). `DEEPGRAM_API_KEY` è una chiave gestita
  (`src/core/auth-keys.ts:34`). Caveat: la stessa `keys.ts` segnala un "cold-boot hole" in cui
  una lettura diretta dell'env precede il fill-gap (`src/server/keys.ts:180-183`); con
  `LH_IGNORE_ENV_KEYS=1` le env reali vengono cancellate e restano solo le copie da file
  (`:80-85`, `:105-112`). Va trattato come dettaglio implementativo che può cambiare.
- **Proxy: no.** Il bridge è "JSON-compatible operation results, not arbitrary HTTP endpoints"
  (`src/core/plugins.ts:126`, `docs/plugin-architecture.md:84-87`). Nessuno streaming,
  nessun WebSocket: si può mintare e restituire un token, non fare da proxy all'audio.
- Nota: il frontend plugin gira nella stessa origine, quindi potrebbe anche chiamare la route
  core esistente `GET /api/stt/token` (`main:app/api/stt/token/route.ts:18-47`) — anche questo
  non è parte dell'API plugin.

## 5. Estendere/sostituire la dettatura — NO (solo consumo)

**Perché il branch tocca `components/dictation.tsx` (+213/−59 circa).** Per rendere il motore
di dettatura riusabile dai plugin tramite `ctx.ui.mountDictation`
(`src/core/plugin-ui.ts:38-46`, `:146`; host: `components/plugin-ui-host.tsx:187`;
adattatore: `components/plugin-dictation.tsx:10-15`). Le modifiche:
- lock di registrazione **unico per finestra** (`recordingOwner`) e messaggio
  `"Dictation is active elsewhere"` (`components/dictation.tsx:113-144`, start `:524-531`);
- `destination: "primary" | "explicit"`: le destinazioni plugin non registrano il target
  core composer/domanda né il finisher core (`:332-337`, `:357-363`, `:687-692`, flush `:392-399`);
- `onTranscript` asincrono che fa parte di `finish()`, con propagazione degli errori
  (cleanup/complete, `:401-458`);
- registro condiviso degli shortcut ⌘D/Esc con priorità al controllo plugin focalizzato
  (`registerController`, `:131-144`; `components/plugin-input-focus.ts`);
- prop `disabled` (`:336`, `DictationBar`).
Documentato in `docs/plugin-authoring.md:209-212` ("one recording owner per window…").

**Perché `src/core/settings.ts` (+10).** Non c'entra con la dettatura: aggiunge solo
`agent.workerRequestConcurrency` (default 8) per il pool di admission dei worker in background
(`git diff main...upstream/feature/plugin-architecture -- src/core/settings.ts`;
`docs/plugin-authoring.md:485-492`).

**Nessun seam per provider/parametri.**
- Props di `mountDictation`: solo `{disabled, onTranscript}` + handle `toggle/finish/cancel`
  (`src/core/plugin-ui.ts:38-46`).
- Provider cablato: token da `/api/stt/token` (`components/dictation.tsx:153`), socket
  `new WebSocket(deepgramListenUrl(language), ...)` (`:466`), lingua = preferenza globale
  utente (`useDictationLanguage`, `:340`).
- Conclusione: un plugin può mettere un pulsante di dettatura nella *propria* UI e ricevere il
  testo, ma non può cambiare modello/parametri Deepgram, sostituire il provider o intercettare
  la dettatura core. Unica leva indiretta: `registerSubmissionInterceptor` trasforma l'input
  del learner **dopo** la dettatura, prima dell'invio (`docs/plugin-authoring.md:393-398`) —
  post-processing di testo, non STT.

## Stabilità dichiarata dell'API

- `PLUGIN_API_VERSION = 1` (`src/core/plugins.ts:22`), manifest con `apiVersion: 1`
  obbligatorio (`:242`).
- **Nessuna policy di stabilità/deprecazione dichiarata.** Le doc dicono di descrivere "the
  current checkout's API v1, including additions not present in the released 0.4.1 app"
  (`docs/plugin-authoring.md:8-10`, `docs/plugin-architecture.md:4-6`,
  `docs/plugin-view-context.md:3-5`) e che non sono un verdetto di release-readiness
  (`docs/archive/plugin-implementation-history.md:3-8`, `docs/plugin-architecture.md:243-247`).
- I tipi nativi Pi fanno parte dell'API v1 e seguono l'SDK pinnato dell'app
  (`docs/plugin-authoring.md:62-63`): dipendenza esterna che può muovere i tipi.

## Quanto è vicino al merge

- 18 commit su `main..branch`, tutti di amosblomqvist, dal 2026-11-01
  (`33dded2 feat: add trusted plugin platform…`) al 2026-11-07
  (`685c110 feat: add drawing pad and coding practice plugins`); attività quotidiana,
  messaggi ancora `feat:`/`chore: checkpoint …`.
- Merge-base con `main` = `e69e852 release: prepare 0.4.1` (2026-10-31); unico merge di main
  nel branch: `efe175d Merge main v0.4.1`. Da allora `main` ha **83 commit** non integrati ed è
  passato a **0.5.0** (`main:package.json`) senza il sistema plugin; il branch è ancora a 0.4.1
  e tutti i manifest dichiarano `minAppVersion: "0.4.1"`.
- ~540 file cambiati; doc estese e curate (architettura, authoring, 5 contratti, skill
  `skills/learnharness-plugins/SKILL.md`), suite di test ampia (`tests/plugin-*.test.ts*`).
- Lettura: maturo come design e test, ma **non in rilascio**; servirà almeno un rebase/merge
  su 0.5.0 e la verifica di produzione che la doc stessa elenca come mancante
  (`docs/plugin-architecture.md:235-247`). Nessun segnale di data di merge.

## Plugin di esempio con pattern simili

| Plugin | Pattern rilevante per noi |
|---|---|
| `plugins/rich-code-blocks` | **Il più vicino**: istruzioni al teacher per emettere fence specifici + `registerCodeBlock` + righe Settings persistite via backend (`src/backend.ts:16-47`, `src/main.ts:13-46`). |
| `examples/plugins/teaching-extension` | Istruzioni + tool nativi per teacher/practice + renderer + worker (`dist/backend.cjs:13-18`, `dist/main.js:9`). |
| `plugins/drawing-pad` | Settings via `ctx.handle`, azione top-bar con `context.view`, interceptor di invio (`src/backend.ts:21-46`, `src/main.ts:36-51`, `:99`, `:140-141`). |
| `plugins/coding-questions` | Istruzioni teacher/practice, tipi di domanda/problema (`src/backend.ts:32-33`). |
| `plugins/flashcards` | Backend con operazioni multiple, settings custom mount (`src/backend.cjs:227`, `src/main.js:377`). |
| `examples/plugins/hello-plugin`, `managed-agent`, `plugins/ui-showcase` | View/azioni/settings base; agenti gestiti; galleria dei 15 helper `ctx.ui`. |

## Implicazioni per le due funzioni

- **TTS su passaggi segnati**: fattibile come plugin oggi su questo branch, *se* si accetta un
  blocco fence al posto del pulsante inline e una lettura `process.env` fuori contratto (o un
  provider TTS chiamato dal backend con chiave salvata nei dati del plugin). Azione su
  selezione richiederebbe DOM non supportato o una modifica core.
- **Dettatura configurabile (provider/parametri)**: non fattibile come plugin; richiede
  modifiche core a `components/dictation.tsx` / `src/core/dictation.ts` o una nuova estensione
  dell'API da proporre al maintainer.
