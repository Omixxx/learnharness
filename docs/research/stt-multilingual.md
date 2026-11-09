# Streaming STT for mixed Italian / Japanese / English dictation

Research for [Omixxx/learnharness#2](https://github.com/Omixxx/learnharness/issues/2). Written 2026-11-09.
I made no live calls to any provider. Everything below comes from documentation, changelogs and pricing pages
read on 2026-11-09. Each claim is tagged:

- **[V]**: verified on a first-party page (URL given).
- **[S]**: secondary or aggregator source only. Confirm it before relying on it.
- **[I]**: my inference, not a stated fact.

All prices are list prices in USD. EUR figures are approximate (about 1 USD = 0.86 EUR).

## TL;DR

1. **Stay on Deepgram Nova-3 `multi` and add the missing knobs:**
   - `language_hint=it&language_hint=ja&language_hint=en`
   - `endpointing=100` (Deepgram's documented value for code-switching)
   - `keyterm=...` for terms such as "Deepgram"
   - drop the redundant `punctuate=true`

   Cost is $0.0092/min, plus $0.0013/min for keyterms. That is about 550 min/month for 5 EUR, and the
   ~200 EUR credit covers about 22k minutes.
2. **The Arabic output most likely comes from a Deepgram model change, not from our code [I, strongly supported].**
   - On **2026-10-23**, Deepgram switched `model=nova-3&language=multi` from a 10-language model, which had
     no Arabic, to a new 55-language model, which includes Arabic [V].
   - This app first sent `language=multi` on 2026-10-25. Before that the URL had no `language`, so it used
     the English default (git `3bb28e5~1`).
   - So every `multi` session the user ran used the 55-language model.
3. **Deepgram has no hard restriction to a language set.** `language_hint` only biases detection [V]. There
   are only two ways to keep Arabic out of the candidate set:
   - pin the old 10-language streaming model with `version=2026-01-27.9249` [V]. Deepgram does not document
     how long this pin will be supported, so treat it as a stopgap.
   - switch to **Flux Multilingual** (`flux-general-multi`) [V]. Its fixed set is en, es, fr, de, hi, ru,
     pt, ja, it, nl, which includes it, ja and en and excludes Arabic. It is also cheaper at $0.0078/min.
     It needs a `/v2/listen` integration, and two things there are unverified (see below).
4. **Best non-Deepgram fallback: Soniox real-time (`stt-rt-v6`).**
   - $0.12/hr (about $0.002/min) [V], with `language_hints` and an optional `language_hints_strict` [V].
   - Short-lived browser keys [V].
   - Mid-sentence code-switching is claimed only on marketing pages [S].
5. **Next step is an A/B test, not a verdict.** Use the user's three real failing sentences (see the test
   plan at the end).

## 1. Deepgram, detailed findings

### 1.1 Is it/ja in the Nova-3 streaming `multi` set?

- **Yes [V].** The Nova-3 "Multilingual" row on the
  [models & languages overview](https://developers.deepgram.com/docs/models-languages-overview) lists
  Italian (`it`), Japanese (`ja`) and Arabic (`ar`). It covers about 55 languages.
- The [code-switching doc](https://developers.deepgram.com/docs/multilingual-code-switching) says
  "Nova-3 multilingual supports 55 languages". It covers both batch and streaming WebSocket.
- **History [V]:** the [changelog of 2026-10-23](https://developers.deepgram.com/changelog/2026/10/23)
  says `model=nova-3&language=multi` "now serves a new model covering 55 languages", for batch and streaming.
  - The previous 10-language model can be pinned with `version=2026-01-27.9107` (batch) or
    `version=2026-01-27.9249` (streaming).
  - Deepgram's own pages give that 10-language set as en, es, fr, de, hi, ru, pt, ja, it, nl. The Flux
    Multilingual model lists the same set. Neither includes Arabic.
- Monolingual Nova-3 `it` and `ja` also exist [V].
  - The Italian streaming model was improved on 2026-09-22
    ([changelog](https://developers.deepgram.com/changelog)).
  - A monolingual model cannot transcribe mixed sentences, so it does not answer this ticket.

### 1.2 Can the language set be restricted or hinted?

- **Nova-3 `language_hint` [V]** ([Nova-3 language prompting](https://developers.deepgram.com/docs/nova-3-language-prompting),
  [code-switching doc](https://developers.deepgram.com/docs/multilingual-code-switching)):
  - The parameter is repeatable, for example `&language_hint=it&language_hint=ja&language_hint=en`.
  - It takes effect only with `language=multi`. Nova-2 does not support it, and an unsupported code returns
    HTTP 400.
  - It is a bias, not a filter. The docs say: "Hints bias detection; they do not restrict it, so the model
    still transcribes any supported language it hears."
  - Hints "stay fixed for the life of a streaming connection". Changing them means reconnecting.
  - For code-switching the docs say to "hint every language you expect, or send no hints".
  - Responses echo the hints in `languages_hinted` and report detected languages in `languages` and
    `words[].language`. Use these fields to log drift.
- **A hard restriction does not exist on Deepgram [V, absence].** The only ways to shrink the candidate
  set are the version pin (10 languages) or Flux Multilingual (10 languages).
- **Flux Multilingual [V]** ([Flux language prompting](https://developers.deepgram.com/docs/flux/language-prompting),
  [GA changelog 2026-04-15](https://developers.deepgram.com/changelog/2026/4/15)):
  - Endpoint: `wss://api.deepgram.com/v2/listen?model=flux-general-multi`.
  - `language_hint` is also a bias here. Unlike Nova-3, the hints can be replaced mid-stream with a
    `Configure` message.
  - The docs say: "Flux handles mid-sentence language switches natively."
  - The response shape differs from Nova-3: it uses `TurnInfo` events and has built-in turn detection.
    That means a new client protocol, not a URL tweak [I].
- **`detect_language`:** this is a separate feature, not code-switching. It was not researched further,
  because it picks one language per request and so does not fit mixed sentences [I].
- **Nova-2 `multi`** covers only Spanish and English [V], so it is irrelevant here.

### 1.3 Keyterm prompting with `multi`

- **Supported [V]** ([keyterm doc](https://developers.deepgram.com/docs/keyterm)):
  - The keyterm doc says it is "available for both monolingual and multilingual transcription using the
    Nova-3 Models". The [Dec 2025 announcement](https://deepgram.com/learn/deepgram-expands-nova-3-with-10-new-languages-and-multilingual-keyterm-prompting)
    says the same.
  - The query parameter is **`keyterm`** (singular, repeatable). `keyterms` is the JSON array used in a
    mid-stream `Configure` message.
  - Limits: up to 100 terms and 500 tokens. Deepgram recommends 20 to 50 terms.
  - Mid-stream `Configure` updates work on all Nova-3 streaming models, including multilingual ones
    (changelog 2026-10-02 and 2026-10-20).
  - A `Configure` update over the limit does not return an error. The stream silently stops and closes with
    code 1011 after about 30 s (changelog 2026-10-08), so check the list size before sending.
- **Cost [V]:** keyterms add $0.0013/min on top of transcription ([pricing](https://deepgram.com/pricing)).
- **Japanese terms:** the docs say nothing about native-script keyterms. Whether kana or kanji keyterms
  help is untested [I]. Worth trying in the A/B test, for example `keyterm=Deepgram` plus a few JLPT
  vocabulary words in kanji.
- **Flux keyterms, unresolved conflict:**
  - The keyterm doc says Flux supports keyterms, including mid-stream updates.
  - The 2026-10-02 changelog says sending `keyterms` "on a non-Nova-3 model returns a
    `KeytermsNotSupported` error".
  - This needs a test.

### 1.4 Why `multi` drifts to Arabic, and why words get misheard

- Deepgram publishes no known-issue note about drift to the wrong language.
- **[I] Most likely cause: the larger candidate set since 2026-10-23.**
  - The user's sessions all used the 55-language model (see TL;DR, point 2).
  - With no hints, the model auto-detects across all 55 languages, so short or noisy segments can be
    assigned to an unrelated language and written in its script.
  - Soniox documents the same failure mode on its own models: "the model may still choose the 'wrong'
    language and transcribe using the wrong script"
    ([Soniox language restrictions](https://docs.soniox.com/stt/concepts/language-restrictions)).
- **[I] Contributing factors in our config:**
  - **No `language_hint`.** Detection is unbiased across 55 languages.
  - **`endpointing=300`.** Deepgram recommends `endpointing=100` for code-switching
    ([code-switching doc](https://developers.deepgram.com/docs/multilingual-code-switching)). The
    endpointing reference doesn't say why 100 ms is better. My guess is that shorter segments give the
    model more chances to re-detect the language at a switch.
  - **No keyterms.** "Deepcam" for "Deepgram" is the classic case keyterm prompting targets [V for the
    feature, I for this case].
- The other misheard sentences ("Il filo nuovo si sta impigliando…", "Vieni qua, vieni tu") are pure
  Italian. Two things are plausible [I]:
  - The 55-language model is weaker on Italian than the dedicated `it` model.
  - With no hints, Italian is competing with Spanish, Portuguese and others.

  Both are testable with the A/B plan below.

### 1.5 Current params in `src/core/dictation.ts`

- `smart_format=true` already applies punctuation. The docs say: "If you've set `smart_format=true`, no
  need to also set `punctuate=true`" ([smart format](https://developers.deepgram.com/docs/smart-format)) [V].
  So `punctuate=true` is redundant but harmless.
- Japanese numerals formatting on Nova-3 multilingual and `flux-general-multi` was added on 2026-10-23 [V].
  Nothing documented suggests `smart_format` harms Japanese.
- `endpointing=300` vs the recommended 100 for code-switching: see 1.4. This is a UX trade-off, because
  shorter endpointing means more `speech_final` segments.
  - The code only buffers `is_final` text, so more segments should not change the joined result [I].
  - It does change when the turn is considered finished.

### 1.6 Auth (browser-safe, short-lived)

- The current flow is verified in code: `POST /v1/auth/grant` with `ttl_seconds: 300`, then the browser
  opens `new WebSocket(url, ["bearer", jwt])`. The code comments say this was verified live.
- Deepgram docs [V] ([token-based auth](https://developers.deepgram.com/docs/token-based-authentication)):
  - Tokens work for the "/listen WebSocket API", among others.
  - Default TTL is 30 s, maximum 3600 s.
  - The key that mints tokens needs Member permission or higher.
- **Unverified:** whether grant tokens work on **`/v2/listen`** (Flux), and with the `bearer` subprotocol.
  The token doc does not mention `/v2`. This must be tested before choosing Flux.

### 1.7 Deepgram pricing (streaming, pay-as-you-go) [V]

Source: [deepgram.com/pricing](https://deepgram.com/pricing)

| Product | $/min | ≈ min per 5 EUR (~$5.8) |
|---|---|---|
| Nova-3 Monolingual | 0.0077 | ~750 |
| Nova-3 Multilingual | 0.0092 | ~630 |
| Nova-3 Multilingual + Keyterm | 0.0105 | ~550 |
| Flux Multilingual | 0.0078 | ~740 |

- The existing ~200 EUR credit is worth about 22k minutes of Nova-3 multi with keyterms, or about 370 hours.
- **Unverified:** whether that credit expires. Check the Deepgram console.
- The silence streamed by the worklet is billed as audio [I: Deepgram bills audio duration].

## 2. Alternatives

| Provider / model | it+ja same sentence | Language steering | Streaming price | Browser short-lived auth |
|---|---|---|---|---|
| **Soniox `stt-rt-v6`** | ja and it supported in real-time [V]; mid-sentence switching claimed on marketing pages only [S] | `language_hints` (bias) and `language_hints_strict` (best-effort restriction; docs recommend a single language for strict mode) [V] | **$0.12/hr ≈ $0.002/min** [V] | Temporary API key, `usage_type: transcribe_websocket`, 1–3600 s, `single_use`, `max_session_duration_seconds` [V] |
| **AssemblyAI Universal-3.6 Pro Realtime** | ja and it supported; docs say "the only streaming model with native (mid-sentence) code switching" [V] | `language_codes` (bias), updatable mid-session [V] | $0.45/hr; keyterms included [V]. Idle WebSocket time is billed [V] | Temporary token from the server, passed as `token` query parameter [V] |
| AssemblyAI Universal-Streaming Multilingual | **no ja** (en, es, de, fr, pt, it), per-turn detection only [V] | n/a | $0.15/hr [V] | same as above |
| **Speechmatics Melia 1** (real-time) | 56 languages, mid-sentence switching, no language chosen up front [V] | not researched | $0.40/hr list, $0.27/hr with the training-data opt-in [V]; $100 signup credit [V] | not verified (JWT temporary keys exist in Speechmatics docs but not confirmed here) [S] |
| ElevenLabs Scribe v2 Realtime | 90+ languages [V]; code-switching claimed only by third parties [S] | `language_code` (single language, optional) [V] | $0.39/hr, +$0.08/hr keyterms [V] | Single-use token `POST /v1/single-use-token/realtime_scribe`, 15 min, as query parameter [V] |
| OpenAI realtime transcription | not verified for mixed it+ja | `language`, plus a `languages` list on newer models [S] | `gpt-realtime-whisper` ~$0.017/min [S] (above budget) | Ephemeral client secret (`/v1/realtime/client_secrets`) [S] |
| Gladia real-time | `code_switching: true` with an optional list of expected languages [S] | language list narrows detection [S] | figures conflict, $0.25–0.75/hr [S] | not researched |

Sources:
- Soniox: [pricing](https://soniox.com/pricing), [supported languages](https://docs.soniox.com/stt/concepts/supported-languages),
  [language restrictions](https://docs.soniox.com/stt/concepts/language-restrictions),
  [language hints](https://soniox.com/docs/stt/core-concepts/language-hints),
  [Web SDK / temp keys](https://soniox.com/docs/sdk/web-SDK),
  [API reference](https://soniox.com/docs/api-reference)
- AssemblyAI: [multilingual streaming](https://www.assemblyai.com/docs/streaming/multilingual-transcription.md),
  [pricing](https://www.assemblyai.com/pricing),
  [streaming API](https://www.assemblyai.com/docs/api-reference/streaming)
- Speechmatics: [pricing](https://www.speechmatics.com/pricing)
- ElevenLabs: [API pricing](https://elevenlabs.io/pricing/api),
  [realtime STT reference](https://elevenlabs.io/docs/api-reference/speech-to-text/v-1-speech-to-text-realtime),
  [single-use token](https://elevenlabs.io/docs/api-reference/tokens/create)
- OpenAI: [transcription sessions](https://developers.openai.com/api/reference/resources/realtime/subresources/transcription_sessions/methods/create),
  [gpt-transcribe model page](https://developers.openai.com/api/docs/models/gpt-transcribe)
- Gladia: [code-switching](https://www.gladia.io/product/audio-intelligence/code-switching)

Notes on the table:
- Every option except OpenAI's live model fits within 5 EUR/month at hobby volume.
- Soniox is cheapest by about 4x vs Deepgram.
- Only Deepgram uses the existing credit.

## 3. Ranked options

1. **Deepgram Nova-3 `multi` with hints, `endpointing=100` and keyterms.**
   - It needs only URL changes in `deepgramListenUrl` and keeps the same auth.
   - It uses the existing credit.
   - Risk: hints are only a bias, so Arabic drift may continue.
2. **The same, plus `version=2026-01-27.9249`.**
   - This pins the old 10-language set, which has no Arabic.
   - It is still a URL-only change.
   - Risk: Deepgram has not documented how long the pin is supported, so it is a stopgap.
3. **Deepgram Flux Multilingual (`flux-general-multi`, `/v2/listen`).**
   - Its 10-language set has no Arabic, it supports native code-switching and mid-stream hints, and it is
     cheaper.
   - It needs a new client protocol (`TurnInfo`).
   - Two things must be tested first: grant tokens on `/v2`, and the keyterm conflict.
4. **Soniox `stt-rt-v6`.**
   - It is the cheapest option and has browser-safe temporary keys and hints, with an optional strict mode.
   - It needs a new provider integration and a new key, and it doesn't use the Deepgram credit.
   - Its code-switching quality is unverified.
5. **AssemblyAI Universal-3.6 Pro Realtime.**
   - It is the only provider whose docs explicitly state mid-sentence code-switching that includes ja.
   - It costs $0.45/hr, which is still under 5 EUR at hobby volume, and idle socket time is billed.
6. Speechmatics Melia 1, ElevenLabs Scribe v2 Realtime, OpenAI, Gladia: the evidence is weaker or the
   price is higher. Not recommended as a first move.

## 4. A/B test plan (owner runs it; no calls made here)

Test set:
- the user's three failing samples: "Il filo nuovo si sta impigliando nei solchi della roccia",
  "Vieni qua, vieni tu", and a sentence containing "Deepgram"
- 3–5 mixed it/ja/en sentences (for example "Oggi ho studiato 勉強 e la parola 'meeting'")

Record each sample once as 16 kHz linear16 and replay the same audio to every config, so the comparison is
fair.

| Run | URL delta vs today |
|---|---|
| A (baseline) | today's URL (`language=multi`, `endpointing=300`) |
| B | `&language_hint=it&language_hint=ja&language_hint=en&endpointing=100&keyterm=Deepgram`, without `punctuate` |
| C | B + `&version=2026-01-27.9249` |
| D | Flux: `wss://api.deepgram.com/v2/listen?model=flux-general-multi&language_hint=it&language_hint=ja&language_hint=en` (+ `keyterm=Deepgram` if accepted) |

What to record for each run:
- how faithful the transcript is
- whether any non-it/ja/en script appears
- `languages` / `words[].language`
- how responsive it feels

Consider Soniox only if C and D both fail.

## Open questions (unverified)

- How long `version=2026-01-27.9249` stays available.
- Whether grant tokens and the `bearer` subprotocol work on `/v2/listen` (Flux).
- Whether keyterms work on Flux (the docs and the changelog conflict).
- Whether keyterms written in kana or kanji help Japanese recognition.
- Whether the Deepgram promotional credit expires.
- How good Soniox's mid-sentence it/ja switching really is.
