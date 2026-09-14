# Privacy-Preserving Vision Agent

A browser agent that reads the screen **locally**, strips PII before any network
request, sends only a sanitized text description to a server-side LLM, and
executes the action it returns.

The server never sees a password, card number, government ID, or a single pixel
of an image — none of it is ever collected into the payload to begin with.

## The idea

A capable agent needs a powerful model, but sending it your screen means sending
it your passwords. So split the job:

- **Perception + redaction stay local** — DOM reading, checksum-validated PII
  masking, and a local CNN for face detection.
- **Reasoning stays server-side** — Gemini picks the next click/type/scroll from
  the sanitized description alone.

Two independent safeguards, enforced in different processes:

1. **Never collect it.** Sensitive field values are never read. A final leak
   check re-scans the serialized payload before it leaves the browser.
2. **Never act on it.** The client refuses any action targeting a field it
   marked sensitive — *regardless of what the server said*.

Breaking one doesn't break the other. Compromise the server entirely and you
still get no password: it was never in the payload, and the client won't type
into the field.

## Architecture

```
[Webpage]
   │ content_script.js walks the DOM → {idx, tag, label, sensitive, filled, bbox}
   │ redact.js masks PII; sensitive field values are never read at all
   ▼
[Sanitized JSON]  ──► background.js re-scans it for leaks before sending
   │ POST /agent/step
   ▼
[FastAPI] → prompt.py describes the redaction scheme to Gemini
   │        → {"action", "target_idx", "value", "reasoning"}
   ▼
[background.js] → content_script.js executes it
   │              └─ refuses anything targeting a data-agent-sensitive element
   ▼
   loop until "done" (max 12 steps)

(in parallel, decoupled)
background.js screenshots the viewport → offscreen.js runs UltraFace via ONNX
Runtime Web (WASM, bundled weights) → returns bounding boxes only → blur overlay
drawn on the page. No image bytes are sent anywhere.
```

## What makes the redaction hold up

**Structural classification, not keyword matching.** `classifyElementSensitivity`
reads the DOM contract — `type=password`, `autocomplete=cc-number`,
`one-time-code` — so it works regardless of how a page labels its fields.

**Checksums, not just shape.** A 16-digit string only masks if it passes **Luhn**;
a 12-digit string only masks as Aadhaar if it passes **Verhoeff**. This is what
keeps order numbers and tracking IDs out of the false-positive column.

**Keyword anchoring for generic values.** A bare 4-digit number can't be an OTP
on sight, so OTP/CVV/bank-account patterns require an adjacent keyword.

**18 text categories** — email, phone, card, SSN-shaped ID, Aadhaar, PAN, IFSC,
voter ID, passport, driving licence, UPI, JWT, API keys, OTP, CVV, password,
bank account, address. **10 field categories**, plus images and free-text PII.

**The `filled` flag.** Sensitive fields report *whether* they hold a value, never
what it is — the minimum disclosure that lets the agent tell "waiting on you"
from "already done" instead of stalling forever.

**Write-only recipients.** An editable email field sharing a `<form>` with a
password is your own login (safe to show). Anything else defaults to
`recipient`: typable by the agent if you named the address in your task, but
masked on every read, before and after filling.

## Requirements

- **Chrome 114+** (or Edge/Brave/any Chromium 114+) — `chrome.sidePanel` landed
  in 114. **Firefox is not supported**: no `sidePanel` or `offscreen` API.
- **Python 3.9+** for the server.
- **Node.js** for the eval harness only.
- No GPU. Inference is single-threaded WASM on CPU.

Installed size is **~12.2 MB** (~3.9 MB compressed), of which 12.16 MB is the
vendored ONNX Runtime and model. The agent's own code is 52 KB.

## Setup

**1. Demo site**

```bash
python3 demo-site/serve.py 8000
```

**2. Server**

```bash
cd server
python3 -m venv .venv
./.venv/bin/pip install -r requirements.txt
./.venv/bin/uvicorn main:app --port 8787 --reload
```

For real reasoning, create `server/.env` with your key from
[aistudio.google.com](https://aistudio.google.com/apikey):

```
GEMINI_API_KEY=your_key_here
# GEMINI_MODEL=gemini-flash-lite-latest
```

Without a key the server runs in **stub mode** — a deterministic "click the
first safe element" rule that proves the client/server wiring end-to-end with no
API key needed. `curl http://127.0.0.1:8787/healthz` reports which mode is
active.

Defaults to `gemini-flash-lite-latest`, a rolling alias chosen because the full
flash model is capped at 5 requests/minute on the free tier — too tight for a
multi-step loop.

**3. Extension**

1. `chrome://extensions` → enable Developer mode → **Load unpacked** →
   select `extension/`.
2. Click the extension icon to open the side panel (it stays open across
   navigations, unlike a popup).
3. Open `http://127.0.0.1:8000/index.html`, type a task, click **Run task**.
4. **You** type the password and card number. The agent never sees or fills
   them — their values never leave the page.

## Repo layout

| Path | What |
|---|---|
| `extension/background.js` | Agent loop, leak check, send/scope guards |
| `extension/content_script.js` | DOM reader, action executor, the sensitive-field veto |
| `extension/redact.js` | All PII rules. Shared verbatim with the eval harness |
| `extension/offscreen.js` | UltraFace inference (ONNX Runtime Web, WASM) |
| `extension/lib/` | Vendored ORT runtime (10.5 MB) + UltraFace weights (1.2 MB) |
| `server/main.py` | FastAPI endpoint, Gemini call, stub fallback |
| `server/prompt.py` | Prompt builder — explains the redaction scheme to the model |
| `server/schema.py` | Pydantic contracts; `Literal` allowlist on the action verb |
| `demo-site/` | Login → profile → checkout flow, plus a webmail inbox |
| `eval/` | Precision/recall harness for PII detection and redaction |

`redact.js` is a UMD module, so the extension and the Node eval harness load the
*same bytes* — the measured numbers describe exactly what ships.

## Verification

```bash
node eval/run_redaction_eval.js
```

**100% precision and recall** across 54 text cases and 22 element-classification
cases — covering Luhn-valid vs. invalid cards, Verhoeff-valid vs. invalid
Aadhaar, clean text that must *not* be flagged, and structural field typing.

Also verified:

- **The veto fires in practice.** Gemini has been observed proposing a `type`
  into the password field while its own `reasoning` said it shouldn't. The guard
  is load-bearing, not decorative.
- **Face detection** — 99.998% confidence on the bundled portrait, ~50–90 ms
  inference, running under MV3's real CSP with the shipped files unmodified.
- **End-to-end Gemini reasoning** across a multi-page login → profile → checkout
  task, ~1.3 s per step.

**Not automatable:** loading the full extension in Chrome requires
`chrome://extensions`, which is blocked from browser automation. Every component
was verified independently, including under the exact MV3 CSP, but the final
assembly needs a human to load it once.

## Evaluation criteria

| Criterion | Weight | Where |
|---|---|---|
| Accuracy of visual context | 25% | DOM walk in `content_script.js` (structure, labels, bboxes) + UltraFace for face presence |
| PII detection precision/recall | 20% | `redact.js`; measured at 100/100 in `eval/` |
| Redaction precision | 20% | Same harness, plus visible on-screen masking of page text, field values, and faces |
| Client-side resource use | 20% | 1.2 MB model, 50–90 ms inference, ~12 MB install, no GPU required |
| End-to-end latency | 15% | Side panel reports per-step DOM-read and server round-trip time live |

## Known limitations

- **Face detection is decoupled from the reasoning loop.** It drives the
  on-screen blur only; its boxes never enter the payload. Since no image data is
  ever transmitted, it isn't what keeps faces off the network — not sending
  images is. It demonstrates local CV capability and guards against
  shoulder-surfing.
- **Redaction is rule-based, not ML-based.** It catches structured PII
  exhaustively, but not free-form sensitive text with no pattern — a name or a
  diagnosis in a sentence. An NER model is the natural next step.
- **`host_permissions` is `<all_urls>`.** Not by choice:
  `chrome.tabs.captureVisibleTab` requires literally `<all_urls>` or
  `activeTab`, and `activeTab` proved unreliable across a long-lived side panel
  session. A production build should revisit this.
- **The 10.5 MB WASM binary is the ONNX *runtime*, not the model.** The model is
  1.2 MB. Shrinking it needs a custom operator-trimmed ORT build.
- **WASM, single-threaded** — `numThreads = 1` avoids the cross-origin isolation
  headers `SharedArrayBuffer` requires. WebGPU is available in ORT and unused.
- **A submit click can silently no-op** if the page's own HTML validation
  rejects it; `el.click()` doesn't throw. `content_script.js` checks
  `form.checkValidity()` afterward and surfaces a warning rather than a
  misleading "ok".
