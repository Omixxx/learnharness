# TTS provider for pronouncing Japanese (and other study-language) passages

Ticket: [Omixxx/learnharness#3](https://github.com/Omixxx/learnharness/issues/3) (part of #1).
Researched 2026-11-09 from documentation and primary sources only. No live calls were made with the user's key. Every source below was accessed 2026-11-09 unless another date is given.

**Use case.** A Japanese learner clicks a speaker button next to a short Japanese phrase the teacher wrote. The phrase plays at normal speed or slowed. Nothing autoplays. The budget is about 5 EUR/month (≈ US$5.8). Deepgram is preferred: the user has about 200 EUR of credit and an existing `DEEPGRAM_API_KEY`. `app/api/stt/token/route.ts` already mints `/v1/auth/grant` tokens.

---

## TL;DR

1. **Deepgram is viable, under two conditions.** It supports Japanese and Italian in Aura-2 (`/v1/speak`) and in Flux TTS (`/v2/speak`, Deepgram's current recommended model). For Japanese, Deepgram has **no documented native speed control** and **no reading/pronunciation override**. Those exist only for en/es (Aura-2) or English (Flux). So with Deepgram:
   - slowed playback is client-side `playbackRate` with `preservesPitch = true` (the browser default);
   - wrong kanji readings can only be fixed by changing the input text, for example by sending kana.

   If both are acceptable, use Deepgram. Prefer **Flux** (`flux-*-ja`), and fall back to Aura-2 (`aura-2-*-ja`) if the A/B test favours it. Cost is negligible, and the existing credit covers it if it applies to TTS (unverified, see §1.3).
2. **Call it through a Next.js server route, not from the browser.** Grant tokens are authorized for `/speak` (REST and WS). However, Deepgram's JS SDK says browser REST calls need a proxy because of CORS. A server route that holds the key, like the STT token route, removes the CORS problem. It also makes the Flux `/v2` token question moot and allows caching.
3. **If natively slowed speech or reading overrides are required, Azure Speech is the strongest fallback.**
   - Neural ja-JP voices.
   - SSML `prosody rate` from 0.5× to 2×.
   - Katakana `sapi` phonemes for ja-JP.
   - Short-lived STS tokens.
   - 0.5 M chars/month free, then US$15/1M.

   Google Chirp 3 HD and Amazon Polly are close behind.
4. **Rule out:**
   - OpenAI `/v1/audio/speech`: the TTS models shut down 2027-01-06.
   - Web Speech API as the primary path: voice availability varies by OS, and on Linux there are often no voices.
   - Piper: no Japanese voices.
   - ElevenLabs: the paid tier exceeds budget.
   - VOICEVOX: useful only as a local reference.

---

## 1. Deepgram (preferred)

### 1.1 Models, languages, voices — verified

| Model | Endpoint | ja voices | it voices | Notes |
|---|---|---|---|---|
| **Aura-2** | `POST /v1/speak` (REST), `wss://…/v1/speak` (WS) | `aura-2-fujin-ja`, `aura-2-izanami-ja`, `aura-2-uzume-ja`, `aura-2-ebisu-ja`, `aura-2-ama-ja` (featured: fujin, izanami) | `aura-2-{melia,elio,flavio,maia,cinzia,cesare,livia,dionisio,demetra}-it` (featured: livia, dionisio) | Languages: en, es, de, fr, nl, it, ja. Described as "Our previous-generation model". |
| **Flux TTS** | `/v2/speak` (WS streaming + batch REST) | `flux-ayumi-ja` (F), `flux-hayato-ja` (M), `flux-nozomi-ja` (F), `flux-sayaka-ja` (F) | `flux-{angela,chiara,federico,giorgio,matteo,paola,rocco,sofia,stefano,vincenzo,vito}-it` | 42 languages. **Text normalization** (numbers, dates, abbreviations) is available for 10 of them, including **ja and it**. The rest are Early Access without normalization. Recommended "for all new builds". |
| Aura (v1) | `/v1/speak` | — | — | English only. |

Sources:
- <https://developers.deepgram.com/docs/tts-models>
- <https://developers.deepgram.com/docs/tts-models-languages-overview>
- <https://developers.deepgram.com/docs/flux-tts/voices.md>
- Japanese and Italian were added to Aura-2 in Dec 2025: <https://deepgram.com/learn/aura-2-now-speaks-dutch-french-german-italian-japanese>

### 1.2 API shape — verified

**Aura-2 REST** (`POST https://api.deepgram.com/v1/speak?model=…`, JSON body `{ "text": "…" }`), from <https://developers.deepgram.com/reference/text-to-speech/speak-request>:
- `encoding`: `linear16`, `flac`, `mulaw`, `alaw`, `mp3` (default), `opus`, `aac`.
- `container`: `none`, `wav`, `ogg`.
- `sample_rate`: `mp3` is 22050. `opus` is 48000.
- `bit_rate`: 32000 or 48000 for mp3.
- `speed` (default 1): "Not yet supported in all languages."
- Auth: `Authorization: Token <key>` or `Authorization: Bearer <JWT>`.

**Flux batch** (`https://api.deepgram.com/v2/speak?model=flux-…-ja`), from <https://developers.deepgram.com/docs/flux-tts/batch.md>:
- `encoding`: mp3 (default), opus, flac, aac, linear16, mulaw, alaw.
- `speed`: 0.5–1.5 in 0.05 steps. Unsupported model/language combinations return `SPEED_NOT_SUPPORTED`.
- SSML or competitor markup is stripped, with an `INPUT_MARKUP_STRIPPED` warning (<https://developers.deepgram.com/docs/flux-tts/feature-overview.md>).

**Speed and pronunciation controls by language**, from <https://developers.deepgram.com/docs/tts-voice-controls>:
- Aura-2: "Aura-2 speed and pronunciation are available in English and Spanish". The Aura-2 speed range is 0.7–1.5.
- Flux: "Flux TTS controls are available in English". Flux pronunciation (IPA inline) is English-only and Early Access.
- **Japanese is not mentioned**, so treat native speed and pronunciation overrides as unavailable for ja and it.

### 1.3 Price — verified

From <https://deepgram.com/pricing>:

| Model | Pay-as-you-go | Growth |
|---|---|---|
| Aura-2 | US$0.030 / 1k chars | US$0.027 / 1k chars |
| Aura-1 | US$0.015 / 1k chars | US$0.0135 / 1k chars |
| Flux TTS | US$0.045 / 1k chars | US$0.045 / 1k chars |

- Flux promo: "$1 in credits for every $1 spent, up to $500, through Dec. 31, 2026."
- Free credit: pay-as-you-go lists a "Free $200 credit". The page's FAQ example is STT. **Whether the user's existing credit is spendable on TTS is not stated. Check the Deepgram console billing page.**

### 1.4 Auth: can the browser call `/speak` with the grant token? — verified + inference

- **Verified.** Temporary tokens from `/v1/auth/grant` carry `usage::write` for `/listen` REST+WS, **`/speak` REST+WS**, `/read`, and `/agent`. They do not work for Management APIs.
  - Default TTL is 30 s. `ttl_seconds` can raise it to a maximum of 3600.
  - Send the token as `Authorization: Bearer <JWT>`.
  - Source: <https://developers.deepgram.com/guides/fundamentals/token-based-authentication>.
- **Verified.** The Deepgram JS SDK README says: "Due to CORS header restrictions in the Deepgram API, you must use a proxy server when making REST API calls from browsers." The same README lists live TTS over WebSocket as a direct `wss://api.deepgram.com` connection. Source: <https://github.com/deepgram/deepgram-js-sdk>.
- **Not verified.** Whether grant tokens cover Flux `/v2/speak`. The token page says only "`/speak`".
- **Inference.**
  - Browser-direct is possible only via the `/v1/speak` WebSocket. Short clips do not need streaming.
  - Simplest and safest: a Next.js route handler (e.g. `app/api/tts/route.ts`) calls Deepgram with the server-held key, like `app/api/stt/token/route.ts` does, and returns audio bytes.
  - This avoids CORS. It also works for `/v2`, keeps the key out of the renderer, and lets the server or renderer cache clips.

### 1.5 Budget arithmetic — inference

- 5 EUR ≈ US$5.8, which is about 190k chars/month on Aura-2 or about 130k chars/month on Flux.
- A short Japanese phrase is about 20–40 characters, so that is thousands of plays per month before caching.
- Cache each clip by a hash of (text, voice, model). Then:
  - a replay is free;
  - a slowed replay is free (it is the same clip played with `playbackRate`).
- Any provider here fits the budget, and the 200 EUR credit would last years if it applies to TTS.

### 1.6 Japanese quality — not verified

No first-party benchmark of Deepgram's Japanese accuracy (kanji readings, pitch accent) was found. Flux documents Japanese text normalization. Aura-2's normalization for ja is not stated. **Recommendation:** run a short listening A/B test with the user's own key, outside this research. Use about 10 phrases with tricky readings, for example:
- 今日 (きょう/こんにち)
- 一日 (ついたち/いちにち)
- 上手
- 生 (various readings)
- numbers and counters

Compare `flux-*-ja`, `aura-2-*-ja`, and one Azure or Google free-tier clip as a control.

---

## 2. Slowed playback: native rate vs client `playbackRate`

| Provider | Native slow-down for ja | Source |
|---|---|---|
| Deepgram Aura-2 / Flux | **Not documented for ja** (en/es or en only) | <https://developers.deepgram.com/docs/tts-voice-controls> |
| Azure | SSML `<prosody rate>` from 0.5× to 2×. Also `x-slow` = 0.5 and `slow` = 0.64. | <https://learn.microsoft.com/en-us/azure/ai-services/speech-service/speech-synthesis-markup-voice> (updated 2026-08-18) |
| Google Chirp 3 HD | Pace from 0.25 to 2.0 | <https://docs.cloud.google.com/text-to-speech/docs/chirp3-hd> |
| Amazon Polly | SSML `prosody rate` (standard SSML) | Polly SSML docs |
| OpenAI | Steerable by prompt (`instructions`). Being shut down, see §3. | <https://developers.openai.com/api/docs/guides/text-to-speech> |
| VOICEVOX (local) | `speedScale` in the audio query | <https://github.com/VOICEVOX/voicevox_engine> |

- **Client side — verified.** `HTMLMediaElement.preservesPitch` defaults to `true`, so changing `playbackRate` time-stretches without changing pitch. It is Baseline, widely available since Dec 2023. Source: <https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/preservesPitch>.
- **Inference, no source found.** Time-stretched audio at about 0.5–0.75× is likely to sound less natural than speech synthesized slower, and it may smear consonants and morae. For a learner who wants to hear mora timing, that may matter. A 0.75× client rate is a reasonable "slow" default if Deepgram is chosen. This is a design choice for the human.

---

## 3. Alternatives with Japanese

All prices are list prices in USD as of 2026-11-09.

| Provider | ja voices | Price | Free tier | Reading override for kanji | Browser auth |
|---|---|---|---|---|---|
| **Azure Speech** | Neural ja-JP voices. Multilingual voices (e.g. `en-US-AvaMultilingualNeural`) also speak ja-JP. | Neural **US$15 / 1M chars** (Azure Retail Prices API, `S1 Neural Text To Speech Characters`, eastus, effective 2024-02-01) | **0.5M chars/month** (F0) | **Yes.** `<phoneme alphabet="sapi" ph="…katakana…">` for ja-JP (katakana-based phone set), plus custom lexicons | STS token from `/sts/v1.0/issueToken`, valid 10 min, sent as `Authorization: Bearer` |
| **Google Cloud TTS** | Chirp 3 HD ja-JP, Neural2/WaveNet ja-JP | Chirp 3 HD **US$30/1M**; Neural2 US$16/1M; WaveNet and Standard US$4/1M | Chirp 3 HD **1M/month**; WaveNet and Standard **4M/month**; Neural2 1M/month | **Yes.** Japanese `<phoneme>` "must be specified as `yomigana`" (SSML). Chirp 3 HD SSML is **Preview** and not available for streaming. | API key or OAuth on the server (inference: proxy) |
| **Amazon Polly** | ja-JP: Mizuki (standard), Takumi (standard + neural), Kazuha (neural), Tomoko (neural). No generative ja voice. | Neural US$16/1M; Standard US$4/1M | Neural 1M/month **first 12 months only**; Standard 5M/month | **Yes, most explicit.** `x-amazon-yomigana` (kana or romaji) and `x-amazon-pron-kana` (encodes pitch accent) | AWS SigV4 (inference: server proxy) |
| **ElevenLabs** | Multilingual v2 / v3 cover ja | Flash/Turbo US$0.04/1k; Multilingual v2 / v3 US$0.08/1k; Starter **US$6/month** for 75k chars | about 10k chars/month (inferred from page layout); commercial use unclear | Not researched | Server proxy |
| **OpenAI** `gpt-4o-mini-tts` / `tts-1` | ja is supported, but voices are "optimized for English" | US$0.60/1M text + US$12/1M audio tokens; `tts-1` US$15/1M chars | — | Prompt only | — |

Notes on the table:
- **OpenAI is out.** The docs say `tts-1`, `tts-1-hd` and `gpt-4o-mini-tts` "are deprecated and will shut down on January 6, 2027".
- **Gemini TTS** is also on the Google pricing page (token-priced, preview). It was not evaluated.

Sources:
- Azure pricing: <https://azure.microsoft.com/en-us/pricing/details/speech/> and the Retail Prices API `https://prices.azure.com/api/retail/prices` (filter `productName eq 'Azure Speech'`).
- Azure pronunciation: <https://learn.microsoft.com/en-us/azure/ai-services/speech-service/speech-synthesis-markup-pronunciation> and <https://learn.microsoft.com/en-us/azure/ai-services/speech-service/speech-ssml-phonetic-sets> (both updated 2026-06-05).
- Azure REST and auth: <https://learn.microsoft.com/en-us/azure/ai-services/speech-service/rest-text-to-speech> (ms.date 2026-05-21).
- Google pricing: <https://cloud.google.com/text-to-speech/pricing>. Google SSML: <https://docs.cloud.google.com/text-to-speech/docs/ssml>. Chirp 3 HD: <https://docs.cloud.google.com/text-to-speech/docs/chirp3-hd>.
- Polly pricing: <https://aws.amazon.com/polly/pricing/>. Polly voices: <https://docs.aws.amazon.com/polly/latest/dg/available-voices.html>. Polly Japanese phonemes: <https://docs.aws.amazon.com/polly/latest/dg/ph-table-japanese.html>.
- ElevenLabs: <https://elevenlabs.io/pricing/api>.
- OpenAI: <https://developers.openai.com/api/docs/pricing> and <https://developers.openai.com/api/docs/guides/text-to-speech>.

---

## 4. Web Speech API (`speechSynthesis`) in Electron

- **Verified.** On Linux, Chromium's TTS backend (`content/browser/speech/tts_linux.cc`) dynamically loads `libspeechd.so.2` (speech-dispatcher). It enumerates voices only if that library loads and a connection opens. Without speech-dispatcher and an installed output module, `getVoices()` returns nothing. Source: <https://chromium.googlesource.com/chromium/src/+/refs/heads/main/content/browser/speech/tts_linux.cc>.
- **Verified (issue reports).**
  - [electron/electron#586](https://github.com/electron/electron/issues/586), "Speech Synthesis: No Voices", 2014.
  - [electron/electron#22844](https://github.com/electron/electron/issues/22844), "`getVoices()` returns an empty array", 2020. It was also reproduced on macOS, and the voices appear only after the `voiceschanged` event fires.
- **Inference.**
  - macOS and Windows expose OS voices, for example a Japanese system voice if the language pack is installed. Linux typically exposes espeak-ng, which is poor for Japanese.
  - The "Google 日本語" network voices are a Google Chrome feature, not something Chromium or Electron ship.
  - Result: Web Speech is free but unpredictable per machine. It suits a last-resort fallback, not the primary path.

## 5. Local engines (reference)

- **VOICEVOX Engine.**
  - Japanese-only HTTP server on `127.0.0.1:50021`. The flow is `POST /audio_query`, then `POST /synthesis`, which returns 24 kHz WAV.
  - `speedScale` controls rate.
  - Licence: LGPL-3.0, dual-licensed. Runs on Windows, Linux and macOS, on CPU or NVIDIA GPU.
  - Per-voice/character credit terms were **not** on the engine README; they live on the VOICEVOX site.
  - Inference: the audio query exposes accent phrases and moras, so readings and pitch accent can be edited. This is valuable as a quality reference.
  - Source: <https://github.com/VOICEVOX/voicevox_engine>.
- **Piper.** **No Japanese voice** is listed. Italian (it_IT) is. Source: <https://github.com/OHF-Voice/piper1-gpl/blob/main/docs/VOICES.md>.

## 6. Kanji reading ambiguity and mixed-language text

**Verified:**
- Deepgram models are per-language (`*-ja`, `*-it`). Neither the docs nor the voices page describe handling of mixed-language input or code-switching.
- Google `<lang>`: "Japanese with Kanji is not supported" there. The text is "transliterated and read as Chinese characters". So a Japanese span inside a non-ja voice is read wrongly.
- Azure multilingual voices autodetect language and accept `<lang xml:lang="ja-JP">` at the word or sentence level. Non-multilingual voices don't support `<lang>`.
- Reading overrides for Japanese exist in:
  - Azure: `sapi` katakana phonemes.
  - Google: `yomigana`.
  - Polly: `x-amazon-yomigana` and `x-amazon-pron-kana`.
- Reading overrides do **not** exist in Deepgram for ja.

**Inference and open design questions for the human. These are not decided here.**
- **Send only the Japanese span.** The speaker button should send only the Japanese passage to a ja voice, not the surrounding Italian sentence. That needs the passage to be marked up, which is a decision for the teacher-output format.
- **Kana reading next to kanji.** The teacher could emit a kana reading alongside each kanji passage, e.g. `{text: "今日は", reading: "きょうは"}`. Sending the reading, or a mix where only ambiguous words are replaced, sidesteps kanji ambiguity on any provider, including Deepgram. The cost may be some loss of naturalness: pitch accent and word boundaries are inferred from kana alone. On Azure, Google or Polly the reading could instead go into a `<phoneme>` tag around the kanji.
- Which approach to take, and whether the teacher reliably produces correct readings, needs the human's decision.

---

## 7. Ranked recommendation

1. **Deepgram Flux TTS (`flux-*-ja`, `flux-*-it`), with Aura-2 `aura-2-*-ja` as an A/B alternative.**
   - Called via a Next.js server route that holds the key.
   - mp3 or opus output, cached per text+voice.
   - Slow mode is client `playbackRate` (e.g. 0.75) with `preservesPitch`.
   - Uses the existing key and credit, and fits the budget easily.
   - Condition: the human accepts client-side slow-down and no reading override. Kana input could mitigate the missing override.
2. **Azure Speech (ja-JP neural voice).**
   - Choose it if natively slowed synthesis or kanji reading overrides are required.
   - 0.5M chars/month free is likely enough on its own. Prosody rate 0.5–2×, katakana phonemes, browser-capable STS tokens.
3. **Google Cloud TTS Chirp 3 HD (or WaveNet).**
   - 1M (Chirp 3 HD) or 4M (WaveNet) chars/month free, and pace from 0.25 to 2.0.
   - `yomigana` phonemes exist, but SSML on Chirp 3 HD is Preview.
4. **Amazon Polly Neural (Takumi, Kazuha, Tomoko).**
   - Has the most explicit kanji override (`pron-kana` with pitch accent).
   - The neural free tier lasts 12 months only.
5. **Web Speech API** as an offline, zero-cost fallback only. It is unreliable on Linux.
6. Not recommended:
   - ElevenLabs: over budget for paid use.
   - OpenAI: shuts down 2027-01-06.
   - Piper: no ja.
   - VOICEVOX: a local reference only.

## 8. Verified vs inference summary

- **Verified from primary docs:**
  - Deepgram ja/it voices, endpoints, formats and prices.
  - Grant tokens cover `/speak`.
  - The SDK says browser REST needs a proxy (CORS).
  - Deepgram speed and pronunciation controls are en/es (Aura-2) or en (Flux) only.
  - Azure, Google, Polly, ElevenLabs and OpenAI prices and free tiers.
  - Azure `prosody rate` range.
  - Japanese phoneme alphabets: Azure `sapi` katakana, Google `yomigana`, Polly yomigana and pron-kana.
  - Google `<lang>` reads kanji as Chinese.
  - OpenAI TTS shutdown date.
  - Chromium on Linux uses speech-dispatcher.
  - Piper has no ja.
  - VOICEVOX API and licence.
- **Inference or unverified:**
  - Whether grant tokens cover `/v2/speak`.
  - Whether the existing credit applies to TTS.
  - Relative Japanese quality of Deepgram vs Azure, Google or Polly; no benchmark was found.
  - Naturalness cost of time-stretching.
  - Web Speech voice availability on macOS and Windows.
  - Chrome-only Google network voices.
  - The kana-input workaround's effect on naturalness.
