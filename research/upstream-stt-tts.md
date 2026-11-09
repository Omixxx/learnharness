# Lezioni dal lavoro upstream su STT e TTS

Ticket: https://github.com/Omixxx/learnharness/issues/5 (parte della mappa #1 del fork).
Data della rilevazione: 2026-11-09. Upstream: `amosblomqvist/learnharness` (solo lettura).
Main upstream di riferimento: `da08cb5` (dopo v0.5.0, 2026-11-04).

Fonti primarie usate: `gh pr view`/`gh issue view` (corpo, commenti, review, file toccati),
GraphQL per le Discussion, `gh api search/issues` per l'attività del maintainer,
`git log` sul nostro clone, `gh api .../compare/main...feature/plugin-architecture`
e i file `docs/plugin-*.md` su quel branch upstream.

**Limite dichiarato:** il work board (https://github.com/users/amosblomqvist/projects/2)
non è stato letto. Il token `gh` non ha lo scope `read:project` (GraphQL restituisce
`INSUFFICIENT_SCOPES`), e il fetch anonimo della pagina restituisce solo la cornice di GitHub
senza item. Lo stato "In progress" delle voci STT/TTS sul board resta quindi sconosciuto.

---

## 1. Il dato principale: nessun coinvolgimento del maintainer sulle PR STT/TTS

Su ogni PR STT/TTS analizzata (#330, #471, #203, #478, #575, #562, #534, #101, #239) ci sono
**zero review e zero commenti di `amosblomqvist`**. Il dato viene da `gh pr view --json comments,reviews`
e dalla ricerca `repo:amosblomqvist/learnharness commenter:amosblomqvist`, che conta 90 item.
Fra questi l'unica traccia STT è la chiusura di issue sulla lingua (#100, #231, #238).
Anche le Discussion #607 (TTS) e #587 (multilingua) non hanno nessuna risposta.

Contesto quantitativo: upstream ha **177 PR aperte, 29 mergiate e 61 chiuse senza merge**
(search API, 2026-11-09). Le PR STT/TTS aperte che competono sullo stesso seam sono almeno **14**:
#50, #85, #92, #203, #239, #330, #387, #454, #471, #478, #507, #534, #562 e #575, più #101.

## 2. L'unico lavoro di dettatura arrivato in main: la lingua (via #114, rifinito dal maintainer)

`git log -- src/core/dictation.ts` mostra un solo commit d'autore esterno:
`3bb28e5` di Bartosz Chronowski (PR https://github.com/amosblomqvist/learnharness/pull/114),
"add persistent language selection in General settings". Poi il maintainer:

- `88266ea` *Merge PR114 dictation foundation for approved provider-row finishing*. Il messaggio dice:
  "the donor General-only language placement is explicitly superseded by the human-approved row
  BELOW the full-width Deepgram key … Foundation only, not a checked candidate."
- `fbafc5e`, `f5a655b`, `27e2770`, `8337836`: finiture del maintainer (picker ricercabile condiviso
  `components/settings/search-menu-list.tsx`, guida al setup, link alla chiave Deepgram).
- `0348897` *Merge reviewed Deepgram dictation language feature for 0.3.9*.

#101 (diegoPenguino) e #239 (Neurozepam) restano aperte e superate. Il maintainer non le ha
commentate. Ha chiuso invece le issue #100, #231 e #238 rimandando a 0.3.9:
"Settings -> Providers -> dictation language, below the Deepgram key".

**Pattern d'integrazione del maintainer:** fa il merge locale di un commit piccolo, conserva
l'autore, rifinisce la UX secondo una decisione "human-approved", poi rilascia. Lo stesso schema
appare in #332 ("I integrated it locally with the paperclip moved to the right next to 'send'").
`CONTRIBUTING.md` lo formalizza: "We may integrate a contribution with finishing changes while
preserving authorship … adopting part of a PR is not acceptance of the whole proposal."

### Formato settings scelto (quello vincente)

- Sezione top-level `dictation` in `src/core/settings.ts`: `interface DictationSettings { language?: DictationLanguage }`,
  con `dictation: Required<DictationSettings>` nella forma risolta. Le posizioni `general.dictationLanguage`
  (#101) e `general.transcriptionLanguage` (#239) **hanno perso**.
- UI in **Settings → Providers → Dictation**, sotto la riga della chiave Deepgram (`components/settings/keys-section.tsx`).
- Le aggiunte di campi non incrementano `CURRENT_SETTINGS_VERSION`, che va incrementata solo per cambi di forma che rompono la compatibilità.
- `src/core/dictation.ts` contiene la lista Nova-3 e `deepgramListenUrl(language)`, con il commento
  "Only language varies. Keep the owner's tuned streaming parameters."
- Chiave: `DEEPGRAM_API_KEY` in `src/core/auth-keys.ts`, token a vita breve da `app/api/stt/token/route.ts`.

## 3. Schede per PR e issue

| # | Cosa | Approccio / seam | Dimensione | Maintainer | Perché è ferma |
|---|---|---|---|---|---|
| [330](https://github.com/amosblomqvist/learnharness/pull/330) | ElevenLabs: dettatura + read aloud | `DictationEngine` adapter in `src/core/dictation-engine.ts` (Deepgram e ElevenLabs Scribe), `/api/stt/token` restituisce `{provider, token}`; TTS su `POST /api/tts` (proxy streaming mp3), `/api/tts/voices`, `/api/tts/models`; `src/core/speech-text.ts` per markdown→parlato; player per finestra con cache in memoria; nuova sezione Settings "Read aloud"; Esc chain estesa | 69 file | nessuna review/commento | due feature in una PR (STT + TTS), nuova integrazione senza discussion preventiva, ampia superficie UI (bottone Listen per blocco, auto-read) |
| [471](https://github.com/amosblomqvist/learnharness/pull/471) | Narrazione locale con Piper | helper nativo C++ (`native/speech/`) bundlato, processo gestito dal server Next; voci scaricate on demand da HF con SHA-256; route `/api/speech/*` dietro capability desktop; `src/core/narration*.ts` (proiezione pura del contenuto visibile); cache su disco; sezione Settings "Speech" | 69 file, tocca CI/release/electron | nessuna | build nativa + packaging per 3 OS non verificati (macOS/Windows), solo voci inglesi, sovrapposizione esplicita con #330 |
| [203](https://github.com/amosblomqvist/learnharness/pull/203) | Dettatura OpenAI (batch) | WAV limitato su Stop → `/api/stt/openai` → `gpt-transcribe`; provider in General settings; serializzazione delle scritture settings | 32 file, riscrive molto `components/dictation.tsx` (+248/−279) | nessuna | posizionamento in General (in contrasto con la decisione Providers), refactor ampio dell'hook, duplica il fix di #205 |
| [478](https://github.com/amosblomqvist/learnharness/pull/478) | Parakeet locale via sherpa-onnx | `dictation.provider` (`deepgram`\|`parakeet`), modello ~487 MB scaricato in data dir, `/api/stt/parakeet{,/transcribe}`, trascrizione su Stop lato server Node | 42 file, 8 commit separabili, issue #477 prima | nessuna (solo tester esterni) | pacchetto nativo: bug Windows (tar, symlink NSIS) trovati dalla community; peso del modello; licenza CC BY 4.0 |
| [575](https://github.com/amosblomqvist/learnharness/pull/575) | Groq + locale (transcribe.cpp) + Silero VAD | "Engine" in Providers → Dictation; chunk da 12–18 s con VAD lato server; catalogo di 29 modelli derivato da pi-voice | 82 file, +444 righe di lock | nessuna | la più grande; catalogo/raccomandazioni/licenze; binari nativi nel server standalone |
| [562](https://github.com/amosblomqvist/learnharness/pull/562) | llama.cpp / whisper.cpp utente | server di trascrizione gestito dall'utente (base URL + bearer); WAV su Stop → server app → endpoint; `components/dictation-engines.ts` | 32 file, riscrive `keys-section.tsx` (−254) | nessuna | chiede esplicitamente feedback sullo scope; si sovrappone a #50, #387 e #478 |
| [534](https://github.com/amosblomqvist/learnharness/pull/534) | Deepgram `mip_opt_out=true` | una riga in `deepgramListenUrl` + doc | 3 file | nessuna | cambia un default senza discussion; prezzo/compatibilità coi token temporanei non verificati. In main il parametro **manca ancora** |
| [101](https://github.com/amosblomqvist/learnharness/pull/101) | Lingua dettatura | `general.dictationLanguage`, fix replace-whole | 9 file | nessuna sulla PR; issue #100 chiusa con 0.3.9 | superata da #114 rifinita dal maintainer |
| [239](https://github.com/amosblomqvist/learnharness/pull/239) | Lingua trascrizione | `general.transcriptionLanguage`, `src/core/transcription.ts` | 14 file | nessuna; issue #238 chiusa con 0.3.9 | superata, come #101 |
| [15](https://github.com/amosblomqvist/learnharness/issues/15) | Astrazione STT (issue) | propone engine dietro `useDictation` (`start/pushAudio/stop`), Settings → Dictation, nessun fallback silenzioso, Deepgram resta disponibile | — | nessuna | resta aperta; ha generato le PR #50, #203, #562 e altre, ognuna su una parte |
| [315](https://github.com/amosblomqvist/learnharness/issues/315) | Multilingua | chiusa dall'autore dopo che un utente ha indicato l'opzione "Multilingual" (Deepgram `multi`) | — | nessuna | risolta da 0.3.9 |
| [189](https://github.com/amosblomqvist/learnharness/issues/189) | Chiave ElevenLabs STT+TTS | richiesta utente; l'autore di #330 risponde con la PR | — | nessuna | dipende da #330 |
| [607](https://github.com/amosblomqvist/learnharness/discussions/607) | TTS automatico (Discussion) | "ElevenLabs con API key e Voice ID" | — | nessuna risposta | aperta da un giorno, zero commenti |
| [587](https://github.com/amosblomqvist/learnharness/discussions/587) | Prompt di sistema in altre lingue | — | — | nessuna risposta | correlata a #156 e #457 (lingua app) |

Altri item STT rilevanti: #50 (WebGPU/CPU locale + chooser al primo avvio), #85 e #92 (on-device
Apple), #387 (Handy, **con fallback a Deepgram**, contro quanto chiede #15), #454 (Codex), #507 (scelta
microfono, che generalizza `dictation-preference.ts` a tutta la sezione `dictation`).

## 4. Direzione del maintainer: i plugin

1. **`CONTRIBUTING.md`** (`958328c`, 2026-11-04, cioè dopo quasi tutte le PR STT): "New features,
   integrations, and storage-format changes also need discussion first." e "Plugins are the intended
   home for optional capabilities that do not belong in core. The plugin infrastructure is still under
   development". In più: "Passing tests does not, by itself, establish that a proposed behavior belongs in the product."
2. **Issue #197** (commento del maintainer, 2026-10-27): "Core stays general: teaching, practice, and the shared
   learning record. Features specific to a learner's needs, subject, or workflow belong in optional plugins …
   Plugins would own their UI, prompts, tools, and data."
3. **Branch upstream `feature/plugin-architecture`**: 18 commit del maintainer dal 2026-10-31 al 2026-11-07,
   circa 300 file, ancora da mergiare. Contiene `docs/plugin-architecture.md`, `docs/plugin-authoring.md`,
   plugin ufficiali (flashcards, coding-questions, drawing pad) ed esempi. Per noi conta che:
   - la **dettatura resta host-owned**. I plugin la *consumano* (`ctx.ui.mountDictation({onTranscript})`,
     `components/plugin-dictation.tsx` riusa `useDictation` con `destination: "explicit"`), ma non esiste un
     punto d'estensione per **fornire** un motore STT;
   - non esiste nessuna API audio/TTS. Ci sono però le placement `top-bar` di sessione con `context.view`
     (snapshot congelato di `messages` e `transcript`), i backend fidati nel server Node con accesso alla rete,
     le pagine Settings per plugin e i dati persistenti in `<data>/plugins/<id>/data/`. Un plugin TTS
     "a pulsante" sembra fattibile su API v1. Non ci sono invece placement per singolo messaggio né hook "turno finito → leggi"
     (per auto-read servirebbero; da verificare su `plugin-session-extension-contract.md`);
   - tra i non-goal: "generic forms/UI/shortcuts framework", sandbox, marketplace.
4. **`docs/archive/vision.md`**: "Dictation is a requirement, not a luxury". Deepgram streaming con
   parametri tarati dal maintainer. "On-device whisper … deferred unless Deepgram disappoints". **Il TTS non compare mai.**

Lettura: lo STT è core (requisito della visione), ma la scelta di altri provider è stata rimandata. Il TTS è una
capacità opzionale non prevista dalla visione. Per lo statuto di CONTRIBUTING è il candidato tipico per un plugin.

## 5. Pattern riusabili

- **Adapter di motore dietro l'hook esistente**: il `DictationEngine` di #330 lascia invariati coda di frame,
  flush differito, token caldo e reconnect, e aggiunge un adapter per provider. È la forma richiesta da #15.
- **Sezione `dictation` top-level + riga in Providers → Dictation**: è la scelta già approvata. Un `dictation.provider`
  additivo (come in #478 e #575) si inserisce senza incrementare la versione.
- **Nessun fallback silenzioso tra provider**, errori che nominano il motore (#15, #562). Fa eccezione #387, che fa fallback.
- **Chiave solo lato server**, token a vita breve o proxy (Deepgram, #330 `/api/tts`).
- **Proiezione pura del testo parlabile** in `src/core/` (`speech-text.ts` di #330, `narration.ts` di #471): niente risposte
  o feedback nascosti, codice/matematica/tabelle annunciati con un marcatore. Si testa senza rete.
- **Ordine dell'Esc**: cancella dettatura → stop lettura → interrompi turno (#330, `docs/keyboard.md`); dettatura e lettura si escludono a vicenda.
- **Download di modelli**: SHA-256 e dimensione fissati, staging e attivazione atomica, URL immutabile (un HF pinnato a commit evita il guasto
  visto in #478 quando l'archivio upstream è stato ricaricato).
- **Commit piccoli e separabili** (#478, 8 commit): il maintainer integra commit, non PR intere.

## 6. Cose da evitare

- PR da 30 a 80 file che uniscono più feature (STT + TTS in #330), che riscrivono `components/dictation.tsx` (#203, #562, #575) o
  `keys-section.tsx`, o che toccano CI/electron/release (#471).
- Mettere le impostazioni in `general.*`: è già stato superato due volte.
- Clobber della semantica replace-whole nelle PUT delle sezioni settings (`src/server/settings-body.ts`). È riemerso in #101, #203 e #507.
  Chi aggiunge campi a `dictation` deve inviare l'intera sezione o fare merge.
- Toccare i parametri Deepgram tarati ("Keep the owner's tuned streaming parameters").
- Dipendenze native o modelli pesanti nel pacchetto (#471, #478, #575): bug Windows/NSIS, firma macOS, payload.
- Cambiare un default di privacy senza discussion (#534), anche quando ha senso.
- Aprire una PR senza Discussion o issue concordata: dal 2026-11-04 è requisito esplicito.

## 7. Una PR TTS in core ha possibilità di merge oggi?

**Valutazione (inferenza dalle evidenze, non certezza): basse.** Gli indizi vanno tutti nella stessa direzione:

- il TTS non compare nella visione, e CONTRIBUTING indica i plugin come casa delle capacità opzionali;
- il maintainer sta costruendo attivamente la piattaforma plugin (branch del 31/10–7/11);
- le due PR TTS esistenti (#330 e #471), complete e testate, non hanno ricevuto nessuna risposta in 7–10 giorni;
- nessuna PR STT di provider è stata integrata. L'unica feature di dettatura arrivata (la lingua) è stata
  ridotta a "foundation" e rifinita dal maintainer;
- la Discussion #607 sul TTS non ha risposte.

Cosa cambierebbe la valutazione: una risposta del maintainer in #607, un item STT/TTS "In progress" sul board
(non verificato, vedi limite sopra), oppure il merge di `feature/plugin-architecture` con un'API che esponga
audio o placement per messaggio.

**Raccomandazione per il fork:** prima di scrivere codice, il proprietario del fork (non un agente) chieda in
https://github.com/amosblomqvist/learnharness/discussions/607 se il TTS è in scope per il core o per un plugin. Nel
frattempo conviene progettare il TTS come **plugin su API v1** (top-bar action + backend con chiave + pagina
Settings), tenendo la logica di proiezione testo→parlato pura e portabile. Lato STT conviene un adapter piccolo dietro
`useDictation` più `dictation.provider` additivo, proposto prima via Discussion/issue e consegnato in commit separabili.
