# Dashboard files to paste into GitHub

There are 11 files. Create each one in your GitHub repository like this:

1. Open the repository, click **Add file**, then **Create new file**.
2. In the file name box, type the exact path shown in the heading below. Typing a slash creates the folder automatically, for example `netlify/functions/decide.mjs`.
3. Copy everything inside the code block under that heading and paste it into the editor. Copy only the code, not the lines of backticks.
4. Click **Commit changes**, then **Commit changes** again in the box that appears.
5. Repeat for every file. `netlify.toml` must be at the top level, not inside a folder.

When all 11 files are in the repository, link it to your Netlify site and it will deploy by itself.

Do not put any keys or secrets into these files. They are entered in Netlify's settings.

---

## 1. `netlify.toml`

````
[build]
  publish = "public"
  functions = "netlify/functions"

[functions]
  node_bundler = "esbuild"
````

## 2. `netlify/lib/util.mjs`

````
// Shared helpers for the dashboard's server functions.
// Secrets live in Netlify environment variables and never reach the browser.
import { createHash } from 'node:crypto';

export function env(name) {
  const v = process.env[name];
  if (!v) throw new Error('Missing environment variable ' + name);
  return v;
}

export function json(data, status = 200) {
  return new Response(JSON.stringify(data), {
    status,
    headers: { 'Content-Type': 'application/json', 'Cache-Control': 'no-store' },
  });
}

export const UUID = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
export const EMAIL = /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/;

export function clean(text, max) {
  return String(text ?? '')
    .replace(/[\u0000-\u0008\u000b\u000c\u000e-\u001f\u007f]/g, '')
    .trim()
    .slice(0, max);
}

// Call a Postgres function in Supabase through its REST interface.
export async function rpc(fn, args) {
  const base = env('SUPABASE_URL').replace(/\/$/, '');
  const key = env('SUPABASE_SERVICE_KEY');
  const headers = { 'Content-Type': 'application/json', apikey: key };
  if (key.startsWith('eyJ')) headers.Authorization = 'Bearer ' + key; // legacy JWT-style keys only
  const res = await fetch(base + '/rest/v1/rpc/' + fn, {
    method: 'POST',
    headers,
    body: JSON.stringify(args ?? {}),
  });
  const text = await res.text();
  let data;
  try { data = JSON.parse(text); } catch { data = text; }
  if (!res.ok) {
    const err = new Error((data && data.message) || 'Database error');
    err.status = res.status;
    throw err;
  }
  return data;
}

// Per-visitor and global limits so the public demo cannot drain Make credits or OpenAI budget.
export async function allow(action, context, perHour, globalPerDay = 0) {
  const ip = (context && context.ip) || 'unknown';
  const ip_hash = createHash('sha256').update(ip + '|' + (process.env.IP_SALT || 'roofing')).digest('hex').slice(0, 24);
  const result = await rpc('demo_rate_check', { p: { action, ip_hash, per_hour: perHour, global_per_day: globalPerDay } });
  if (result.allowed) return null;
  const message = result.reason === 'daily_limit'
    ? 'The demo has reached its daily limit. Please try again tomorrow.'
    : 'You have used all of your demo actions for this hour. Please try again later.';
  return json({ error: message, reason: result.reason }, 429);
}

// Send a request to the Make gateway scenario. The secret is added here, server-side.
export async function gateway(payload, timeoutMs = 22000) {
  try {
    const res = await fetch(env('GATEWAY_URL'), {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ ...payload, secret: env('GATEWAY_SECRET') }),
      signal: AbortSignal.timeout(timeoutMs),
    });
    const text = await res.text();
    let data;
    try { data = JSON.parse(text); } catch { data = { raw: text }; }
    return { ok: res.ok, data, timedOut: false };
  } catch (e) {
    if (e && (e.name === 'TimeoutError' || e.name === 'AbortError')) return { ok: false, data: {}, timedOut: true };
    throw e;
  }
}

export function fail(e) {
  const status = e && e.status && e.status >= 400 && e.status < 500 ? e.status : 500;
  const message = status === 500 ? 'Something went wrong on the server. Please try again.' : e.message;
  return json({ error: message }, status);
}
````

## 3. `netlify/functions/overview.mjs`

````
import { rpc, json, fail } from '../lib/util.mjs';

export default async () => {
  try {
    return json(await rpc('dashboard_overview', {}));
  } catch (e) {
    return fail(e);
  }
};

export const config = { path: '/api/overview', method: 'GET' };
````

## 4. `netlify/functions/lead.mjs`

````
import { rpc, json, fail, UUID } from '../lib/util.mjs';

export default async (req) => {
  try {
    const id = new URL(req.url).searchParams.get('id') || '';
    if (!UUID.test(id)) return json({ error: 'A valid lead id is required.' }, 400);
    return json(await rpc('dashboard_opportunity', { p: { opportunity_id: id } }));
  } catch (e) {
    return fail(e);
  }
};

export const config = { path: '/api/lead', method: 'GET' };
````

## 5. `netlify/functions/test-lead.mjs`

````
import { rpc, json, fail, allow, gateway, clean, EMAIL } from '../lib/util.mjs';

// Realistic enquiries a roofing company receives. Chosen by the visitor; text is fixed server-side.
const SCENARIOS = {
  storm_leak: {
    address_line: '42 Maple Street', city: 'Austin', region: 'TX', postcode: '78701',
    message: "Water is dripping through our bedroom ceiling after last night's storm and a few shingles blew off the back of the house. The roof is about 18 years old. Can someone come out as soon as possible?",
  },
  hail: {
    address_line: '918 Cedar Lane', city: 'Denver', region: 'CO', postcode: '80203',
    message: 'We had hail last week and I can see dents on the gutters and a few cracked tiles near the chimney. The roof is concrete tile, around 12 years old. Would like someone to look at it this week and tell us if we need a full replacement.',
  },
  replacement: {
    address_line: '27 Birchwood Drive', city: 'Columbus', region: 'OH', postcode: '43215',
    message: 'Our roof is about 22 years old and the shingles are curling at the edges. We are thinking about replacing it before winter and would like a quote. No leaks that we know of.',
  },
  gutters: {
    address_line: '5 Harbor View Road', city: 'Portland', region: 'ME', postcode: '04101',
    message: 'The gutters along the front of the house are pulling away from the fascia and overflowing when it rains. About 60 feet needs replacing. Roof itself seems fine.',
  },
};

export default async (req, context) => {
  try {
    if (req.method !== 'POST') return json({ error: 'Method not allowed' }, 405);
    const body = await req.json().catch(() => ({}));

    const scenario = SCENARIOS[body.scenario];
    if (!scenario) return json({ error: 'Choose one of the example enquiries.' }, 400);

    const name = clean(body.name, 60) || 'Demo Customer';
    const email = clean(body.email, 120);
    if (email && !EMAIL.test(email)) return json({ error: 'That email address does not look right.' }, 400);

    const limited = await allow('test_lead', context, 3, 40);
    if (limited) return limited;

    const result = await gateway({
      full_name: name,
      email,
      phone: '+1 555 0100',
      ...scenario,
      source: 'demo',
    });

    if (result.timedOut) return json({ status: 'processing', message: 'The lead is still being processed. It will appear shortly.' }, 202);
    if (!result.ok || result.data.status !== 'ok') {
      return json({ error: 'The intake automation did not accept the lead. Check that the Make gateway scenario is switched on.' }, 502);
    }

    // Send the acknowledgement now instead of waiting for the 5-minute scheduler.
    await rpc('dispatch_outbox', {}).catch(() => {});

    return json({ status: 'ok', opportunity_id: result.data.opportunity_id, lead: result.data });
  } catch (e) {
    return fail(e);
  }
};

export const config = { path: '/api/test-lead', method: 'POST' };
````

## 6. `netlify/functions/inspection-report.mjs`

````
import { rpc, json, fail, allow, gateway, clean, UUID } from '../lib/util.mjs';

export default async (req, context) => {
  try {
    if (req.method !== 'POST') return json({ error: 'Method not allowed' }, 405);
    const body = await req.json().catch(() => ({}));
    if (!UUID.test(body.opportunity_id || '')) return json({ error: 'A valid lead id is required.' }, 400);

    const notes = clean(body.notes, 2000);
    if (notes.length < 40) return json({ error: 'Add a few sentences of inspector notes (at least 40 characters).' }, 400);

    const detail = await rpc('dashboard_opportunity', { p: { opportunity_id: body.opportunity_id } });
    const open = (detail.inspections || []).find((i) => ['scheduled', 'confirmed'].includes(i.status));
    if (!open) return json({ error: 'This lead has no inspection waiting for a report.' }, 409);

    const limited = await allow('inspection_report', context, 6, 60);
    if (limited) return limited;

    const result = await gateway({ action: 'inspection_report', inspection_id: open.id, raw_notes: notes }, 25000);

    if (result.timedOut) return json({ status: 'processing', message: 'The AI is still preparing the estimate.' }, 202);
    if (!result.ok || result.data.status !== 'pending_approval') {
      return json({ error: 'The estimate automation did not accept the report. Check that the Make gateway scenario is switched on.' }, 502);
    }
    return json({ status: 'ok', estimate_id: result.data.estimate_id, total: result.data.total });
  } catch (e) {
    return fail(e);
  }
};

export const config = { path: '/api/inspection-report', method: 'POST' };
````

## 7. `netlify/functions/decide.mjs`

````
import { rpc, json, fail, allow, clean, UUID } from '../lib/util.mjs';

export default async (req, context) => {
  try {
    if (req.method !== 'POST') return json({ error: 'Method not allowed' }, 405);
    const body = await req.json().catch(() => ({}));
    if (!UUID.test(body.estimate_id || '')) return json({ error: 'A valid estimate id is required.' }, 400);
    if (!['approve', 'reject'].includes(body.decision)) return json({ error: 'Decision must be approve or reject.' }, 400);

    const limited = await allow('decide', context, 20);
    if (limited) return limited;

    const result = await rpc('decide_estimate', {
      p: { estimate_id: body.estimate_id, decision: body.decision, note: clean(body.note, 300) },
    });

    // Approved proposals go out now instead of waiting for the 5-minute scheduler.
    if (body.decision === 'approve') await rpc('dispatch_outbox', {}).catch(() => {});

    return json({ status: 'ok', result });
  } catch (e) {
    return fail(e);
  }
};

export const config = { path: '/api/decide', method: 'POST' };
````

## 8. `netlify/functions/reply.mjs`

````
import { rpc, json, fail, allow, clean, UUID } from '../lib/util.mjs';

const INTENTS = ['interested', 'question', 'declined'];

export default async (req, context) => {
  try {
    if (req.method !== 'POST') return json({ error: 'Method not allowed' }, 405);
    const body = await req.json().catch(() => ({}));
    if (!UUID.test(body.opportunity_id || '')) return json({ error: 'A valid lead id is required.' }, 400);
    const intent = INTENTS.includes(body.intent) ? body.intent : 'interested';

    const limited = await allow('reply', context, 15);
    if (limited) return limited;

    const result = await rpc('record_customer_reply', {
      p: { opportunity_id: body.opportunity_id, intent, message: clean(body.message, 500) },
    });
    return json({ status: 'ok', result });
  } catch (e) {
    return fail(e);
  }
};

export const config = { path: '/api/reply', method: 'POST' };
````

## 9. `public/index.html`

````
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Summit Roofing - Operations hub (live demo)</title>
  <meta name="description" content="A live demo of an AI operations system for a roofing company: enquiries, inspections, estimates, approvals and follow-ups in one place.">
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Archivo:wdth,wght@75..125,400..800&family=Instrument+Sans:wght@400;500;600&display=swap" rel="stylesheet">
  <link rel="stylesheet" href="/styles.css">
</head>
<body>
  <header class="topbar">
    <div class="brand">
      <svg class="brand-mark" viewBox="0 0 32 32" aria-hidden="true"><path d="M3 17 16 5l13 12" fill="none" stroke="currentColor" stroke-width="3" stroke-linecap="round" stroke-linejoin="round"/><path d="M8 16v11h16V16" fill="none" stroke="currentColor" stroke-width="3" stroke-linejoin="round"/></svg>
      <div>
        <p class="brand-name">Summit Roofing</p>
        <p class="brand-sub">Operations hub</p>
      </div>
    </div>
    <p class="demo-note">Live demo. Every customer here is fictional, and the automation behind it is real.</p>
    <button class="btn btn-primary" id="openTest" type="button">Send test lead</button>
  </header>

  <main class="page">
    <section class="summary" aria-label="Business summary" id="summary"></section>

    <div class="layout">
      <section class="board-wrap" aria-labelledby="boardTitle">
        <div class="section-head">
          <h2 id="boardTitle">Pipeline</h2>
          <p class="closed" id="closedSummary"></p>
        </div>
        <div class="board" id="board" tabindex="0" aria-label="Pipeline columns"></div>
      </section>

      <aside class="side">
        <section aria-labelledby="approvalsTitle">
          <div class="section-head"><h2 id="approvalsTitle">Needs your decision</h2></div>
          <div id="approvals" class="approvals"></div>
        </section>
        <section aria-labelledby="activityTitle">
          <div class="section-head"><h2 id="activityTitle">Latest activity</h2></div>
          <ol id="activity" class="activity"></ol>
        </section>
      </aside>
    </div>
  </main>

  <div class="scrim" id="scrim" hidden></div>
  <aside class="drawer" id="drawer" role="dialog" aria-modal="true" aria-labelledby="drawerTitle" hidden>
    <div class="drawer-bar">
      <h2 id="drawerTitle">Lead</h2>
      <button class="icon-btn" id="closeDrawer" type="button" aria-label="Close lead details">&times;</button>
    </div>
    <div class="drawer-body" id="drawerBody"></div>
  </aside>

  <dialog id="testDialog" aria-labelledby="testTitle">
    <form method="dialog" id="testForm" class="dialog-form">
      <h2 id="testTitle">Send a test lead</h2>
      <p class="muted">Pick an enquiry a roofing company might get. It enters the real system and you can follow it from first message to proposal.</p>

      <fieldset class="scenarios">
        <legend class="sr-only">Enquiry</legend>
        <label class="scenario"><input type="radio" name="scenario" value="storm_leak" checked><span><strong>Storm leak</strong><small>Water through the ceiling, shingles blown off</small></span></label>
        <label class="scenario"><input type="radio" name="scenario" value="hail"><span><strong>Hail damage</strong><small>Cracked tiles and dented gutters</small></span></label>
        <label class="scenario"><input type="radio" name="scenario" value="replacement"><span><strong>Roof replacement</strong><small>22-year-old roof, wants a quote</small></span></label>
        <label class="scenario"><input type="radio" name="scenario" value="gutters"><span><strong>Gutter repair</strong><small>Gutters pulling away from the house</small></span></label>
      </fieldset>

      <label class="field">
        <span>Customer name</span>
        <input type="text" name="name" maxlength="60" placeholder="Jordan Ellis" autocomplete="off">
      </label>
      <label class="field">
        <span>Your email (optional)</span>
        <input type="email" name="email" maxlength="120" placeholder="you@company.com" autocomplete="email">
        <small>Add it to receive the real acknowledgement and proposal emails. Follow-ups are never sent to demo leads.</small>
      </label>

      <p class="form-error" id="testError" role="alert" hidden></p>
      <div class="dialog-actions">
        <button class="btn btn-quiet" type="button" id="cancelTest">Cancel</button>
        <button class="btn btn-primary" type="submit" id="submitTest">Send test lead</button>
      </div>
    </form>
  </dialog>

  <div class="toasts" id="toasts" aria-live="polite"></div>
  <script src="/app.js" defer></script>
</body>
</html>
````

## 10. `public/styles.css`

````
:root {
  --ink: #1b2830;
  --slate: #26343d;
  --slate-2: #33444f;
  --paper: #edf0f1;
  --panel: #ffffff;
  --line: #d3d9dc;
  --line-soft: #e4e8ea;
  --muted: #5a6972;
  --patina: #1f7a6b;
  --patina-dark: #165c51;
  --patina-wash: #e2f1ee;
  --flash: #d8501f;
  --flash-wash: #fbe9e1;
  --amber: #b97a0f;
  --amber-wash: #f7eddb;
  --ok: #2f7d4f;
  --radius: 6px;
  --font-display: "Archivo", "Helvetica Neue", Arial, sans-serif;
  --font-body: "Instrument Sans", "Helvetica Neue", Arial, sans-serif;
}

* { box-sizing: border-box; }
html { -webkit-text-size-adjust: 100%; }
body {
  margin: 0;
  background: var(--paper);
  color: var(--ink);
  font-family: var(--font-body);
  font-size: 15px;
  line-height: 1.5;
}
h1, h2, h3, p, ol, ul, dl, dd { margin: 0; }
ol, ul { padding: 0; list-style: none; }
button, input, select, textarea { font: inherit; color: inherit; }
.sr-only { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0 0 0 0); white-space: nowrap; }
.muted { color: var(--muted); }
:focus-visible { outline: 3px solid var(--patina); outline-offset: 2px; }

/* ---------- top bar ---------- */
.topbar {
  display: flex; align-items: center; gap: 24px;
  padding: 14px 28px;
  background: var(--slate); color: #fff;
}
.brand { display: flex; align-items: center; gap: 12px; }
.brand-mark { width: 30px; height: 30px; color: #8fd3c6; flex: none; }
.brand-name { font-family: var(--font-display); font-stretch: 90%; font-weight: 700; font-size: 19px; line-height: 1.1; }
.brand-sub { font-size: 13px; color: #aebcc4; }
.demo-note { flex: 1; font-size: 13px; color: #c5d0d6; max-width: 46ch; }

.btn {
  display: inline-flex; align-items: center; justify-content: center; gap: 8px;
  min-height: 40px; padding: 8px 16px;
  border: 1px solid transparent; border-radius: var(--radius);
  font-weight: 600; font-size: 14px; cursor: pointer;
  transition: background-color .15s, border-color .15s;
}
.btn:disabled { opacity: .55; cursor: not-allowed; }
.btn-primary { background: var(--patina); color: #fff; }
.btn-primary:hover:not(:disabled) { background: var(--patina-dark); }
.topbar .btn-primary { background: #35a592; color: #0f2a25; }
.topbar .btn-primary:hover:not(:disabled) { background: #52b9a7; }
.btn-quiet { background: transparent; border-color: var(--line); color: var(--ink); }
.btn-quiet:hover:not(:disabled) { background: var(--line-soft); }
.btn-danger { background: transparent; border-color: #e3b7a5; color: #a43a14; }
.btn-danger:hover:not(:disabled) { background: var(--flash-wash); }
.btn-small { min-height: 34px; padding: 5px 12px; font-size: 13px; }

/* ---------- page ---------- */
.page { max-width: 1480px; margin: 0 auto; padding: 22px 28px 48px; }

.summary {
  display: grid; grid-template-columns: repeat(5, minmax(0, 1fr));
  background: var(--panel); border: 1px solid var(--line); border-radius: var(--radius);
  margin-bottom: 22px;
}
.kpi { padding: 14px 18px; border-left: 1px solid var(--line-soft); }
.kpi:first-child { border-left: 0; }
.kpi-value { font-family: var(--font-display); font-stretch: 92%; font-weight: 700; font-size: 26px; line-height: 1.15; }
.kpi-label { font-size: 13px; color: var(--muted); }
.kpi-attn .kpi-value { color: var(--flash); }

.layout { display: grid; grid-template-columns: minmax(0, 1fr) 330px; gap: 22px; align-items: start; }
.layout > * { min-width: 0; }
.section-head { display: flex; align-items: baseline; justify-content: space-between; gap: 12px; margin-bottom: 10px; }
.section-head h2 { font-family: var(--font-display); font-stretch: 90%; font-weight: 700; font-size: 18px; }
.closed { font-size: 13px; color: var(--muted); }

/* ---------- pipeline board ---------- */
.board { display: grid; grid-auto-flow: column; grid-auto-columns: minmax(176px, 1fr); gap: 12px; overflow-x: auto; padding-bottom: 8px; }
.col { background: #e3e8ea; border-radius: var(--radius); border-top: 4px solid var(--slate-2); display: flex; flex-direction: column; min-height: 140px; max-height: 72vh; }
.col-attn { border-top-color: var(--flash); }
.col-head { padding: 10px 12px 6px; display: flex; justify-content: space-between; align-items: baseline; gap: 8px; }
.col-title { font-weight: 600; font-size: 14px; }
.col-count { font-family: var(--font-display); font-weight: 700; font-size: 14px; color: var(--muted); }
.col-list { padding: 4px 8px 10px; overflow-y: auto; display: grid; gap: 8px; align-content: start; }
.col-empty { padding: 6px 12px 12px; font-size: 13px; color: var(--muted); }

.card {
  display: block; width: 100%; text-align: left;
  background: var(--panel); border: 1px solid var(--line); border-left: 4px solid var(--slate-2);
  border-radius: var(--radius); padding: 10px 12px; cursor: pointer;
}
.card:hover { border-color: #aab5bb; border-left-color: inherit; }
.card.u-emergency { border-left-color: var(--flash); }
.card.u-high { border-left-color: var(--amber); }
.card.u-low { border-left-color: #a7b3b9; }
.card-name { font-weight: 600; line-height: 1.3; }
.card-meta { font-size: 13px; color: var(--muted); }
.card-row { display: flex; justify-content: space-between; align-items: center; gap: 8px; margin-top: 6px; }
.card-value { font-family: var(--font-display); font-weight: 700; font-size: 14px; }

.chip { display: inline-block; padding: 1px 8px; border-radius: 999px; font-size: 12px; font-weight: 600; background: var(--line-soft); color: var(--slate-2); white-space: nowrap; }
.chip-urgent { background: var(--flash-wash); color: #a43a14; }
.chip-high { background: var(--amber-wash); color: #7d5207; }
.chip-ok { background: var(--patina-wash); color: var(--patina-dark); }
.chip-warn { background: var(--amber-wash); color: #7d5207; }
.chip-bad { background: var(--flash-wash); color: #a43a14; }

/* ---------- side panels ---------- */
.side { display: grid; gap: 22px; }
.approvals { display: grid; gap: 10px; }
.approval { background: var(--panel); border: 1px solid var(--line); border-left: 4px solid var(--flash); border-radius: var(--radius); padding: 12px 14px; }
.approval-top { display: flex; justify-content: space-between; gap: 10px; align-items: baseline; }
.approval-name { font-weight: 600; }
.approval-total { font-family: var(--font-display); font-weight: 700; font-size: 18px; }
.approval p { font-size: 13px; color: var(--muted); margin-top: 2px; }
.approval-actions { display: flex; gap: 8px; margin-top: 10px; flex-wrap: wrap; }
.empty { background: var(--panel); border: 1px dashed var(--line); border-radius: var(--radius); padding: 16px; color: var(--muted); font-size: 14px; }

.activity { background: var(--panel); border: 1px solid var(--line); border-radius: var(--radius); }
.activity li { display: grid; grid-template-columns: 1fr auto; gap: 2px 12px; padding: 9px 14px; border-top: 1px solid var(--line-soft); }
.activity li:first-child { border-top: 0; }
.activity button { all: unset; cursor: pointer; font-size: 14px; }
.activity button:hover .act-what { text-decoration: underline; }
.activity button:focus-visible { outline: 3px solid var(--patina); outline-offset: 2px; }
.act-who { font-size: 12px; color: var(--muted); }
.act-time { font-size: 12px; color: var(--muted); white-space: nowrap; align-self: start; }

/* ---------- drawer ---------- */
.scrim { position: fixed; inset: 0; background: rgba(20, 30, 36, .45); z-index: 20; }
.drawer {
  position: fixed; top: 0; right: 0; bottom: 0; z-index: 30;
  width: min(620px, 100%); background: var(--paper);
  box-shadow: -12px 0 32px rgba(20, 30, 36, .22);
  display: flex; flex-direction: column;
  animation: slide-in .22s ease-out;
}
.drawer[hidden], .scrim[hidden] { display: none; }
@keyframes slide-in { from { transform: translateX(24px); opacity: 0; } to { transform: none; opacity: 1; } }
.drawer-bar { display: flex; align-items: center; justify-content: space-between; padding: 14px 20px; background: var(--slate); color: #fff; }
.drawer-bar h2 { font-family: var(--font-display); font-stretch: 90%; font-size: 18px; font-weight: 700; }
.icon-btn { background: none; border: 0; color: inherit; font-size: 28px; line-height: 1; width: 40px; height: 40px; cursor: pointer; border-radius: var(--radius); }
.icon-btn:hover { background: rgba(255, 255, 255, .12); }
.drawer-body { overflow-y: auto; padding: 18px 20px 40px; display: grid; gap: 16px; align-content: start; }

.panel { background: var(--panel); border: 1px solid var(--line); border-radius: var(--radius); padding: 14px 16px; }
.panel h3 { font-family: var(--font-display); font-stretch: 90%; font-weight: 700; font-size: 15px; margin-bottom: 8px; }
.facts { display: grid; grid-template-columns: auto 1fr; gap: 3px 14px; font-size: 14px; }
.facts dt { color: var(--muted); }
.lead-head h3 { font-size: 20px; margin: 0; }
.lead-head .chips { display: flex; gap: 6px; flex-wrap: wrap; margin-top: 6px; }
.summary-text { font-size: 15px; }

/* journey strip: the roof-ridge steps */
.journey { display: flex; overflow-x: auto; padding: 2px 0 6px; }
.step {
  flex: 1 0 96px; position: relative;
  padding: 10px 12px 10px 22px; margin-right: -10px;
  background: #dfe5e8; color: var(--muted);
  font-size: 12.5px; font-weight: 600; line-height: 1.25;
  clip-path: polygon(0 0, calc(100% - 12px) 0, 100% 50%, calc(100% - 12px) 100%, 0 100%, 12px 50%);
  display: flex; align-items: center;
}
.step:first-child { clip-path: polygon(0 0, calc(100% - 12px) 0, 100% 50%, calc(100% - 12px) 100%, 0 100%); padding-left: 14px; }
.step.done { background: var(--patina); color: #fff; }
.step.current { background: var(--flash); color: #fff; }

.actions { border-left: 4px solid var(--flash); }
.actions h3 { margin-bottom: 4px; }
.actions p { font-size: 14px; color: var(--muted); margin-bottom: 10px; }
.actions textarea, .field textarea { width: 100%; min-height: 150px; padding: 10px; border: 1px solid var(--line); border-radius: var(--radius); background: #fff; resize: vertical; }
.row-actions { display: flex; gap: 8px; flex-wrap: wrap; margin-top: 10px; align-items: center; }

table.items { width: 100%; border-collapse: collapse; font-size: 14px; }
table.items th { text-align: left; font-weight: 600; color: var(--muted); padding: 4px 0; border-bottom: 1px solid var(--line); }
table.items td { padding: 6px 0; border-bottom: 1px solid var(--line-soft); vertical-align: top; }
table.items .num { text-align: right; white-space: nowrap; padding-left: 12px; }
table.items tfoot td { border-bottom: 0; padding: 3px 0; }
table.items tfoot .total td { font-weight: 700; font-size: 15px; padding-top: 6px; border-top: 1px solid var(--ink); }

.list { display: grid; gap: 0; font-size: 14px; }
.list li { display: flex; justify-content: space-between; gap: 12px; padding: 7px 0; border-top: 1px solid var(--line-soft); align-items: baseline; }
.list li:first-child { border-top: 0; }
.list .sub { color: var(--muted); font-size: 13px; }
details.preview { margin-top: 4px; }
details.preview summary { cursor: pointer; color: var(--patina-dark); font-size: 13px; }
details.preview pre { white-space: pre-wrap; font-family: var(--font-body); font-size: 13px; background: var(--paper); border-radius: var(--radius); padding: 10px; margin: 6px 0 0; }

.timeline { display: grid; gap: 0; }
.timeline li { display: grid; grid-template-columns: 14px 1fr auto; gap: 0 10px; padding: 6px 0; font-size: 14px; position: relative; }
.timeline li::before { content: ""; width: 9px; height: 9px; border-radius: 50%; background: var(--slate-2); margin-top: 6px; }
.timeline li.t-ai::before { background: var(--patina); }
.timeline li.t-human::before { background: var(--flash); }
.timeline li.t-customer::before { background: var(--amber); }
.timeline .t-time { font-size: 12px; color: var(--muted); white-space: nowrap; }
.timeline .t-detail { grid-column: 2 / 4; font-size: 13px; color: var(--muted); }

/* ---------- dialog ---------- */
dialog { border: 0; border-radius: 8px; padding: 0; width: min(560px, calc(100% - 24px)); box-shadow: 0 20px 50px rgba(20, 30, 36, .35); background: var(--panel); color: var(--ink); }
dialog::backdrop { background: rgba(20, 30, 36, .5); }
.dialog-form { padding: 22px 24px 20px; display: grid; gap: 14px; }
.dialog-form h2 { font-family: var(--font-display); font-stretch: 90%; font-size: 21px; font-weight: 700; }
.scenarios { border: 0; padding: 0; margin: 0; display: grid; gap: 8px; grid-template-columns: 1fr 1fr; }
.scenario { display: flex; gap: 10px; align-items: flex-start; border: 1px solid var(--line); border-radius: var(--radius); padding: 10px 12px; cursor: pointer; }
.scenario input { margin-top: 4px; accent-color: var(--patina); }
.scenario strong { display: block; font-size: 14px; }
.scenario small { color: var(--muted); font-size: 12.5px; line-height: 1.35; display: block; }
.scenario:has(input:checked) { border-color: var(--patina); background: var(--patina-wash); }
.field { display: grid; gap: 4px; font-size: 14px; }
.field > span { font-weight: 600; }
.field input[type="text"], .field input[type="email"] { min-height: 40px; padding: 8px 10px; border: 1px solid var(--line); border-radius: var(--radius); background: #fff; }
.field small { color: var(--muted); font-size: 12.5px; }
.form-error { color: #a43a14; background: var(--flash-wash); border-radius: var(--radius); padding: 8px 12px; font-size: 14px; }
.dialog-actions { display: flex; justify-content: flex-end; gap: 10px; }

/* ---------- toasts ---------- */
.toasts { position: fixed; left: 50%; bottom: 20px; transform: translateX(-50%); z-index: 60; display: grid; gap: 8px; width: min(440px, calc(100% - 24px)); }
.toast { background: var(--ink); color: #fff; padding: 11px 16px; border-radius: var(--radius); font-size: 14px; box-shadow: 0 8px 24px rgba(20, 30, 36, .3); }
.toast.error { background: #8c2c0d; }

.loading { color: var(--muted); padding: 20px 0; }

/* ---------- responsive ---------- */
@media (max-width: 1100px) {
  .layout { grid-template-columns: minmax(0, 1fr); }
}
@media (max-width: 760px) {
  .topbar { flex-wrap: wrap; padding: 12px 16px; gap: 10px 16px; }
  .demo-note { order: 3; flex-basis: 100%; max-width: none; }
  .topbar .btn { margin-left: auto; }
  .page { padding: 16px 14px 40px; }
  .summary { grid-template-columns: 1fr 1fr; }
  .kpi { border-left: 0; border-top: 1px solid var(--line-soft); }
  .kpi:nth-child(-n+2) { border-top: 0; }
  .kpi:nth-child(even) { border-left: 1px solid var(--line-soft); }
  .board { grid-auto-columns: 82%; scroll-snap-type: x mandatory; }
  .col { scroll-snap-align: start; max-height: none; }
  .scenarios { grid-template-columns: 1fr; }
  .drawer-body { padding: 14px 14px 40px; }
}
@media (prefers-reduced-motion: reduce) {
  .drawer { animation: none; }
  .btn { transition: none; }
}
````

## 11. `public/app.js`

````
(() => {
  'use strict';

  // ---------- small helpers ----------
  const $ = (sel, root = document) => root.querySelector(sel);
  const esc = (v) => String(v ?? '').replace(/[&<>"']/g, (c) => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));
  const usd0 = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 });
  const usd2 = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD', minimumFractionDigits: 2, maximumFractionDigits: 2 });
  const money = (n) => (n == null || n === '' ? '' : usd0.format(Number(n)));
  const money2 = (n) => (n == null || n === '' ? '' : usd2.format(Number(n)));
  const dateFmt = new Intl.DateTimeFormat(undefined, { weekday: 'short', month: 'short', day: 'numeric', hour: 'numeric', minute: '2-digit' });
  const when = (ts) => (ts ? dateFmt.format(new Date(ts)) : '');

  function ago(ts) {
    if (!ts) return '';
    const s = Math.round((Date.now() - new Date(ts).getTime()) / 1000);
    if (s < 45) return 'just now';
    if (s < 3600) return Math.round(s / 60) + ' min ago';
    if (s < 86400) return Math.round(s / 3600) + ' h ago';
    return Math.round(s / 86400) + ' d ago';
  }

  const SERVICE = { repair: 'Repair', replacement: 'Replacement', storm_damage: 'Storm damage', inspection_only: 'Inspection', gutter_work: 'Gutters' };
  const URGENCY = { emergency: 'Urgent', high: 'High priority', normal: 'Standard', low: 'Low priority' };
  const STAGE_LABEL = {
    new: 'New', inspection_scheduled: 'Inspection booked', inspection_done: 'Inspected',
    proposal_pending_approval: 'Needs approval', proposal_sent: 'Proposal sent', won: 'Won', lost: 'Lost',
  };
  const BOARD_STAGES = ['new', 'inspection_scheduled', 'inspection_done', 'proposal_pending_approval', 'proposal_sent'];

  const EVENT_LABEL = {
    lead_created: 'Enquiry received', lead_received: 'Enquiry received',
    ai_triage_completed: 'AI triaged the enquiry', ai_classified: 'AI triaged the enquiry',
    salesperson_assigned: 'Salesperson assigned', inspection_scheduled: 'Inspection booked',
    acknowledgement_queued: 'Acknowledgement email queued', acknowledgement_sent: 'Acknowledgement email sent',
    inspection_completed: 'Inspection completed', report_analyzed: 'AI analysed the inspection report',
    estimate_drafted: 'AI drafted the estimate', approval_requested: 'Manager approval requested',
    estimate_approved: 'Estimate approved', estimate_rejected: 'Estimate rejected',
    proposal_queued: 'Proposal email queued', proposal_sent: 'Proposal emailed to customer',
    proposal_blocked: 'Proposal blocked: no email address', follow_ups_scheduled: 'Follow-ups scheduled',
    follow_up_sent: 'Follow-up sent', customer_replied: 'Customer replied', email_failed: 'Email failed',
  };
  const eventLabel = (t) => EVENT_LABEL[t] || String(t || '').replace(/_/g, ' ').replace(/^./, (c) => c.toUpperCase());

  // Example inspector notes, used to pre-fill the "simulate inspector report" form.
  const NOTES = {
    storm_damage: 'Arrived mid-morning. Two-story home, asphalt shingle roof, roughly 2,100 sq ft of roof area, about 18 years old. Active leak confirmed above the master bedroom: wet decking and stained ceiling drywall. About 40 shingles missing on the south slope from the storm, creased shingles near the ridge, lifted flashing around the chimney. Heavy granule loss across the whole roof. No structural sag. Recommend an emergency tarp today and a full replacement within a few weeks. About 3 sheets of decking under the leak need replacing.',
    repair: 'On site for the leak report. Single-story home, asphalt shingle roof, about 1,600 sq ft, roughly 14 years old. Found a failed pipe boot flashing on the north slope causing a slow leak into the hallway ceiling. Eight shingles around it are cracked. Decking is damp but sound. Recommend replacing the pipe boot, 8 shingles and sealing the area. No other issues found, rest of the roof is in fair condition.',
    replacement: 'Inspected the full roof. Two-story colonial, asphalt shingles, around 2,400 sq ft of roof area and about 22 years old. Shingles are curling at the edges across every slope with heavy granule loss and several soft spots near the valleys. No active leaks, attic is dry. Flashing is corroded at the chimney. Roof is at end of life. Recommend a full tear-off and replacement with new underlayment and new chimney flashing before winter.',
    gutter_work: 'Checked the gutters and fascia along the front and sides. About 60 linear feet of gutter has pulled away from the fascia with several loose hangers and sagging sections. Fascia board is sound apart from two short soft spots. Roof itself is in good condition with no leaks. Recommend replacing the 60 feet of gutter with new hangers and repairing the two soft fascia sections.',
    inspection_only: 'Completed a full roof inspection at the owner\'s request ahead of a sale. Single-story ranch, asphalt shingles, roughly 1,700 sq ft and about 9 years old. Roof is in good condition: no leaks, minimal granule loss, flashing intact. A few raised nails and one cracked shingle near the ridge. Recommend a minor repair to the cracked shingle and re-securing the raised nails. No further work needed.',
  };

  // ---------- state ----------
  const state = { overview: null, openId: null, detail: null, opener: null, burstTimer: null, busy: false };

  // ---------- network ----------
  async function api(path, body) {
    const init = body ? { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(body) } : undefined;
    const res = await fetch(path, init);
    let data = null;
    try { data = await res.json(); } catch (e) { /* non-JSON error page */ }
    if (!res.ok) {
      const err = new Error((data && data.error) || 'Something went wrong. Please try again.');
      err.status = res.status;
      throw err;
    }
    return data;
  }

  function toast(message, isError) {
    const el = document.createElement('div');
    el.className = 'toast' + (isError ? ' error' : '');
    el.textContent = message;
    if (isError) el.setAttribute('role', 'alert');
    $('#toasts').appendChild(el);
    setTimeout(() => el.remove(), isError ? 7000 : 4500);
  }

  // ---------- overview ----------
  async function loadOverview() {
    try {
      state.overview = await api('/api/overview');
      renderOverview();
    } catch (e) {
      if (!state.overview) {
        $('#board').innerHTML = '<p class="loading">The dashboard could not load its data. Check the Netlify environment variables, then refresh.</p>';
      }
    }
  }

  function renderOverview() {
    const o = state.overview;
    if (!o) return;
    renderSummary(o.kpis);
    renderBoard(o);
    renderApprovals(o.approvals);
    renderActivity(o.activity);
  }

  function renderSummary(k) {
    const items = [
      { v: k.open_leads, l: 'Open leads' },
      { v: money(k.pipeline_value), l: 'Out with customers or awaiting approval' },
      { v: k.awaiting_approval, l: 'Waiting for your approval', attn: Number(k.awaiting_approval) > 0 },
      { v: k.follow_ups_scheduled, l: 'Follow-ups scheduled' },
      { v: money(k.won_value), l: 'Won so far' },
    ];
    $('#summary').innerHTML = items.map((i) =>
      `<div class="kpi${i.attn ? ' kpi-attn' : ''}"><p class="kpi-value">${esc(i.v)}</p><p class="kpi-label">${esc(i.l)}</p></div>`
    ).join('');
  }

  function cardHtml(l) {
    const urgent = l.urgency === 'emergency' ? '<span class="chip chip-urgent">Urgent</span>' : l.urgency === 'high' ? '<span class="chip chip-high">High</span>' : '';
    const place = [l.city, l.region].filter(Boolean).join(', ');
    return `<button type="button" class="card u-${esc(l.urgency)}" data-open="${esc(l.id)}">
      <p class="card-name">${esc(l.customer)}</p>
      <p class="card-meta">${esc(place)}</p>
      <div class="card-row"><span class="chip">${esc(SERVICE[l.service_type] || l.service_type)}</span>${urgent}</div>
      <div class="card-row"><span class="card-value">${esc(money(l.estimated_value))}</span><span class="card-meta">${esc(ago(l.last_contact_at || l.created_at))}</span></div>
    </button>`;
  }

  function renderBoard(o) {
    const byStage = {};
    o.leads.forEach((l) => { (byStage[l.stage] = byStage[l.stage] || []).push(l); });
    $('#board').innerHTML = BOARD_STAGES.map((s) => {
      const list = byStage[s] || [];
      const attn = s === 'proposal_pending_approval' && list.length ? ' col-attn' : '';
      return `<section class="col${attn}" aria-label="${esc(STAGE_LABEL[s])}">
        <div class="col-head"><h3 class="col-title">${esc(STAGE_LABEL[s])}</h3><span class="col-count">${list.length}</span></div>
        ${list.length ? `<div class="col-list">${list.map(cardHtml).join('')}</div>` : '<p class="col-empty">Nothing here right now.</p>'}
      </section>`;
    }).join('');

    const won = o.stages.find((s) => s.stage === 'won') || { count: 0, value: 0 };
    const lost = o.stages.find((s) => s.stage === 'lost') || { count: 0 };
    $('#closedSummary').textContent = `Won: ${won.count} jobs worth ${money(won.value)}. Lost: ${lost.count}.`;
  }

  function renderApprovals(list) {
    const el = $('#approvals');
    if (!list.length) {
      el.innerHTML = '<p class="empty">No estimates are waiting. When an inspector files a report, the AI drafts an estimate and it lands here for you to approve.</p>';
      return;
    }
    el.innerHTML = list.map((a) => {
      const summary = (a.ai_summary || '').length > 150 ? a.ai_summary.slice(0, 147) + '...' : (a.ai_summary || '');
      return `<article class="approval">
        <div class="approval-top"><span class="approval-name">${esc(a.customer)}</span><span class="approval-total">${esc(money(a.total))}</span></div>
        <p>${esc(SERVICE[a.service_type] || a.service_type)} in ${esc([a.city, a.region].filter(Boolean).join(', '))}</p>
        <p>${esc(summary)}</p>
        <div class="approval-actions">
          <button class="btn btn-primary btn-small" type="button" data-approve="${esc(a.estimate_id)}">Approve and send</button>
          <button class="btn btn-quiet btn-small" type="button" data-open="${esc(a.opportunity_id)}">Review</button>
        </div>
      </article>`;
    }).join('');
  }

  function renderActivity(list) {
    $('#activity').innerHTML = list.slice(0, 14).map((e) =>
      `<li><button type="button" data-open="${esc(e.opportunity_id)}"><span class="act-what">${esc(eventLabel(e.event_type))}</span><br><span class="act-who">${esc(e.customer || '')}</span></button><span class="act-time">${esc(ago(e.created_at))}</span></li>`
    ).join('') || '<li><span class="act-who">No activity yet.</span></li>';
  }

  // ---------- lead drawer ----------
  function journeyHtml(d) {
    const o = d.opportunity;
    const raw = [
      true,
      !!o.ai_summary || d.events.some((e) => e.event_type === 'ai_triage_completed'),
      d.inspections.length > 0,
      d.inspections.some((i) => i.report),
      d.estimates.length > 0,
      d.estimates.some((e) => ['approved', 'sent', 'accepted'].includes(e.status)),
      ['proposal_sent', 'won'].includes(o.stage) || d.estimates.some((e) => ['sent', 'accepted'].includes(e.status)),
      d.follow_ups.length > 0 || o.stage === 'won',
    ];
    let last = -1;
    raw.forEach((v, i) => { if (v) last = i; });
    if (o.stage === 'won') last = raw.length - 1;
    const labels = ['Enquiry', 'AI triage', 'Inspection booked', 'Report analysed', 'Estimate drafted', 'Approved', 'Proposal sent', 'Follow-ups'];
    return `<div class="journey" role="list" aria-label="Lead journey">${labels.map((l, i) => {
      const cls = i <= last ? 'done' : i === last + 1 && o.stage !== 'lost' ? 'current' : '';
      const status = cls === 'done' ? 'done' : cls === 'current' ? 'next' : 'upcoming';
      return `<div class="step ${cls}" role="listitem"><span class="sr-only">${status}: </span>${esc(l)}</div>`;
    }).join('')}</div>`;
  }

  function actionsHtml(d) {
    const o = d.opportunity;
    const openInsp = d.inspections.find((i) => ['scheduled', 'confirmed'].includes(i.status));
    const pending = d.estimates.find((e) => e.status === 'pending_approval');

    if (openInsp) {
      const notes = NOTES[o.service_type] || NOTES.repair;
      return `<section class="panel actions"><h3>Next step: inspection report</h3>
        <p>In real life the inspector files notes from the roof. Edit the example below or send it as it is, and the AI will turn it into a priced draft estimate.</p>
        <textarea id="reportNotes" aria-label="Inspector notes" maxlength="2000">${esc(notes)}</textarea>
        <div class="row-actions"><button class="btn btn-primary" type="button" data-action="send-report">Send report to the AI</button></div>
      </section>`;
    }
    if (pending) {
      return `<section class="panel actions"><h3>Your decision</h3>
        <p>The AI drafted this estimate from the inspection. Nothing reaches the customer until you approve it.</p>
        <div class="row-actions">
          <button class="btn btn-primary" type="button" data-action="approve" data-estimate="${esc(pending.id)}">Approve and send proposal</button>
          <button class="btn btn-danger" type="button" data-action="reject-toggle">Reject</button>
        </div>
        <div id="rejectBox" hidden>
          <label class="field" style="margin-top:12px"><span>Reason for rejecting</span><input type="text" id="rejectNote" maxlength="300" placeholder="For example: price too low for the roof size"></label>
          <div class="row-actions"><button class="btn btn-danger btn-small" type="button" data-action="reject" data-estimate="${esc(pending.id)}">Confirm rejection</button></div>
        </div>
      </section>`;
    }
    if (o.stage === 'proposal_sent' && d.follow_ups.some((f) => f.status === 'pending')) {
      return `<section class="panel actions"><h3>Try it: the customer replies</h3>
        <p>Follow-ups are scheduled for Day 2, 5 and 10. As soon as a customer replies, they all stop and the team is told.</p>
        <label class="field"><span>Reply</span><textarea id="replyText" maxlength="500" style="min-height:70px">Looks good, when could you start?</textarea></label>
        <div class="row-actions">
          <select id="replyIntent" aria-label="What the reply means"><option value="interested">Interested</option><option value="question">Has a question</option><option value="declined">Not interested</option></select>
          <button class="btn btn-primary" type="button" data-action="reply">Log the reply</button>
        </div>
      </section>`;
    }
    return '';
  }

  function factsHtml(o) {
    const rows = [
      ['Service', SERVICE[o.service_type] || o.service_type],
      ['Priority', URGENCY[o.urgency] || o.urgency],
      ['Property', o.property_type],
      ['Roof', [o.roof_type, o.roof_age_years ? o.roof_age_years + ' years old' : ''].filter(Boolean).join(', ')],
    ].filter((r) => r[1]);
    return `<dl class="facts">${rows.map((r) => `<dt>${esc(r[0])}</dt><dd>${esc(r[1])}</dd>`).join('')}</dl>`;
  }

  function inspectionHtml(d) {
    if (!d.inspections.length) return '';
    return `<section class="panel"><h3>Inspection</h3>` + d.inspections.map((i) => {
      const x = (i.report && i.report.extracted) || {};
      const facts = [
        ['Condition', x.roof_condition], ['Damage', x.damage],
        ['Roof area', x.roof_area_sqft ? x.roof_area_sqft + ' sq ft' : ''],
        ['Recommended work', x.recommended_work],
        ['Safety', Array.isArray(x.safety_issues) && x.safety_issues.length ? x.safety_issues.join(', ') : ''],
      ].filter((r) => r[1]);
      const status = i.status === 'completed' ? `Completed ${esc(ago(i.completed_at))}` : `Booked for ${esc(when(i.scheduled_at))}`;
      return `<p><strong>${esc(i.inspector || 'Inspector')}</strong> <span class="muted">${status}</span></p>` +
        (i.report ? `<p class="summary-text" style="margin-top:8px">${esc(i.report.ai_summary)}</p>` : '') +
        (facts.length ? `<dl class="facts" style="margin-top:8px">${facts.map((r) => `<dt>${esc(r[0])}</dt><dd>${esc(r[1])}</dd>`).join('')}</dl>` : '');
    }).join('') + '</section>';
  }

  function estimateHtml(d) {
    if (!d.estimates.length) return '';
    const e = d.estimates[d.estimates.length - 1];
    const tone = { pending_approval: 'chip-warn', approved: 'chip-ok', sent: 'chip-ok', accepted: 'chip-ok', rejected: 'chip-bad', declined: 'chip-bad' }[e.status] || '';
    const labels = { pending_approval: 'Waiting for approval', approved: 'Approved', sent: 'Sent to customer', accepted: 'Accepted', rejected: 'Rejected', declined: 'Declined', draft: 'Draft' };
    const items = Array.isArray(e.line_items) ? e.line_items : [];
    return `<section class="panel"><h3>Estimate <span class="chip ${tone}" style="margin-left:6px">${esc(labels[e.status] || e.status)}</span></h3>
      ${items.length ? `<table class="items"><thead><tr><th>Work</th><th class="num">Qty</th><th class="num">Price</th></tr></thead><tbody>${items.map((i) =>
        `<tr><td>${esc(i.item)}</td><td class="num">${esc(i.qty)}</td><td class="num">${esc(money2(i.unit_price))}</td></tr>`).join('')}</tbody>
        <tfoot><tr><td colspan="2">Subtotal</td><td class="num">${esc(money2(e.subtotal))}</td></tr>
        <tr><td colspan="2">Tax</td><td class="num">${esc(money2(e.tax))}</td></tr>
        <tr class="total"><td colspan="2">Total</td><td class="num">${esc(money2(e.total))}</td></tr>
        <tr><td colspan="2" class="muted">Deposit to book the job</td><td class="num muted">${esc(money2(e.deposit_amount))}</td></tr></tfoot></table>`
        : '<p class="muted">No priced line items yet.</p>'}
      ${e.ai_notes ? `<p class="muted" style="margin-top:8px;font-size:13px">${esc(e.ai_notes)}</p>` : ''}
    </section>`;
  }

  function emailsHtml(d) {
    if (!d.emails.length) return '';
    const kind = { acknowledgement: 'Acknowledgement', proposal: 'Proposal', follow_up: 'Follow-up' };
    return `<section class="panel"><h3>Emails</h3><ul class="list">${d.emails.map((m) => {
      let chip;
      if (m.status === 'sent') chip = m.simulated ? '<span class="chip chip-ok">Sent (simulated)</span>' : '<span class="chip chip-ok">Sent</span>';
      else if (m.status === 'cancelled') chip = '<span class="chip">Preview only</span>';
      else if (m.status === 'failed') chip = '<span class="chip chip-bad">Failed</span>';
      else chip = '<span class="chip chip-warn">Queued</span>';
      return `<li><span>${esc(kind[m.kind] || m.kind)}<span class="sub"><br>${esc(m.subject)}</span></span>${chip}</li>`;
    }).join('')}</ul></section>`;
  }

  function followUpsHtml(d) {
    if (!d.follow_ups.length) return '';
    const label = { pending: 'Scheduled', sent: 'Sent', skipped: 'Preview only', cancelled: 'Cancelled' };
    const tone = { pending: 'chip-warn', sent: 'chip-ok', skipped: '', cancelled: 'chip-bad' };
    return `<section class="panel"><h3>Follow-up sequence</h3><ul class="list">${d.follow_ups.map((f) =>
      `<li><span>Day ${esc(f.sequence_day)}<span class="sub"> ${esc(when(f.scheduled_for))}</span>
        <details class="preview"><summary>Preview message</summary><pre>${esc(f.message_body)}</pre></details></span>
        <span class="chip ${tone[f.status] || ''}">${esc(label[f.status] || f.status)}</span></li>`).join('')}</ul></section>`;
  }

  function eventDetail(e) {
    const x = e.details || {};
    switch (e.event_type) {
      case 'salesperson_assigned': return x.salesperson ? 'Assigned to ' + x.salesperson : '';
      case 'inspection_scheduled': return [x.inspector, x.scheduled_at ? when(x.scheduled_at) : ''].filter(Boolean).join(', ');
      case 'estimate_drafted': return x.total != null ? 'Total ' + money2(x.total) + ', ' + x.line_items + ' line items' : '';
      case 'estimate_approved': return [x.approved_by ? 'Approved by ' + x.approved_by : '', x.note].filter(Boolean).join('. ');
      case 'estimate_rejected': return [x.rejected_by ? 'Rejected by ' + x.rejected_by : '', x.note].filter(Boolean).join('. ');
      case 'customer_replied': return [x.message ? '"' + x.message + '"' : '', x.intent ? 'Intent: ' + x.intent : '', x.follow_ups_cancelled != null ? x.follow_ups_cancelled + ' follow-ups cancelled' : ''].filter(Boolean).join('. ');
      case 'email_failed': return x.error || '';
      default: return '';
    }
  }

  function timelineHtml(d) {
    const ev = d.events.slice().sort((a, b) => new Date(a.created_at) - new Date(b.created_at));
    if (!ev.length) return '';
    return `<section class="panel"><h3>Timeline</h3><ol class="timeline">${ev.map((e) => {
      const det = eventDetail(e);
      return `<li class="t-${esc(e.actor)}"><span>${esc(eventLabel(e.event_type))}</span><span class="t-time">${esc(ago(e.created_at))}</span>${det ? `<span class="t-detail">${esc(det)}</span>` : ''}</li>`;
    }).join('')}</ol></section>`;
  }

  function renderDrawer(d) {
    const o = d.opportunity;
    const c = o.customer;
    $('#drawerTitle').textContent = c.name;
    const urgent = o.urgency === 'emergency' ? '<span class="chip chip-urgent">Urgent</span>' : o.urgency === 'high' ? '<span class="chip chip-high">High priority</span>' : '';
    const contact = [c.email, c.phone].filter(Boolean).join(', ');
    $('#drawerBody').innerHTML = `
      <section class="panel lead-head">
        <h3>${esc(c.name)}</h3>
        <p class="muted">${esc(c.address)}</p>
        <p class="muted">${esc(contact)}</p>
        <div class="chips"><span class="chip chip-ok">${esc(STAGE_LABEL[o.stage] || o.stage)}</span>${urgent}<span class="chip">${esc(SERVICE[o.service_type] || o.service_type)}</span><span class="chip">via ${esc(String(c.source || '').replace('_', ' '))}</span></div>
        ${o.salesperson ? `<p class="muted" style="margin-top:8px">Salesperson: ${esc(o.salesperson)}</p>` : ''}
      </section>
      <section class="panel"><h3>Where this lead is</h3>${journeyHtml(d)}</section>
      ${actionsHtml(d)}
      <section class="panel"><h3>What the AI understood</h3>
        ${o.ai_summary ? `<p class="summary-text">${esc(o.ai_summary)}</p>` : '<p class="muted">No AI summary yet.</p>'}
        <div style="margin-top:10px">${factsHtml(o)}</div>
        ${o.description ? `<details class="preview"><summary>Original message</summary><pre>${esc(o.description)}</pre></details>` : ''}
      </section>
      ${inspectionHtml(d)}${estimateHtml(d)}${emailsHtml(d)}${followUpsHtml(d)}${timelineHtml(d)}`;
  }

  async function loadDetail(id, keepScroll) {
    try {
      const d = await api('/api/lead?id=' + encodeURIComponent(id));
      if (state.openId !== id) return;
      const body = $('#drawerBody');
      const top = body.scrollTop;
      const typed = $('#reportNotes') ? $('#reportNotes').value : null;
      state.detail = d;
      renderDrawer(d);
      if (typed && $('#reportNotes')) $('#reportNotes').value = typed;
      if (keepScroll) body.scrollTop = top;
    } catch (e) {
      if (state.openId === id && !state.detail) $('#drawerBody').innerHTML = `<p class="loading">${esc(e.message)}</p>`;
    }
  }

  function openLead(id, opener) {
    stopBurst();
    state.openId = id;
    state.detail = null;
    state.opener = opener || document.activeElement;
    $('#drawerTitle').textContent = 'Lead';
    $('#drawerBody').innerHTML = '<p class="loading">Loading lead...</p>';
    $('#scrim').hidden = false;
    $('#drawer').hidden = false;
    document.body.style.overflow = 'hidden';
    $('#closeDrawer').focus();
    loadDetail(id, false);
  }

  function closeLead() {
    stopBurst();
    state.openId = null;
    state.detail = null;
    $('#drawer').hidden = true;
    $('#scrim').hidden = true;
    document.body.style.overflow = '';
    if (state.opener && document.contains(state.opener)) state.opener.focus();
  }

  // After the visitor does something, check back quickly while the automation works.
  function burst(id, ticks = 14, every = 2500) {
    stopBurst();
    let n = ticks;
    state.burstTimer = setInterval(() => {
      n -= 1;
      loadOverview();
      if (state.openId === id) loadDetail(id, true);
      if (n <= 0 || state.openId !== id) stopBurst();
    }, every);
  }
  function stopBurst() { if (state.burstTimer) clearInterval(state.burstTimer); state.burstTimer = null; }

  // ---------- actions ----------
  async function withBusy(button, task) {
    if (state.busy) return;
    state.busy = true;
    const label = button ? button.textContent : '';
    if (button) { button.disabled = true; button.textContent = 'Working...'; }
    try { await task(); } catch (e) { toast(e.message, true); }
    finally {
      state.busy = false;
      if (button && document.contains(button)) { button.disabled = false; button.textContent = label; }
    }
  }

  async function approve(estimateId, button, opportunityId) {
    await withBusy(button, async () => {
      const r = await api('/api/decide', { estimate_id: estimateId, decision: 'approve' });
      toast(r.result && r.result.proposal_email_queued ? 'Approved. The proposal is on its way to the customer.' : 'Approved. This customer has no email address, so nothing was sent.');
      await loadOverview();
      const id = opportunityId || state.openId;
      if (state.openId) { await loadDetail(state.openId, true); burst(state.openId, 8, 2500); }
      else if (id) { /* approved from the side list: refresh only */ }
    });
  }

  async function onDrawerClick(ev) {
    const btn = ev.target.closest('[data-action]');
    if (!btn) return;
    const id = state.openId;
    const action = btn.dataset.action;

    if (action === 'reject-toggle') { const box = $('#rejectBox'); box.hidden = !box.hidden; if (!box.hidden) $('#rejectNote').focus(); return; }

    if (action === 'approve') { await approve(btn.dataset.estimate, btn, id); return; }

    if (action === 'reject') {
      await withBusy(btn, async () => {
        await api('/api/decide', { estimate_id: btn.dataset.estimate, decision: 'reject', note: $('#rejectNote').value });
        toast('Rejected. The lead went back to the inspection stage for rework.');
        await loadOverview();
        await loadDetail(id, true);
      });
      return;
    }

    if (action === 'send-report') {
      await withBusy(btn, async () => {
        const r = await api('/api/inspection-report', { opportunity_id: id, notes: $('#reportNotes').value });
        toast(r.status === 'ok' ? 'Estimate drafted by the AI. It is waiting for your approval.' : 'The AI is still working. This page will update when it finishes.');
        await loadOverview();
        await loadDetail(id, true);
        if (r.status !== 'ok') burst(id, 12, 3000);
      });
      return;
    }

    if (action === 'reply') {
      await withBusy(btn, async () => {
        const r = await api('/api/reply', { opportunity_id: id, intent: $('#replyIntent').value, message: $('#replyText').value });
        const n = r.result ? r.result.follow_ups_cancelled : 0;
        toast(n ? 'Reply logged. ' + n + ' follow-ups were cancelled.' : 'Reply logged.');
        await loadOverview();
        await loadDetail(id, true);
      });
    }
  }

  // ---------- test lead dialog ----------
  const dialog = $('#testDialog');
  function openTestDialog() {
    $('#testError').hidden = true;
    dialog.showModal();
  }
  async function submitTest(ev) {
    ev.preventDefault();
    const form = ev.target;
    const btn = $('#submitTest');
    const err = $('#testError');
    err.hidden = true;
    btn.disabled = true;
    btn.textContent = 'Sending...';
    try {
      const r = await api('/api/test-lead', {
        scenario: form.scenario.value,
        name: form.name.value,
        email: form.email.value,
      });
      dialog.close();
      form.reset();
      if (r.status === 'ok') {
        toast('Lead received. Watch it move through the system.');
        await loadOverview();
        openLead(r.opportunity_id, $('#openTest'));
        burst(r.opportunity_id, 10, 2500);
      } else {
        toast('The lead is still being processed. It will appear in the pipeline shortly.');
        setTimeout(loadOverview, 6000);
      }
    } catch (e) {
      err.textContent = e.message;
      err.hidden = false;
    } finally {
      btn.disabled = false;
      btn.textContent = 'Send test lead';
    }
  }

  // ---------- wiring ----------
  document.addEventListener('click', (ev) => {
    const approveBtn = ev.target.closest('[data-approve]');
    if (approveBtn) { approve(approveBtn.dataset.approve, approveBtn, null); return; }
    const openBtn = ev.target.closest('[data-open]');
    if (openBtn && !ev.target.closest('#drawer')) openLead(openBtn.dataset.open, openBtn);
  });
  $('#drawerBody').addEventListener('click', onDrawerClick);
  $('#closeDrawer').addEventListener('click', closeLead);
  $('#scrim').addEventListener('click', closeLead);
  $('#openTest').addEventListener('click', openTestDialog);
  $('#cancelTest').addEventListener('click', () => dialog.close());
  $('#testForm').addEventListener('submit', submitTest);

  document.addEventListener('keydown', (ev) => {
    if (ev.key === 'Escape' && !$('#drawer').hidden && !dialog.open) { closeLead(); return; }
    if (ev.key === 'Tab' && !$('#drawer').hidden && !dialog.open) {
      const nodes = Array.from($('#drawer').querySelectorAll('button, input, select, textarea, summary, [href]')).filter((n) => !n.disabled && n.offsetParent !== null);
      if (!nodes.length) return;
      const first = nodes[0], last = nodes[nodes.length - 1];
      if (ev.shiftKey && document.activeElement === first) { ev.preventDefault(); last.focus(); }
      else if (!ev.shiftKey && document.activeElement === last) { ev.preventDefault(); first.focus(); }
    }
  });

  setInterval(() => { if (!document.hidden) loadOverview(); }, 20000);
  document.addEventListener('visibilitychange', () => { if (!document.hidden) loadOverview(); });
  loadOverview();
})();
````
