# Create API

One endpoint that turns **text into a marketing image**.

You send some text and say which platform it is for. The service picks the right
prompt, generates the image, and returns it at the exact pixel size that platform
needs.

```
POST /image-transform/create
```

**Generated images contain no text.** No words, headlines, captions, prices,
watermarks or logos. The text you send decides *what the scene shows* — it is never
drawn into the image. (One exception: Google Ads *logo* dimensions, covered below.)

---

## Contents

1. [The one thing to understand](#1-the-one-thing-to-understand)
2. [Request fields](#2-request-fields)
3. [Platforms and dimensions](#3-platforms-and-dimensions)
4. [What text to send](#4-what-text-to-send)
5. [The response](#5-the-response)
6. [Errors](#6-errors)
7. [Locales](#7-locales)
8. [Building a UI on this](#8-building-a-ui-on-this)
9. [Setup and running](#9-setup-and-running)
10. [Files](#10-files)
11. [The prompts](#11-the-prompts)
12. [Matching a client's look](#12-matching-a-clients-look)

---

## 1. The one thing to understand

**`image_type` decides which prompt runs. `dimension` decides the pixel size.**

That is the whole design. Same URL every time; the payload changes the behaviour.

```jsonc
// a Facebook / Instagram story
{ "input_text": "Book your fall security check today.",
  "image_type": "meta", "dimension": "meta_story" }          → 1080 × 1920

// a website hero from a brand kit
{ "brand_kit": { ... },
  "image_type": "website", "dimension": "website_hero" }     → 1920 × 1080

// no platform named — a general marketing image
{ "input_text": "Sharma Dental Care. Family dentistry since 2011..." }
                                                             → 1024 × 1024
```

That last one is `image_type: "generic"`, the default. If you send no
`image_type`, you get a general marketing image at a standard size.

**You get 3 images per request** (three different angles on the same business).
Google Ads logo dimensions return **1**.

---

## 2. Request fields

| Field | Required | Default | Notes |
|---|---|---|---|
| `input_text` | **yes** — except the two heroes | — | The source text. Aliases: `website_text`, `post_text`. Minimum 20 characters. |
| `brand_kit` | no | — | **Only `website_hero` and `website_hero_banner`.** Sending it with any other dimension returns 400. Object or JSON string. |
| `image_type` | no | `generic` | `generic` · `meta` · `gbp` · `google_ads` · `website` |
| `dimension` | no | that platform's default | A named key from the table below — **not** a `WxH` string. |
| `locale` | no | none | `IN` or `en-IN`. Also accepted as `country`. |
| `variations` | no | `3` (logos: `1`) | 1 to 3. |
| `quality` | no | `low` | `low` · `medium` · `high` · `auto` |
| `reference_image_urls` | no | — | **The visual board's URL.** It is a list, but send **one** — the board. See [Matching a client's look](#12-matching-a-clients-look). |
| `reference_kind` | no | `visual_board` | Already correct for a board, so you can leave it out. Send `photographs` only if you are attaching plain photos instead. |
| `size` | no | — | Legacy. A raw `1024x1024` / `1024x1536` / `1536x1024` / `auto`, **`generic` only**. New integrations should use `dimension`. |

**To match a client's look, send the board's URL.** `reference_kind` already defaults
to `visual_board`, so that one field is enough. The example below sends it explicitly
because it reads better in a request you are about to copy.

```jsonc
{ "input_text": "Today we are installing a new EPDM roof on a flat commercial roof...",
  "image_type": "meta", "dimension": "meta_square", "locale": "en-US",
  "reference_image_urls": ["https://your-bucket/boards/acme.png"],
  "reference_kind": "visual_board" }
```

The endpoint also accepts `visual_profile`, `shot_types`, and up to 8 reference
photographs instead of a board. None of those are part of this flow and none were
exercised in testing — they are described at the end of
[section 12](#12-matching-a-clients-look) so nobody has to guess what they do, but you
do not need them.

**`brand_kit` is never required.** It is an alternative input that only the two
website hero dimensions accept:

| Dimension | Send |
|---|---|
| `website_hero`, `website_hero_banner` | `brand_kit` **or** `input_text` — at least one. If you send both, the **brand kit wins** and `input_text` is ignored. |
| everything else (all 20 other dimensions) | `input_text`. A `brand_kit` here is rejected with `brand_kit applies to a website hero only`. |

> `quality` defaults to `low`, which is the usual reason an image looks soft. Use
> `"quality": "high"` for anything a client will see. It costs more per image.

**`dimension` is a name, not a size.** Pixel sizes are not unique — Google Ads
"Square" and "Logo Square" are both 1200×1200, and Google Ads landscape (1200×628)
is nearly Meta landscape (1200×630). Only the name says which you want.

---

## 3. Platforms and dimensions

Get this list from **`GET /catalog`** rather than hard-coding it.

### generic — no particular platform (the default)

| `dimension` | You receive |
|---|---|
| `generic_square` ← default | 1024 × 1024 |
| `generic_portrait` | 1024 × 1536 |
| `generic_landscape` | 1536 × 1024 |
| `generic_auto` | model chooses |

### meta — Facebook / Instagram

| `dimension` | You receive | |
|---|---|---|
| `meta_square` ← default | 1080 × 1080 | Feed square |
| `meta_portrait` | 1080 × 1350 | Feed portrait |
| `meta_story` | 1080 × 1920 | Story / Reel |
| `meta_landscape` | 1200 × 630 | Facebook landscape |

### gbp — Google Business Profile

| `dimension` | You receive | |
|---|---|---|
| `gbp_square` ← default | 1080 × 1080 | Recommended |
| `gbp_square_min` | 720 × 720 | Minimum accepted |

### google_ads — Performance Max

| `dimension` | You receive | |
|---|---|---|
| `gads_square` ← default | 1200 × 1200 | photo |
| `gads_landscape` | 1200 × 628 | photo |
| `gads_portrait` | 1200 × 1500 | photo |
| `gads_logo_square` | 1200 × 1200 | **logo — transparent PNG, 1 image** |
| `gads_logo_wide` | 1200 × 300 | **logo — transparent PNG, 1 image** |

### website

| `dimension` | You receive | |
|---|---|---|
| `website_hero` ← default | 1920 × 1080 | takes a brand kit |
| `website_hero_banner` | 1920 × 600 | takes a brand kit |
| `website_section` | 1200 × 628 | that section's own text |
| `website_sidebar_250` | 300 × 250 | card |
| `website_sidebar_600` | 300 × 600 | card |
| `website_sidebar_square` | 300 × 300 | card |
| `website_sidebar_portrait` | 300 × 375 | card |

**Mixing platforms is rejected.** `image_type: "meta"` with
`dimension: "gads_square"` returns 400.

> **Why the delivered size is not what the model generates.** GPT-Image-2 only
> accepts sizes where both edges divide by 16, the pixel count is 655,360–8,294,400,
> and the ratio is at most 3:1. Almost no real platform size qualifies. So each
> dimension is generated at the nearest valid size and centre-cropped to the exact
> delivered pixels. Images are scaled to fill and cropped — **never stretched**.
> You do not need to do anything; you always receive the size in the table.

---

## 4. What text to send

The prompts read the text differently, so send the right kind for the platform.

| `image_type` / dimension | Send | In which field |
|---|---|---|
| `generic` | the business's website text | `input_text` |
| `meta` | the social post text | `input_text` |
| `gbp` | the GBP post text | `input_text` |
| `google_ads` photo sizes | the ad text, or a description of the business | `input_text` |
| `google_ads` logo sizes | a description of the business, so the symbol suits it | `input_text` |
| `website_hero`, `website_hero_banner` | the brand kit (preferred), or plain text about the business | `brand_kit`, or `input_text` |
| `website_section` | that section's own content | `input_text` |
| `website_sidebar_*` | that card's own content | `input_text` |

**Why it matters:** the Meta prompt reads the text as *one post* and depicts that
message. The website hero prompt reads it as *a description of the business* and
depicts what the business does. Sending a business description to `meta` still
works, but you get a generic business shot rather than a post image.

The full text is passed through as-is. URLs, phone numbers and hashtags are left
in; the prompt tells the model not to draw them.

### The brand kit

For `website_hero` and `website_hero_banner`, send the whole brand kit. Only these
**four** fields are used; everything else is ignored:

- `contents_long_description`
- `contents_area_of_focus`
- `style_inspirations_brand_personality_traits`
- `brand_archetype`

Field names match loosely — `brand_archetype`, `brandArchetype` and
`Brand Archetype` all work. The brand kit is accepted as an object **or** as a
JSON string, including one with stray text stuck to it from a copy/paste.

If a brand kit is sent with none of those four fields, you get a 400 naming them.

---

## 5. The response

```jsonc
{
  "level": "Create",
  "image_type": "meta",
  "dimension": "meta_story",
  "size": "1080x1920",           // exactly what you received
  "kind": "photo",               // "photo" or "logo"
  "transparent_png": false,      // true for logo dimensions
  "quality": "low",
  "visual_profile_applied": true,   // false for logos, or if none was sent
  "shot_types_applied": true,       // false for logos, or if none were sent
  "reference_images_used": 5,       // how many downloaded and attached
  "reference_kind": "photographs",  // null when no references were sent
  "locale": "en-IN",             // null if none sent
  "country": "IN",
  "variations": 3,
  "results": [
    {
      "index": 1,
      "angle": "Service in action",     // null whenever only one image is returned
      "ok": true,
      "image_base64": "iVBORw0KGgo...", // raw base64, no "data:" prefix
      "mime": "image/png",
      "render_prompt": "...",           // the exact prompt used
      "usage": {
        "input": 3841, "output": 204, "total": 4045,
        "input_text_tokens": 1180,      // the prompt
        "input_image_tokens": 2661,     // the references, 0 without them
        "cached_input_tokens": 0
      },
      "timing_ms": { "render": 20500, "total": 20500 }
    }
    // ...one entry per variation
  ],
  "usage": { ... },                      // the same six numbers, all combined
  "timing_ms": { "total": 20500 }        // wall clock, not the sum
}
```

**Why `usage` splits the input.** The model bills text input and image input at
different rates — image input is the dearer of the two — so a single `input` number
cannot be costed. With a reference attached most of the input is image tokens. The
published per-million rates are on OpenAI's pricing page; the token counts here are
what the model actually charged for that call.

**Always a `results` array**, even when it holds one entry.

**One variation can fail without losing the others.** That entry comes back with
`"ok": false` and an `error`, and the rest still contain images. Only if every
variation fails does the request return an error status.

The three variations run **in parallel**, so three images take about as long as one.

**Saving the images:**

```javascript
response.results
  .filter(r => r.ok)
  .forEach(r => { img.src = "data:image/png;base64," + r.image_base64; });
```

---

## 6. Errors

Every failure returns the same shape:

```json
{ "error": "a message explaining what to fix" }
```

**There is only ever one shape** — no arrays, no nested detail objects. You never
need to branch on the response body.

**A field of the wrong type is rejected, not ignored.** `"dimension": 5` returns
`dimension must be a string` rather than quietly falling back to the default and
returning a correct-looking image at the wrong size. Fields the API does not know
are ignored.

| Status | Meaning | Examples |
|---|---|---|
| **400** | something in the request, or the content | `input_text is required (or send a brand_kit for a website hero)` · `image_type must be one of: generic, meta, gbp, google_ads, website` · `dimension 'gads_square' belongs to image_type 'google_ads', not 'meta'` · `dimension 'gads_logo_square' returns at most 1 image(s); 3 requested` · `brand_kit applies to a website hero only` · `unknown locale 'ZZ'` (the message lists every valid value) · `dimension must be a string` · `shot_types must be a list of strings` · `reference_image_urls[2] must start with http:// or https://` · `too many reference images: 12 sent, the limit is 8` · `none of the reference_image_urls could be downloaded as an image` · **`request rejected by the OpenAI safety system`** (only when *every* variation was refused) |
| **502** | the image model failed | `image model error: ...` · `image model returned no image` |
| **500** | the service is misconfigured | `OPENAI_API_KEY is not set` · `prompt.txt missing` |

> **Safety blocks are not consistent.** OpenAI sometimes flags an image for a
> category such as `illicit` — security, locksmith and alarm businesses trip this
> most. The same request can pass on one attempt and be blocked on the next, so
> **retrying often just works**. It is not retried automatically. Every prompt
> already asks for lawful, safe scenes to reduce false flags.

---

## 7. Locales

The locale decides **who appears in the image**. `en-IN` produces Indian people,
`ja-JP` produces Japanese people. It applies to every platform.

Send either form:

```json
{ "locale": "IN" }      { "locale": "en-IN" }      { "country": "IN" }
```

21 markets: `US CA GB IE AU NZ IN SG MY PH FR DE ES IT NL MX BR JP KR ZA AE`

Get the list from **`GET /locales`**.

`locale` is **optional**, and there is no default. Omit it and the image model
chooses the people itself — so if the market matters, always send it. The response
echoes `locale: null` when none was used.

Logo dimensions ignore locale — a logo has no people in it.

---

## 8. Building a UI on this

**Build your controls from `GET /catalog`.** It is generated from the backend's own
dimension table, so it cannot fall out of step with what the API accepts.

```jsonc
{
  "default_image_type": "generic",
  "image_types": [
    {
      "image_type": "meta",
      "default_dimension": "meta_square",
      "dimensions": [
        { "dimension": "meta_story", "size": "1080x1920",
          "width": 1080, "height": 1920,
          "aspect": "tall vertical full-screen (9:16)",
          "kind": "photo", "transparent_png": false,
          "max_variations": 3 }
      ]
    }
  ]
}
```

Suggested flow:

1. Platform dropdown ← `image_types[].image_type`
2. Size dropdown ← that platform's `dimensions[]`, preselect `default_dimension`
3. Variations control ← cap at `max_variations`; **hide or disable it when
   `max_variations` is 1** (the logo assets), so the user cannot send a request
   that would be rejected
4. Show `transparent_png: true` as a hint that the result has no background
5. Locale dropdown ← `GET /locales`

Nothing needs hard-coding. Adding a platform or size later shows up in `/catalog`
automatically.

---

## 9. Setup and running

```bash
cd create-onestep-api
pip install -r requirements.txt
```

Create a `.env` in this folder:

```
OPENAI_API_KEY=sk-...
```

**Windows PowerShell** — the `cd` is required so `uvicorn` can import `app`:

```powershell
cd D:\Downloads\transform-image-python\create-onestep-api
D:\Downloads\transform-image-python\.venv\Scripts\python.exe -m uvicorn app:app --reload --port 8300
```

**Anywhere else:**

```bash
uvicorn app:app --reload --port 8300
```

Check it is alive:

```bash
curl http://localhost:8300/health      # {"ok": true}
```

Interactive API docs: <http://localhost:8300/docs>

### Example calls

```bash
# Instagram story
curl -X POST http://localhost:8300/image-transform/create \
  -H "Content-Type: application/json" \
  -d '{"input_text":"Book your fall security check today. Evening slots available.",
       "image_type":"meta","dimension":"meta_story","locale":"en-US","quality":"high"}'

# Google Ads wide logo — transparent PNG, one image
curl -X POST http://localhost:8300/image-transform/create \
  -H "Content-Type: application/json" \
  -d '{"input_text":"A family-run bakery making sourdough and pastries since 1994.",
       "image_type":"google_ads","dimension":"gads_logo_wide"}'

# Website hero from a brand kit — no input_text needed
curl -X POST http://localhost:8300/image-transform/create \
  -H "Content-Type: application/json" \
  -d '{"image_type":"website","dimension":"website_hero","quality":"high",
       "brand_kit":{"contents_long_description":"A family-owned painting company serving Greater Philadelphia since 1979.",
                    "contents_area_of_focus":["Residential painting","Cabinet painting"],
                    "style_inspirations_brand_personality_traits":"Dependable, Family-Oriented",
                    "brand_archetype":"Everyman"}}'
```

**PowerShell** quoting differs:

```powershell
curl.exe -X POST http://localhost:8300/image-transform/create `
  -H "Content-Type: application/json" `
  -d '{\"input_text\":\"Book your fall security check today.\",\"image_type\":\"meta\"}'
```

### Environment variables

| Variable | Required | Default | What it does |
|---|---|---|---|
| `OPENAI_API_KEY` | **yes** | — | your OpenAI key |
| `OPENAI_IMAGE_MODEL` | no | `gpt-image-2` | the image model. `gpt-image-2.5-flare` is also supported and is measurably faster when no references are attached. |
| `MAX_WEBSITE_CHARS` | no | `12000` | text is truncated to this |
| `OPENAI_TIMEOUT` | no | `300` | seconds to wait per call |
| `LOGO_BACKGROUND_TOLERANCE` | no | `30` | 0–255. How aggressively a logo's background is made transparent. Higher removes more but can eat into the logo. |
| `PHOTO_REALISM` | no | `1` | The realism block. `0` disables it and restores the original platform wording. |
| `MAX_PROFILE_CHARS` | no | `16000` | `visual_profile` is truncated past this, with a warning in the log. |
| `MAX_REFERENCE_IMAGES` | no | `8` | More than this is a 400. |
| `REFERENCE_TIMEOUT` | no | `60` | Seconds to wait per reference image. |
| `MAX_REFERENCE_BYTES` | no | `20971520` | 20MB per reference image. |

The key is read on the first request, not at startup, so the server starts and
`/health` answers without one.

---

## 10. Files

```
create-onestep-api/
├── app.py                 FastAPI entrypoint, CORS, /health
├── router.py              the routes
├── controller.py          reads the JSON body and validates it
├── service.py             resolve dimension → prompt → generate → exact size
├── dimensions.py          the 22 dimensions: generated size vs delivered size
├── platform_prompts.py    the meta / gbp / google_ads / website prompts
├── prompt.txt             the `generic` prompt
├── variations.txt         the three creative angles
├── brand_kit.py           pulls the four hero fields out of a brand kit
├── image_postprocess.py   cover-crop to exact size; logo background → transparent
├── locales.py             the 21 markets
└── requirements.txt
```

Two files to read first if you are changing behaviour: **`platform_prompts.py`**
(how images turn out) and **`dimensions.py`** (everything about sizes). The realism
block, the board rule and the prompt assembly all live in **`service.py`**.

`dimensions.py` validates every generation size **at import**. If a size breaks
GPT-Image-2's rules the service refuses to start, rather than failing later on one
dimension in production.

---

## 11. The prompts

| Platform | Prompt |
|---|---|
| `generic` | `prompt.txt` + an angle from `variations.txt` |
| `meta`, `gbp`, `google_ads`, `website` | `platform_prompts.py` |

The three angles are **Service in action**, **Customer moment** and **The place** —
what to photograph. Everything else in the prompt stays identical between them, so
the images differ in framing while staying about the same business.

The locale block and the variation angle are **appended** to a platform prompt
rather than written into it, so the platform prompts stay exactly as written.

**Prompt files are read once and then cached — restart the server after editing
any of them**, including `locales.py`. Editing a prompt on a running server has no
effect.

### Photographic realism

Every photograph is generated with a realism block: rules on anatomy (finger counts,
where limbs join, cropping between joints rather than through them), physics (things
rest on something, one light source, consistent scale), light, wear, wardrobe and
finish. It is stated **first**, before the platform prompt, because whatever is read
first sets the frame.

It also softens the platform prompts' own calls for polish — "high-quality", "clean
composition", "award-winning commercial photographer" — in the composed prompt only.
`platform_prompts.py` is never modified, so `PHOTO_REALISM=0` restores the original
wording exactly.

Logos skip it. A logo is a graphic and has no photography in it.

### When the copy is not a photograph

Plenty of real post copy describes the business rather than a moment: *"Our founder
James started the company in 1984"*, *"thirty years in business"*, *"voted best in
the county"*, *"our office is the tallest building on Main Street"*, a mission
statement. None of that is something a camera can point at. Left alone the model
invents a face for the founder, or a trophy for the award, and the image then asserts
something nobody can stand behind.

Every photograph prompt therefore carries a rule that says what to do instead:

1. **Find what the business actually sells or does** in the same text — the trade,
   the service, the product, the premises — and photograph that.
2. If the text names no product or service at all, photograph the ordinary everyday
   work of the trade the text implies, claiming nothing.

And in that situation it forbids:

- **Any identifiable face.** The copy names or implies a specific person, so a face
  in the frame reads as that person. Whoever is in shot is seen from behind, turned
  away, at a distance, or cropped so the head is outside the frame. Hands and the
  work are usually the better picture.
- **Illustrating the claim** — no trophy, certificate, badge, ribbon, plaque, medal,
  anniversary cake, balloon, year, number, star rating or ranking.
- **Depicting a superlative** — biggest, best, tallest and oldest are claims, not
  subjects.
- **Staging a stand-in** — no handshake for trust, no team lined up for experience,
  no full car park for popularity.

The test the rule states: the photograph must be true of any honest business of this
kind. If a viewer could hold it up as evidence of a specific claim, or as a picture
of a specific real person, it is wrong.

Measured on five posts of this sort, the images came back as the trade being
practised — a roofer nailing shingles for the founder copy, a plumber under a sink
for the award copy — with no faces, badges or trophies. Ordinary copy is unaffected:
posts about a haircut or a training session still come back with people's faces in
them. Logos skip this block too.

### The no-text rule, and its one exception

Every prompt forbids text, invented awards, trust badges, review stars and
fabricated business details. Images get published, so a garbled phone number or a
made-up award would be a real problem.

**The exception is the two Google Ads logo dimensions**, which must produce an
actual logo. Even there the no-text rule holds: the logo is a purely graphical
symbol with **no business name, wordmark, monogram or initials**. It is generated
on a plain white background, which is then flood-filled away to give you a
transparent PNG.

---

## 12. Matching a client's look

**Send the board. Two fields.**

```jsonc
{ "reference_image_urls": ["https://your-bucket/boards/acme.png"],
  "reference_kind": "visual_board" }
```

The board is one PNG produced by the companion **Visual Profile API** from that
client's media library: their own photographs, colour swatches measured from the
pixels, and written notes on how their photography looks. Build it once per business,
host it, and send the same URL with every image you generate for them.

`reference_kind` **defaults to `visual_board`**, so a request that attaches a board
and says nothing gets the board handling: rules telling the model to **read** the board
rather than imitate it, and that no grid, panel, swatch or text may reach the image.
Sending the field explicitly is still clearer, and costs nothing.

The one thing to know: if you ever attach plain photographs instead of a board, you
must say `"reference_kind": "photographs"`. Left out, they are treated as a board.

### The rule that matters

**The post text decides what the image is of. The profile and the board decide how it
looks.**

That division holds well when the copy asks for something the business actually does.
It is weaker when the copy asks for something outside what they photograph. Measured:
a roofing client whose library is all exterior work, given copy about a **ceiling**
stain indoors, returned four images of which one was indoors and three drifted back
outside to the roof. A board is a strong visual signal and it pulls toward the client's
usual subject.

So **name the setting in `input_text`** when it is not the client's usual one — "inside
the roof space", "at the counter", "in the workshop". Copy that only implies the
setting may not be enough.

### What references cost

References are re-uploaded to the model on **every variation**, so a three-variation
request with five references uploads fifteen images. That is the whole of the cost
story, because the model bills image input higher than text input.

Measured on this service at `quality: low`, 1080×1080, one variation, same copy, only
the attachment changed. Tokens are from the `usage` object on each response; cost is
those tokens at $5.00/M text input, $8.00/M image input, $30.00/M output:

| Attached | Text in | Image in | Out | Per image | Per request of 3 |
|---|---|---|---|---|---|
| nothing | 2,378 | 0 | 204 | **$0.0180** | $0.0540 |
| 1 visual board | 2,819 | 1,521 | 204 | **$0.0324** | $0.0971 |
| 5 photographs | 2,378 | 6,722 | 204 | **$0.0718** | $0.2154 |
| 8 photographs | 2,378 | 11,130 | 204 | **$0.1071** | $0.3212 |

**This is the case for a board.** One board carries the same information as the whole
library for about 1,500 image tokens; five photographs cost four times that and eight
cost seven times, for a worse match. The board's text input is slightly higher because
`reference_kind: "visual_board"` adds its own rules to the prompt — about 440 tokens,
or a fifth of a cent.

Attaching more also takes longer. Single samples from the same run: nothing 21.0s, one
board 16.4s, five photographs 28.2s, eight photographs 38.2s. The first two are within
normal variation of each other; the trend above five is not.

**Eight references is the cap** (`MAX_REFERENCE_IMAGES`). A ninth is a 400, not a
silent truncation.

Quality changes the output tokens, not the input — measured on this service: `low` 204,
`medium` 1,834, `high` 7,336. With references attached the input dominates, which is
why the choice of image model stops mattering: flare and gpt-image-2 landed within half
a second of each other when 16 references were attached.

**Logos ignore all of this.** A logo is a graphic; there is no photography in it to
match, so references, `visual_profile` and `shot_types` are all dropped for the two
Google Ads logo dimensions.

---

### Also accepted — not part of this flow

These three are implemented and validated, and a request carrying them will behave as
described. **None of them were used in any of the end-to-end tests**, and you do not
need them to match a client's look. They are documented so that nobody sending one has
to guess what it does.

| Field | What it does |
|---|---|
| `visual_profile` | A block of text describing how the client's images look — palette, light, camera, wardrobe, finish. Appended to the prompt. The Visual Profile API returns one as `profile_text`, as an alternative to hosting a board. |
| `shot_types` | A list of framings the client uses. One is handed to each variation, so several variations differ in framing as well as in creative angle. Without it the variations differ by angle alone. |
| `reference_kind: "photographs"` | Attaches the client's own photographs instead of a board, up to 8. Every one is re-uploaded on every variation — see the cost table above, where 8 photographs cost more than three times what a board costs, for a looser match. |

A board carries the same information as a folder of photographs in one image, so
`visual_board` with a single URL is both the cheapest and the closest option. That is
why it is the one the flow uses.
