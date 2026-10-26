# BeautyCopy — photo input

Telegram perfume copywriter for Polish and English. Send a bottle photo, confirm/correct
the detected product, then choose Allegro, Amazon, Instagram, dupes, or a Campaign Pack
in Luxury, Minimal or Sales style. Choosing the format before sending a photo also works.

## Photo workflow

1. Send a photo of ONE perfume bottle with a readable label (JPG, PNG or WebP, up to 5 MB).
2. Review the identity. Tap **Użyj / Use**, or type the correct full name.
   Visual matches without a readable label are explicitly tentative and require confirmation.
   Low-confidence or ambiguous photos require a full name instead of confirmation.
3. Choose Polish/English, format and campaign tone.
4. Generate other formats from the saved product, or use **New product** / `/new`.

This feature generates TEXT from photos; it does not edit or generate advertising images.
It does not browse the web or reverse-search images. Review fragrance notes and other
knowledge-based facts before publishing. Bottle colour alone does not establish the exact
Le Male variant, concentration, size or authenticity.

## Implementation

- Cloudflare Workers AI `@cf/google/gemma-4-26b-a4b-it` performs vision and text generation.
- The existing optional Gemini key provides a fallback using inline image bytes.
- SQLite-backed Durable Objects persist a conversation per chat/user/topic.
- The webhook acknowledges an update after queuing it durably; alarms process AI work.
- Recent update IDs are deduplicated. Network/storage failures can still cause at-least-once
  delivery; exactly-once Telegram messaging is not guaranteed.
- Image bytes and token-bearing Telegram download URLs are never stored in conversation
  state or sent as URLs to AI. A queued Telegram file ID is removed after processing.
- Recognised label/appearance, confirmed name and format are retained for 24 hours after
  activity, then removed by an alarm. `/new`, `/cancel` and `/start` reset the current product.
  Provider retention and Cloudflare operational backups follow their own policies.
- Free usage is subject to Workers, AI and Durable Object quotas. Photos add an AI call.

## Update the existing Worker

`wrangler.jsonc` keeps the Worker name `beauty-perfume-ai`. Deploy to the SAME Cloudflare
account. Existing Telegram/Gemini secrets are preserved. No new provider key is required
when Workers AI is available.

```powershell
npm ci
npm run check
npm test
npx wrangler login
npx wrangler deploy --dry-run
npm run deploy
```

Migration `v1-photo-sessions` creates the SQLite Durable Object namespace bound as
`BOT_SESSIONS`. Keep it in future deployments.

For a fresh Worker only, use `npx wrangler secret put NAME` to set `TELEGRAM_BOT_TOKEN`
and `TELEGRAM_WEBHOOK_SECRET`. `GEMINI_API_KEY` remains optional.

When updating the same Worker URL, keep the existing Telegram webhook.
Photos are part of `message` updates, so the existing `allowed_updates: ["message"]` is enough.

## Validation

`npm test` runs 21 offline tests with mocked Telegram and AI responses: both operation
orders, correction/ambiguity, all twelve formats/tones, language changes, photo replacement
and failure, file bounds, Gemini fallback, user/topic isolation, inbox/restart, deduplication,
expiry and webhook authentication. These tests do not measure real-photo accuracy.

After deployment, check the Worker root for `version: "2.0.0-photo"`, `configured: true`,
`photoRecognition: true` and `sessions: true`. In Telegram, use `/start`, send a photo,
confirm and generate a format. Also test another product and an unreadable label.

## References

- [Cloudflare Gemma 4 vision support](https://developers.cloudflare.com/workers-ai/models/gemma-4-26b-a4b-it/)
- [Durable Objects plans and quotas](https://developers.cloudflare.com/durable-objects/platform/pricing/)
- [Telegram getFile](https://core.telegram.org/bots/api#getfile)

