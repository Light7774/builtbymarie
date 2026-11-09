# Build guide: Win-back demo workflow (n8n)

The website form sends this JSON to your webhook:

```json
{ "name": "...", "email": "...", "business": "...", "reason": "...",
  "offer": "...", "tone": "warm|direct|playful", "website": "", "consent": true }
```

The page expects this back:

```json
{ "ok": true, "emails": [ { "send_on": "Day 1", "subject": "...", "body": "..." }, ... ] }
```

On an error, send `{ "ok": false, "message": "friendly text" }`. The page shows that message to the visitor.

The finished workflow looks like this:

```
Webhook → Code: Check → IF ok? ─no─→ Respond (error)
                              └yes→ HTTP: Groq → Code: Parse → Respond (emails)
                                                             → Sheets → Gmail → Telegram
```

---

## Step 1. Webhook

1. Create a new workflow named `Website – Win-back demo`.
2. Add a **Webhook** node:
   - HTTP Method: `POST`
   - Path: `winback-demo`
   - Respond: `Using 'Respond to Webhook' Node`
   - Options → **Allowed Origins (CORS)**: `https://www.builtbymarie.com,https://builtbymarie.com`
3. Click **Listen for test event**. Send a test from your own terminal:

   ```
   curl -X POST https://n8n.builtbymarie.com/webhook-test/winback-demo -H "Content-Type: application/json" -d "{\"name\":\"Test\",\"email\":\"you@example.com\",\"business\":\"Paid yoga community\",\"reason\":\"They miss classes and feel behind\",\"offer\":\"\",\"tone\":\"warm\",\"website\":\"\",\"consent\":true}"
   ```

   The data shows up under `body`.

## Step 2. Code node "Check" (validation, honeypot, rate limit)

Add a **Code** node set to *Run Once for All Items* and paste this in:

```js
const b = $input.first().json.body || {};
const clean = (v, max) => String(v || '').replace(/[<>]/g, '').trim().slice(0, max);
const d = {
  name: clean(b.name, 40), email: clean(b.email, 120).toLowerCase(),
  business: clean(b.business, 140), reason: clean(b.reason, 400),
  offer: clean(b.offer, 120), tone: ['warm','direct','playful'].includes(b.tone) ? b.tone : 'warm',
  consent: b.consent === true,
};
const fail = (message) => [{ json: { ok: false, message } }];

if (b.website) return fail('Something went wrong.');            // honeypot: bots fill this
if (!d.name || !d.business || !d.reason) return fail('Please fill in all the required fields.');
if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(d.email)) return fail('That email does not look right.');
if (!d.consent) return fail('Please tick the consent box.');

// 1 run per email per day + max 40 runs per day overall (protects your Groq credits)
const store = $getWorkflowStaticData('global');
const today = new Date().toISOString().slice(0, 10);
if (store.day !== today) { store.day = today; store.seen = {}; store.count = 0; }
if (store.seen[d.email]) return fail('You already ran the demo today. Check your inbox for your copy.');
if (store.count >= 40) return fail('The demo hit its daily limit. Email angelie@builtbymarie.com and I will send yours by hand.');
store.seen[d.email] = true; store.count++;

return [{ json: { ok: true, ...d } }];
```

> The rate limit only remembers between **production** runs (once the workflow is Active). It resets during testing, which is normal.

## Step 3. IF node

- Condition: `{{ $json.ok }}` is **true** (Boolean).
- **False** branch → add a **Respond to Webhook** node:
  - Respond With: `JSON`
  - Response Body: `{{ { ok: false, message: $json.message } }}`
  - Options → Response Code: `400`

## Step 4. HTTP Request to Groq (true branch)

1. Get an API key at console.groq.com. In n8n, create a **Header Auth** credential: Name `Authorization`, Value `Bearer YOUR_KEY`.
2. Add an **HTTP Request** node:
   - Method `POST`, URL `https://api.groq.com/openai/v1/chat/completions`
   - Authentication: Generic → Header Auth → your credential
   - Body: JSON, using the expression below. For the model, pick a current Llama model from the Groq console; the one below is an example.

```
{{ JSON.stringify({
  model: "llama-3.3-70b-versatile",
  temperature: 0.7,
  response_format: { type: "json_object" },
  messages: [
    { role: "system", content: "You write win-back email sequences for membership businesses. Reply ONLY with JSON: {\"emails\":[{\"send_on\":\"Day 1\",\"subject\":\"...\",\"body\":\"...\"},{\"send_on\":\"Day 4\",...},{\"send_on\":\"Day 10\",...}]}. Exactly 3 emails. Each body under 120 words, plain text, no placeholders except {first_name}. Email 1 acknowledges why they left, email 2 shows what changed or what they are missing, email 3 makes the offer (if any) with a clear deadline. Never invent prices or facts not given." },
    { role: "user", content: "Membership: " + $json.business + "\nWhy members leave: " + $json.reason + "\nOffer: " + ($json.offer || "none, use a soft invitation to come back") + "\nTone: " + $json.tone }
  ]
}) }}
```

## Step 5. Code node "Parse"

```js
const raw = $input.first().json.choices?.[0]?.message?.content || '{}';
let emails = [];
try { emails = JSON.parse(raw).emails || []; } catch (e) {}
emails = emails.slice(0, 3).map(m => ({
  send_on: String(m.send_on || ''), subject: String(m.subject || ''), body: String(m.body || '')
}));
const lead = $('Check').first().json;
return [{ json: { ok: emails.length === 3, emails, ...lead,
  message: emails.length === 3 ? '' : 'The AI tripped up. Please try once more.' } }];
```

## Step 6. Respond to Webhook (success)

- Respond With: `JSON`
- Response Body: `{{ { ok: $json.ok, emails: $json.emails, message: $json.message } }}`

Every node after this keeps running after the visitor already has their answer. That keeps the page fast.

## Step 7. Google Sheets: log the lead

Connect it after the Respond node.

- Operation: **Append Row** to a sheet named `Demo leads`.
- Columns: Date `{{ $now.toISO() }}`, Name, Email, Business, Reason, Offer, Tone.

## Step 8. Gmail: send the visitor a copy

- To: `{{ $json.email }}`
- Subject: `Your win-back sequence from builtbymarie.com`
- Email Type: HTML. For the body, use this expression:

```
{{ "<p>Hi " + $json.name + ",</p><p>Here is the 3-email sequence you generated.</p>" +
   $json.emails.map(m => "<h3>" + m.send_on + ": " + m.subject + "</h3><p>" + m.body.replace(/\n/g,"<br>") + "</p>").join("") +
   "<p>Want this running automatically on your real member list? Reply to this email.</p><p>Marie<br>builtbymarie.com</p>" }}
```

Send it from your **angelie@builtbymarie.com** account if that inbox is connected to n8n. Otherwise use your Gmail and set the Reply-To to angelie@.

## Step 9. Telegram: ping yourself

- Message: `🎯 New demo lead: {{ $json.name }} ({{ $json.email }}) — {{ $json.business }}`
- On both the Gmail and Telegram nodes, go to Settings → **On Error: Continue**. If one of them fails, the lead is still saved in your Sheet.

## Step 10. Go live

1. Save the workflow and switch it to **Active**. The production URL is `https://n8n.builtbymarie.com/webhook/winback-demo`, which is the one the site already uses.
2. Upload the new `index.html` to Cloudflare Pages.
3. Run the form yourself on builtbymarie.com with a real email, then check the page, your inbox, the Sheet and Telegram.
4. Try it with the same email again. You should see the "already ran the demo today" message.
