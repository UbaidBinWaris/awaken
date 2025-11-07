## awAken Workflow – Client Guide

This guide explains in simple terms what the system does and how to use it. You can share it directly with your client.

---

## What It Does

1. You send a short topic (for example: “healing for my friend”) to a Telegram bot.
2. The system writes a short, gentle prayer based on your topic.
3. You can press:
   - Accept → continue to production
   - Revise → get a new version
4. After Accept: it creates audio in Polish, English and Chinese, makes subtitles, adds a soft background video and music, and produces short videos.
5. It saves everything (audio, subtitles, videos) to Google Drive and logs all links in Google Sheets.
6. You receive the final video(s) or a download link back in Telegram.

---

## How To Use (3 Steps)

1) Send your topic in one message on Telegram.
2) Review the prayer. If you like it, press Accept. If not, press Revise to try again.
3) Wait a short time for processing. You’ll receive the finished media or a download link.

Tip: Keep the topic short and specific for the best results.

---

## What You Receive

In Google Drive (organized per request):
- Audio (MP3): Polish, English, Chinese
- Subtitles (SRT): Polish, English, Chinese
- Videos (MP4): Polish, English, Chinese

In Google Sheets:
- A new row with message ID, your topic, and links to all files.

In Telegram:
- The generated prayer with Accept / Revise buttons
- The finished video(s) or a Drive download link

---

## Notes & Limits

- If a video is too large to send directly via Telegram, you’ll get a Drive link instead.
- Background video and music are gentle and neutral by default; we can customize style on request (fonts, colors, music volume, logo).
- Multi-language output is automatic. If you want only certain languages, tell us and we’ll adjust.

---

## Quick FAQ

- Can I change the text? Yes. Press Revise until you’re happy, then press Accept.
- How long does it take? Usually a couple of minutes after pressing Accept.
- Can I use only the audio? Yes. Use the MP3 files from the Drive folder.
- Are the links shareable? Yes, if the Drive folder has share permissions enabled.

---

## Need help?

Reply in the same Telegram chat, or contact your account manager and we’ll assist.

If you’d like a one-page PDF summary or a visual flowchart to share with stakeholders, we can provide it on request.## awAken workflow — detailed walkthrough

This document explains every step/node in `workflow.json` (n8n workflow) in plain language. It describes the purpose of each node, key parameters, how data flows between nodes, decision points, and expected inputs/outputs.

---

## High-level summary

- Trigger: A Telegram message or callback (button press) starts the workflow.
- Input parsing: A `Code` node extracts chat id, message text, message id and button callback data (Accept/Revise).
- Decision routing: An `If` node checks whether the input is a normal message or a callback, and whether the callback is `Accept` or `Revise`.
- AI generation: An AI agent (LangChain node + OpenAI model) composes a prayer text based on a detailed prompt template.
- Clean & log: The generated text is cleaned, then appended to Google Sheets for record-keeping.
- TTS & assets: The cleaned text is used to create audio files (Polish/English/Chinese) using ElevenLabs (or Eleven-like) HTTP requests, saved to Google Drive, transcribed into subtitles (SRT), and exported as files.
- Video creation: A random stock video is selected from Drive, audio is combined, subtitles burned-in using `ffmpeg`, background music mixed, final videos uploaded to Drive, and Telegram messages are sent with links or the file (if small enough).

## Contract (inputs / outputs / success criteria)

- Inputs:
  - Telegram message or callback via the `Telegram Trigger` node.
  - For AI: topic text found in the parsed `Code` node (`text` field).
- Outputs:
  - Google Sheets rows containing generated content, audio and video links.
  - Google Drive files: MP3, SRT, and final MP4 for Polish, English and Chinese.
  - Telegram messages to the originating chat with either the generated message (and inline buttons) or final video download/view links.
- Success criteria:
  - AI produces a prayer text (approx 40–60 words) following the prompt.
  - Audio files produced by ElevenLabs endpoints and uploaded to Drive.
  - Subtitles (SRT) created and saved.
  - Final videos produced and uploaded; Telegram notified.

---

## Quick flow (textual)

1. `Telegram Trigger` → 2. `Code` (parse) → 3. `If1` (message vs callback) →
- Message path: AI Agent → Code4 (clean) → Google Sheets → Generate Audio nodes → ElevenLabs → Drive → transcribe & SRT → write SRT files → select video & music → ffmpeg steps → upload → Google Sheets rows and Telegram notifications
- Callback path (Accept / Revise): Accept triggers an `If` which edits the message and proceeds to folder creation & media generation; Revise will re-run AI generation path.

---

## Node-by-node detailed explanation

Below each node is explained: type, purpose, important parameters, and how it connects to others.

### Triggers & Input parsing

- `Telegram Trigger` (type: `telegramTrigger`)
  - Purpose: Receives incoming Telegram updates (message or callback_query).
  - Key parameter: `updates` set to `["message","callback_query"]`.
  - Output: raw Telegram payload used by `Code` node.

- `Code` (id: `0b0cefe3-...`, type: `code`)
  - Purpose: Normalize Telegram input into a small JSON object with `chatId`, `text`, `message_id`, and `data` (callback label like `Accept`/`Revise`).
  - Logic: If there's a `message` payload it extracts chat id and text; if there's a `callback_query`, it extracts the callback data.
  - Output example: `{ chatId: 12345, text: "topic text", message_id: 6789, data: "Accept" }`.

### Routing logic

- `If1` (type: `if`)
  - Purpose: Uses conditions to route between a new message or a callback action.
  - Conditions include checking whether a message exists or callback data equals `Revise`. It uses combinator `or` so either condition can trigger the first branch.
  - Main branches: the `true` branch goes to `AI Agent` for generation; the `false` branch goes to another `If` that checks for `Accept`.

- `If` (id: `4b8d9b75-...`, type: `if`)
  - Purpose: Checks whether callback `data` equals `Accept` — if yes it edits the message and proceeds to asset generation. If `Revise`, the AI agent is invoked again and a new draft is generated.

### AI generation

- `AI Agent` (LangChain agent node)
  - Purpose: Generate the prayer text using a long, prescriptive prompt. The prompt enforces tone, length, structure, and anchors/phrases to include (like "Najczystszego Źródła Światła i Miłości").
  - Inputs: `TOPIK` is injected via `{{ $('Code').item.json.text }}`. The prompt contains formatting rules and allowed images/emojis.
  - Connected to `OpenAI Chat Model` as its language model provider.

- `OpenAI Chat Model` (type: LangChain OpenAI wrapper)
  - Purpose: The LM used by the `AI Agent`. Configured to `gpt-4.1-mini` in the workflow.

### Post-processing / cleaning

- `Code4` (cleaning code)
  - Purpose: Removes footers like "This message was sent automatically with n8n" and trims whitespace. Returns `{ cleanedText }`.
  - This cleaned text is used for: sending to the user, TTS generation and spreadsheet logging (`Google Sheets` node).

### Google Sheets logging (initial)

- `Google Sheets` (named: `AWAKEN - GixiAI` / `Gixiai-content-data` sheet in the workflow)
  - Purpose: Append each generated piece to a spreadsheet with metadata: chatID, text, messageID, AI response and callback.
  - Important: Keeps an audit trail for content and subsequent audio/video rows.

### Creating a folder for assets

- `Create Audio Folder` (Google Drive folder creation)
  - Purpose: Create a folder per message (folder name uses the `message_id`) to store audio and subtitle assets.
  - Output: folder `id` used by subsequent Google Drive upload nodes.

### TTS / Audio generation

There are three parallel TTS flows, one per target language: Polish, English, Chinese.

- `Generate Audio (Polish)` (HTTP request to ElevenLabs-like endpoint)
  - Purpose: POST to ElevenLabs TTS endpoint with JSON body containing `text`, `model_id`, `language` and `voice_settings`.
  - Response: binary audio file (returned as HTTP file response). This is then saved by `upload polish subtitles1` / `Google Drive2`.
  - Note: The workflow includes a direct API key in the node; keep keys in n8n credentials, not in the JSON.

- `Generate Audio (English)` and `Generate Audio (Chines)`
  - Purpose: Similar to Polish but use different endpoints/voices and output names.
  - Their output is piped into `ElevenLabs` speechToText nodes and Google Drive upload nodes.

### Save audio to Google Drive

- `Google Drive2/3/4` (save MP3s and provide `webViewLink` and `webContentLink`)
  - Purpose: Upload the generated MP3s to the `Create Audio Folder` and return public view/download links.
  - These links are later written to Google Sheets.

### Transcription → SRT generation

- `ElevenLabs` nodes (three nodes named `ElevenLabs`, `ElevenLabs1`, `ElevenLabs2`)
  - Purpose: Speech-to-text transcriptions for Polish (`pl`), Chinese (`zh`) and English (`en`).
  - Output: JSON transcription that includes `words` with timings.

- `Code7`, `Code11`, `Code12` (SRT builders)
  - Purpose: Turn the `words` arrays from transcription into SRT content by grouping blocks of up to N words and formatting start/end times.
  - They create SRT binary data which is saved using `readWriteFile` nodes.

- `upload polish subtitles2`, `upload chines subtitles`, `upload english subtitles` (readWriteFile)
  - Purpose: Save SRT text to local files under `/home/node/n8n-uploads/*.srt` ready for ffmpeg.

### Video selection and download

- `Get all video files` (HTTP request to Drive API)
  - Purpose: Query a Drive folder for video files and list them.

- `select randome video` (code)
  - Purpose: From the returned file list, pick a random file and produce a `downloadLink` that points to `https://drive.google.com/uc?id=...&export=download`.

- `download video` (httpRequest) and `uplod video` (readWriteFile)
  - Purpose: Download the chosen video into `/home/node/n8n-uploads/Video.mp4`.

### FFMPEG processing: burn-in subtitles, attach audio, loop & trim

There are three processing commands (executeCommand nodes) that create language-specific working video files:

- `Process polish video` / `Process polish video1/2/3` (ffmpeg commands)
  - Command summary: Use the random video as background (`-stream_loop -1`) and overlay burned-in subtitles using `subtitles='*.srt'` and style options; map the language audio track and produce `Polish-working.mp4`.
  - Parameters: codec options (libx264), CRF and bitrate settings.
  - Output: `/home/node/n8n-uploads/Polish-working.mp4`.

- `process chines video` and `process english video` — same as above for other languages (different font and styling for Chinese; different encoding params for English).

### Background music selection and mixing

- `Get all music files1` → `Select random music` → `download music` → `uplod music`
  - Purpose: Choose a random music track from Drive, download it and write to `/home/node/n8n-uploads/music.mp3`.

- `Process English Music`, `Process Chines music`, `Process polish music`
  - Purpose: Mix the generated video audio with the background music using `ffmpeg -filter_complex` with `amix`. The music volume is reduced (e.g., `volume=0.05`) and looped to match the voice track.
  - Output: final `*-output.mp4` (English-output.mp4, Polish-output.mp4, Chines-output.mp4).

### Final upload and Google Sheets / Telegram notifications

- `upload polish drive`, `upload chines video`, `upload english video` (Google Drive)
  - Purpose: Upload final MP4s to the previously created folder and produce `webViewLink` and `webContentLink`.

- `upload video drive links` (Google Sheets)
  - Purpose: Append a new row to the `Video -content` sheet with the final Drive links and `MessageId` for tracing.

- `Get audio and subtitles path`, `Get all video files`, merges
  - Purpose: Collect paths and metadata, merge rows into a complete data structure that is appended to the audio-files Google Sheet.

- `Telegram4 / Telegram5 / Telegram6` (telegram nodes)
  - Purpose: Send the final video/media or a message with the download link to the user. There's `If` logic that checks file size and either uploads the video directly or sends a download link if the generated file is too big for Telegram.

---

## Important decision points and conditions

- `If1` (message vs callback or Revise) — routes whether the input triggers generation or a revision flow.
- `If` (Accept) — when the user clicks `Accept` the workflow edits the original message and proceeds to asset generation. If `Revise`, the AI agent is invoked again and a new draft is generated.
- File-size checks (`stat -c%s /home/node/n8n-uploads/Polish-output.mp4` + `If2/If3/If4`) — ensure Telegram size limits. If too large, the workflow sends a download link.

## Security and operational notes

- Credentials: The n8n nodes should reference credentials stored in n8n, not plain API keys in the exported JSON. Rotate keys if present in JSON.
- Local paths: The workflow uses `/home/node/n8n-uploads/` — ensure the n8n worker has write access and sufficient disk space.
- Fonts for subtitles: Chinese processing uses `Noto Sans CJK SC` in the ffmpeg burn command; ensure fonts are installed on the machine.
- Rate-limits & costs: TTS and transcription APIs (ElevenLabs/OpenAI) may have limits and costs. Add retry/backoff/quotas where necessary.

## Edge cases and suggestions

- Empty input: `Code` and `Code4` perform some checks; the AI path should handle empty `text` gracefully and return an informative message.
- Partial or failed uploads: Add checks for uploads (Drive nodes) and retry logic if the responses are missing.
- Long audio / video durations: The ffmpeg `-shortest` option is used; be careful with long looped background videos and memory.
- Logging & monitoring: Consider sending critical failures to an admin Slack/Telegram chat or to an error logging/monitoring system.

## Where to look to change behavior

- AI prompt: edit the `AI Agent` node `text` parameter — it contains instructions and anchored phrases.
- TTS voices/settings: adjust the `Generate Audio (*)` request body and `voice_settings` or ElevenLabs node configuration.
- Video style and sizes: edit the ffmpeg commands in the `Process *` nodes to tune bitrate, fonts, or subtitle font sizes.

---

## Quick references (node IDs and names)

Key nodes referenced in this document (name — id excerpt):
- `Telegram Trigger` — `d2e5dd66-...`
- `Code` — `0b0cefe3-...` (input parsing)
- `If1` — `c997a727-...`
- `AI Agent` — `48671c67-...`
- `OpenAI Chat Model` — `a853d8e1-...`
- `Code4` — `033b22b0-...` (cleaner)
- `Generate Audio (Polish)` — `b3d27d7a-...`
- `Generate Audio (English)` — `c66c1da4-...`
- `Generate Audio (Chines)` — `c3089af3-...`
- `Create Audio Folder` — `d4e66687-...`
- `ElevenLabs` STT nodes — `31a707a6-...`, `a54f2d07-...`, `abd74054-...`
- ffmpeg processors — `dcbe9b21-...`, `f09aff67-...`, `349f122b-...`

---

## Final checklist for running and testing

1. Ensure n8n credentials for Telegram, Google Drive, Google Sheets, OpenAI and ElevenLabs are set up in n8n.
2. Ensure the worker has fonts and ffmpeg installed and `/home/node/n8n-uploads/` exists.
3. Run a test by sending a Telegram message with a short `TOPIK` and watch the workflow: AI generation → message with buttons → Accept → media creation → Google Sheets rows and Drive uploads.
4. Verify SRT files and audio previews exist in Drive and final videos are uploaded.

---

If you want, I can:
- produce a compact flowchart (text or Mermaid) for this workflow;
- sanitize the `workflow.json` by removing embedded API keys before committing;
- or extract each node into a small table file for quicker editing.

Completion note: `workflow.md` created to explain every step. If you want changes (more/less detail or a specific focus), tell me which sections to expand.
