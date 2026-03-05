// ==UserScript==
// @name         Tornium Withdraw UI (OAuth PKCE) - Roxie
// @namespace    roxie-tornium-withdraw
// @version      1.0.0
// @description  Create Tornium faction vault requests (cash/points) from Torn with OAuth PKCE. Icon + Armoury button + Toasts.
// @author       Roxie + ChatGPT
// @match        https://www.torn.com/*
// @match        https://torn.com/*
// @grant        GM_getValue
// @grant        GM_setValue
// @grant        GM_deleteValue
// @grant        GM_xmlhttpRequest
// @grant        GM_registerMenuCommand
// @connect      tornium.com
// @connect      www.tornium.com
// @run-at       document-idle
// ==/UserScript==

(() => {
  'use strict';

  // ============================================================
  // ✅ CONFIG YOU CARE ABOUT
  // ============================================================
  const CLIENT_ID = 'dcbcaf7727875f7295b5570c9699886f6bb03ce2d42182c5';
  const REDIRECT_URI = 'https://www.torn.com/index'; // must match Tornium client exactly
  const SCOPES = 'identity faction faction:banking torn_key:usage';

  // Icon styling
  const ICON_COLOR = '#035104';       // pine green
  const ICON_OUTLINE = '#c0c0c0';     // subtle silver
  const ICON_GLOW = 'rgba(192,192,192,0.35)';

  // Tornium endpoints (these are what worked for you)
  const TORNIUM = {
    AUTH: 'https://tornium.com/oauth/authorize',
    TOKEN: 'https://tornium.com/oauth/token',
    API_BASE: 'https://tornium.com/api/v1',
    VAULT_GET: '/faction/banking/vault',    // GET
    REQUEST_POST: '/faction/banking',       // POST (creates vault request)
  };

  // Where to inject on Torn UI
  const SELECTORS = {
    statusIconsUl: 'ul[class*="status-icons"]',
  };

  // Button injection page (your link)
  const ARMOURY_MATCH = /factions\.php\?step=your/i;

  // Timeout dropdown
  const TIMEOUT_OPTIONS = [
    { label: 'None', minutes: 0 },
    { label: '15 minutes', minutes: 15 },
    { label: '30 minutes', minutes: 30 },
    { label: '1 hour (default)', minutes: 60 },
    { label: '2 hours', minutes: 120 },
    { label: '4 hours', minutes: 240 },
    { label: '8 hours', minutes: 480 },
  ];

  // ============================================================
  // GM STORAGE (PDA/TM-friendly)
  // ============================================================
  const storeKey = (k) => `rx_tornium_${k}`;

  const safeGM = {
    get: async (k, def = null) => {
      try {
        if (typeof GM_getValue === 'function') return await GM_getValue(storeKey(k), def);
      } catch {}
      try {
        const raw = localStorage.getItem(storeKey(k));
        return raw == null ? def : JSON.parse(raw);
      } catch {
        return def;
      }
    },
    set: async (k, v) => {
      try {
        if (typeof GM_setValue === 'function') return await GM_setValue(storeKey(k), v);
      } catch {}
      try {
        localStorage.setItem(storeKey(k), JSON.stringify(v));
      } catch {}
    },
    del: async (k) => {
      try {
        if (typeof GM_deleteValue === 'function') return await GM_deleteValue(storeKey(k));
      } catch {}
      try {
        localStorage.removeItem(storeKey(k));
      } catch {}
    },
    menu: (name, fn) => {
      try {
        if (typeof GM_registerMenuCommand === 'function') GM_registerMenuCommand(name, fn);
      } catch {}
    }
  };

  // ============================================================
  // TOASTS (instead of alerts)
  // ============================================================
  function ensureToastStyles() {
    if (document.getElementById('rx-toast-style')) return;
    const s = document.createElement('style');
    s.id = 'rx-toast-style';
    s.textContent = `
      #rx-toast-wrap{
        position:fixed; z-index:999999;
        right:16px; top:16px;
        display:flex; flex-direction:column; gap:10px;
        pointer-events:none;
      }
      .rx-toast{
        pointer-events:auto;
        min-width:260px; max-width:360px;
        padding:10px 12px;
        border-radius:10px;
        border:1px solid rgba(255,255,255,0.12);
        background:rgba(10,10,10,0.92);
        color:#eee; font: 12px/1.35 Arial, sans-serif;
        box-shadow: 0 10px 24px rgba(0,0,0,0.45);
        backdrop-filter: blur(6px);
      }
      .rx-toast b{ font-size:13px; display:block; margin-bottom:4px; }
      .rx-toast .sub{ color:#bbb; white-space:pre-wrap; }
      .rx-toast .x{
        float:right; margin-left:10px;
        background:none; border:none; color:#aaa; cursor:pointer;
        font-size:14px;
      }
      .rx-toast.ok{ border-color: rgba(80,200,120,0.35); }
      .rx-toast.warn{ border-color: rgba(255,200,80,0.35); }
      .rx-toast.err{ border-color: rgba(255,90,90,0.40); }
    `;
    document.head.appendChild(s);
  }

  function toast(title, msg = '', type = 'ok', ms = 4200) {
    ensureToastStyles();
    let wrap = document.getElementById('rx-toast-wrap');
    if (!wrap) {
      wrap = document.createElement('div');
      wrap.id = 'rx-toast-wrap';
      document.body.appendChild(wrap);
    }

    const el = document.createElement('div');
    el.className = `rx-toast ${type}`;
    el.innerHTML = `
      <button class="x" title="Close">✕</button>
      <b>${escapeHtml(title)}</b>
      <div class="sub">${escapeHtml(msg)}</div>
    `;
    el.querySelector('.x').onclick = () => el.remove();
    wrap.appendChild(el);

    if (ms > 0) setTimeout(() => el.remove(), ms);
  }

  function escapeHtml(str) {
    return String(str ?? '')
      .replaceAll('&','&amp;')
      .replaceAll('<','&lt;')
      .replaceAll('>','&gt;')
      .replaceAll('"','&quot;')
      .replaceAll("'","&#039;");
  }

  // ============================================================
  // PKCE helpers
  // ============================================================
  function b64url(bytes) {
    let str = '';
    bytes.forEach(b => (str += String.fromCharCode(b)));
    return btoa(str).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/g, '');
  }

  async function sha256ToB64Url(input) {
    const enc = new TextEncoder().encode(input);
    const digest = await crypto.subtle.digest('SHA-256', enc);
    return b64url(new Uint8Array(digest));
  }

  function randomString(len = 64) {
    const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~';
    const arr = new Uint8Array(len);
    crypto.getRandomValues(arr);
    return Array.from(arr, x => chars[x % chars.length]).join('');
  }

  function formEncode(obj) {
    return Object.entries(obj)
      .map(([k, v]) => `${encodeURIComponent(k)}=${encodeURIComponent(v)}`)
      .join('&');
  }

  function gmPost(url, bodyObj, headers = {}) {
    return new Promise((resolve, reject) => {
      if (typeof GM_xmlhttpRequest !== 'function') {
        reject(new Error('GM_xmlhttpRequest not available (Tampermonkey grant missing?)'));
        return;
      }
      GM_xmlhttpRequest({
        method: 'POST',
        url,
        headers: { 'Content-Type': 'application/x-www-form-urlencoded', ...headers },
        data: formEncode(bodyObj),
        timeout: 25000,
        onload: resolve,
        onerror: reject
      });
    });
  }

  function gmJson(method, url, token, jsonBody = null) {
    return new Promise((resolve, reject) => {
      if (typeof GM_xmlhttpRequest !== 'function') {
        reject(new Error('GM_xmlhttpRequest not available (Tampermonkey grant missing?)'));
        return;
      }
      GM_xmlhttpRequest({
        method,
        url,
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`,
        },
        data: jsonBody ? JSON.stringify(jsonBody) : null,
        timeout: 25000,
        onload: resolve,
        onerror: reject,
      });
    });
  }

  // ============================================================
  // Amount parsing
  // ============================================================
  function parseTornAmount(input) {
    if (!input) return { ok: false, reason: 'Empty amount' };
    const raw = String(input).trim().toLowerCase().replace(/,/g, '');
    if (raw === 'all') return { ok: true, value: 'all' };

    // allow "$1" or "1"
    const cleaned = raw.replace(/^\$/,'');
    const m = cleaned.match(/^(\d+(\.\d+)?)([kmb])?$/i);
    if (!m) return { ok: false, reason: 'Invalid format (try 1k, 1m, 1b, 500000, all)' };

    const num = Number(m[1]);
    if (!Number.isFinite(num) || num <= 0) return { ok: false, reason: 'Amount must be > 0' };

    const suffix = (m[3] || '').toLowerCase();
    const mult = suffix === 'k' ? 1e3 : suffix === 'm' ? 1e6 : suffix === 'b' ? 1e9 : 1;

    const value = Math.floor(num * mult);
    if (value <= 0) return { ok: false, reason: 'Amount too small' };
    return { ok: true, value };
  }

  function fmt(n) {
    try { return Number(n).toLocaleString(); } catch { return String(n); }
  }

  // ============================================================
  // ICON (status bar) — Pine Green + Silver outline/glow
  // ============================================================
  function iconDataUri() {
    // Simple moneybag + coin SVG (styleable)
    const svg = `
      <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 256 256">
        <defs>
          <filter id="g" x="-40%" y="-40%" width="180%" height="180%">
            <feDropShadow dx="0" dy="0" stdDeviation="3" flood-color="${ICON_GLOW}"/>
          </filter>
        </defs>
        <g filter="url(#g)">
          <path d="M95 22c8 10 18 16 33 16s25-6 33-16c11 8 18 22 10 34-7 10-18 16-43 16S92 66 85 56c-8-12-1-26 10-34z"
            fill="${ICON_COLOR}" stroke="${ICON_OUTLINE}" stroke-width="6" stroke-linejoin="round"/>
          <path d="M54 96c-17 22-26 46-26 78 0 54 42 78 100 78s100-24 100-78c0-32-9-56-26-78-17-22-39-32-74-32S71 74 54 96z"
            fill="${ICON_COLOR}" stroke="${ICON_OUTLINE}" stroke-width="6" stroke-linejoin="round"/>
          <circle cx="190" cy="170" r="46" fill="${ICON_COLOR}" stroke="${ICON_OUTLINE}" stroke-width="6"/>
          <path d="M121 118h10c10 0 18 7 18 16 0 9-8 16-18 16h-10c-10 0-18 7-18 16s8 16 18 16h10"
            fill="none" stroke="#0a0a0a" stroke-width="10" stroke-linecap="round"/>
          <path d="M126 110v68" stroke="#0a0a0a" stroke-width="10" stroke-linecap="round"/>
          <path d="M186 150h8c8 0 14 5 14 12 0 7-6 12-14 12h-8c-8 0-14 5-14 12s6 12 14 12h8"
            fill="none" stroke="#0a0a0a" stroke-width="10" stroke-linecap="round"/>
          <path d="M190 144v54" stroke="#0a0a0a" stroke-width="10" stroke-linecap="round"/>
        </g>
      </svg>
    `.trim();

    return `data:image/svg+xml;charset=utf-8,${encodeURIComponent(svg)}`;
  }

  function ensureIconStyles() {
    if (document.getElementById('rx-withdraw-icon-style')) return;
    const s = document.createElement('style');
    s.id = 'rx-withdraw-icon-style';
    s.textContent = `
      #rx-tornium-withdraw-icon img{
        width: 17px; height: 17px; display:block;
      }
      #rx-tornium-withdraw-icon{
        background: none !important;
      }
      #rx-tornium-withdraw-icon a{ cursor:pointer; }
    `;
    document.head.appendChild(s);
  }

  function upsertStatusIcon() {
    ensureIconStyles();
    const ul = document.querySelector(SELECTORS.statusIconsUl);
    if (!ul) return;

    let li = document.getElementById('rx-tornium-withdraw-icon');
    if (!li) {
      li = document.createElement('li');
      li.id = 'rx-tornium-withdraw-icon';

      const a = document.createElement('a');
      a.href = 'javascript:void(0)';
      a.setAttribute('aria-label', 'Tornium Withdraw');
      a.tabIndex = 0;

      const img = document.createElement('img');
      img.src = iconDataUri();
      img.alt = 'Withdraw';

      a.appendChild(img);
      li.appendChild(a);
      ul.appendChild(li);

      a.addEventListener('click', (e) => {
        e.preventDefault();
        openWithdrawModal({ source: 'icon' });
      });
    }
  }

  // ============================================================
  // MODAL UI
  // ============================================================
  function ensureModalStyles() {
    if (document.getElementById('rx-withdraw-style')) return;
    const s = document.createElement('style');
    s.id = 'rx-withdraw-style';
    s.textContent = `
      #rx-withdraw-overlay{position:fixed;inset:0;background:rgba(0,0,0,.55);z-index:999998;display:flex;align-items:center;justify-content:center}
      #rx-withdraw-modal{width:min(760px,92vw);background:#111;border:1px solid rgba(255,255,255,.10);border-radius:14px;padding:14px 14px 12px;color:#eee;font-family:Arial}
      #rx-withdraw-modal h3{margin:0 0 10px;font-size:16px}
      #rx-withdraw-toprow{display:flex;justify-content:space-between;align-items:center;gap:10px;margin-bottom:10px}
      #rx-withdraw-badge{font-size:12px;color:#cfd3d8;background:rgba(255,255,255,0.06);border:1px solid rgba(255,255,255,0.10);padding:6px 10px;border-radius:999px}
      #rx-withdraw-vault{font-size:12px;color:#cfd3d8;white-space:pre-wrap;border:1px dashed rgba(255,255,255,.12);border-radius:10px;padding:10px;margin:8px 0}
      #rx-withdraw-grid{display:grid;grid-template-columns: 1.3fr 1fr 1fr;gap:10px}
      #rx-withdraw-modal label{display:block;margin:10px 0 4px;font-size:12px;color:#bbb}
      #rx-withdraw-modal input,#rx-withdraw-modal select{width:100%;padding:10px;border-radius:10px;border:1px solid rgba(255,255,255,.12);background:#0b0b0b;color:#eee}
      #rx-withdraw-actions{display:flex;gap:10px;justify-content:flex-end;margin-top:12px;flex-wrap:wrap}
      .rx-btn{padding:9px 12px;border-radius:10px;border:1px solid rgba(255,255,255,.14);background:#1a1a1a;color:#eee;cursor:pointer}
      .rx-btn.primary{background:#2b7cff;border-color:#2b7cff}
      .rx-btn.danger{background:#6b1b1b;border-color:#6b1b1b}
      .rx-btn.ghost{background:#141414}
      #rx-withdraw-error{margin-top:10px;font-size:12px;color:#ff6b6b;white-space:pre-wrap}
      #rx-withdraw-hint{margin-top:8px;font-size:12px;color:#aaa}
      @media (max-width:700px){
        #rx-withdraw-grid{grid-template-columns: 1fr; }
      }
    `;
    document.head.appendChild(s);
  }

  async function openWithdrawModal({ source = 'unknown' } = {}) {
    ensureModalStyles();

    // Only one modal
    const existing = document.getElementById('rx-withdraw-overlay');
    if (existing) existing.remove();

    const overlay = document.createElement('div');
    overlay.id = 'rx-withdraw-overlay';

    const modal = document.createElement('div');
    modal.id = 'rx-withdraw-modal';

    modal.innerHTML = `
      <div id="rx-withdraw-toprow">
        <h3>Tornium Withdraw</h3>
        <div id="rx-withdraw-badge">Source: ${escapeHtml(source)}</div>
      </div>

      <div id="rx-withdraw-vault">Vault: (loading...)</div>

      <div id="rx-withdraw-grid">
        <div>
          <label>Amount</label>
          <input id="rx-amt" placeholder="e.g. 1m, 250k, 1500000, all" />
        </div>
        <div>
          <label>Type</label>
          <select id="rx-type">
            <option value="cash">Cash</option>
            <option value="points">Points</option>
          </select>
        </div>
        <div>
          <label>Timeout</label>
          <select id="rx-timeout"></select>
        </div>
      </div>

      <div id="rx-withdraw-error"></div>
      <div id="rx-withdraw-hint">
        Tip: If you changed scopes in Tornium, click <b>Force Re-login</b> once to mint a new token.
      </div>

      <div id="rx-withdraw-actions">
        <button class="rx-btn danger" id="rx-relogin">Force Re-login</button>
        <button class="rx-btn ghost" id="rx-refresh">Refresh Vault</button>
        <button class="rx-btn" id="rx-close">Close</button>
        <button class="rx-btn primary" id="rx-submit">Submit Request</button>
      </div>
    `;

    overlay.appendChild(modal);
    document.body.appendChild(overlay);

    // Fill timeouts
    const sel = modal.querySelector('#rx-timeout');
    TIMEOUT_OPTIONS.forEach((o) => {
      const opt = document.createElement('option');
      opt.value = String(o.minutes);
      opt.textContent = o.label;
      if (o.minutes === 60) opt.selected = true;
      sel.appendChild(opt);
    });

    const close = () => overlay.remove();
    overlay.addEventListener('click', (e) => { if (e.target === overlay) close(); });
    modal.querySelector('#rx-close').addEventListener('click', close);

    // Buttons
    modal.querySelector('#rx-relogin').addEventListener('click', async () => {
      await forceRelogin();
    });

    modal.querySelector('#rx-refresh').addEventListener('click', async () => {
      await refreshVaultDisplay(modal);
    });

    modal.querySelector('#rx-submit').addEventListener('click', async () => {
      await submitWithdraw(modal);
    });

    // initial vault load
    await refreshVaultDisplay(modal);
  }

  async function setVaultText(modal, txt) {
    const box = modal.querySelector('#rx-withdraw-vault');
    if (box) box.textContent = txt;
  }
  function setErr(modal, txt) {
    const box = modal.querySelector('#rx-withdraw-error');
    if (box) box.textContent = txt || '';
  }

  // ============================================================
  // OAuth callback handler (exchange code -> token) and clean URL
  // ============================================================
  async function handleOAuthCallbackIfPresent() {
    const params = new URLSearchParams(window.location.search);
    const code = params.get('code');
    const state = params.get('state');

    if (!code) return;

    const expectedState = await safeGM.get('oauth_state', null);
    const verifier = await safeGM.get('pkce_verifier', null);

    if (!expectedState || !verifier) {
      toast('Tornium login failed', 'Missing stored state/verifier. Click Force Re-login again.', 'err', 6500);
      return;
    }
    if (!state || state !== expectedState) {
      toast('Tornium login blocked', 'State mismatch (safety check). Try Force Re-login again.', 'err', 6500);
      return;
    }

    toast('Tornium', 'Code received — exchanging for token…', 'ok', 3500);

    try {
      const res = await gmPost(TORNIUM.TOKEN, {
        grant_type: 'authorization_code',
        client_id: CLIENT_ID,
        code,
        redirect_uri: REDIRECT_URI,
        code_verifier: verifier,
        scope: SCOPES,
      });

      let data = null;
      try { data = JSON.parse(res.responseText); } catch {}

      if (res.status >= 200 && res.status < 300 && data?.access_token) {
        await safeGM.set('access_token', data.access_token);
        await safeGM.set('token_saved_at', Date.now());
        toast('Tornium', 'Token saved ✅', 'ok', 4500);

        // IMPORTANT: remove code/state from URL so it doesn't re-trigger and cause "double login"
        try {
          const cleanUrl = new URL(window.location.href);
          cleanUrl.searchParams.delete('code');
          cleanUrl.searchParams.delete('state');
          cleanUrl.searchParams.delete('scope');
          history.replaceState({}, document.title, cleanUrl.toString());
        } catch {}
      } else {
        toast('Tornium token exchange failed', `Status: ${res.status}\n${res.responseText || '(empty)'}`, 'err', 9000);
      }
    } catch (e) {
      toast('Tornium network error', 'Could not call /oauth/token. Check @connect and TM grants.', 'err', 9000);
    }
  }

  async function forceRelogin() {
    // wipe token so we actually re-auth
    await safeGM.del('access_token');

    const state = randomString(32);
    const verifier = randomString(64);
    const challenge = await sha256ToB64Url(verifier);

    await safeGM.set('oauth_state', state);
    await safeGM.set('pkce_verifier', verifier);

    const authUrl =
      TORNIUM.AUTH +
      '?response_type=code' +
      '&client_id=' + encodeURIComponent(CLIENT_ID) +
      '&redirect_uri=' + encodeURIComponent(REDIRECT_URI) +
      '&scope=' + encodeURIComponent(SCOPES) +
      '&state=' + encodeURIComponent(state) +
      '&code_challenge_method=S256' +
      '&code_challenge=' + encodeURIComponent(challenge);

    toast('Tornium', 'Opening Tornium login…', 'ok', 2500);
    window.location.href = authUrl;
  }

  // ============================================================
  // Tornium API calls
  // ============================================================
  async function getToken() {
    return await safeGM.get('access_token', null);
  }

  async function fetchVault() {
    const token = await getToken();
    if (!token) return { ok: false, status: 0, err: 'No token saved. Click Force Re-login.' };

    const url = TORNIUM.API_BASE + TORNIUM.VAULT_GET;
    const res = await gmJson('GET', url, token);
    let data = null;
    try { data = JSON.parse(res.responseText); } catch {}
    if (res.status >= 200 && res.status < 300 && data) {
      return { ok: true, status: res.status, data };
    }
    return { ok: false, status: res.status, err: res.responseText || 'Unknown error' };
  }

  async function createVaultRequest(payload) {
    const token = await getToken();
    if (!token) return { ok: false, status: 0, err: 'No token saved. Click Force Re-login.' };

    const url = TORNIUM.API_BASE + TORNIUM.REQUEST_POST;
    const res = await gmJson('POST', url, token, payload);
    let data = null;
    try { data = JSON.parse(res.responseText); } catch {}
    if (res.status >= 200 && res.status < 300) {
      return { ok: true, status: res.status, data, raw: res.responseText };
    }
    return { ok: false, status: res.status, err: res.responseText || 'Unknown error' };
  }

  async function refreshVaultDisplay(modal) {
    setErr(modal, '');
    await setVaultText(modal, 'Vault: (loading...)');

    const v = await fetchVault();
    if (!v.ok) {
      await setVaultText(modal, `Vault: Error (${v.status || '??'})`);
      setErr(modal, v.err);
      if (v.status === 401) toast('Tornium', 'Token invalid/expired. Click Force Re-login.', 'warn', 6500);
      if (v.status === 403) toast('Tornium', 'Insufficient scope. Re-login after scopes change.', 'warn', 6500);
      return;
    }

    const d = v.data;
    const money = d.money_balance ?? d.money ?? d.cash ?? 0;
    const points = d.points_balance ?? d.points ?? 0;
    const factionId = d.faction_id ?? '?';
    const playerId = d.player_id ?? '?';

    await setVaultText(
      modal,
      `Vault:\n  faction_id: ${factionId}\n  player_id:  ${playerId}\n  money:      ${fmt(money)}\n  points:     ${fmt(points)}`
    );
  }

  async function submitWithdraw(modal) {
    setErr(modal, '');

    const amtRaw = modal.querySelector('#rx-amt')?.value;
    const type = modal.querySelector('#rx-type')?.value || 'cash';
    const timeout = Number(modal.querySelector('#rx-timeout')?.value || 60);

    const parsed = parseTornAmount(amtRaw);
    if (!parsed.ok) {
      setErr(modal, parsed.reason);
      toast('Fix amount', parsed.reason, 'warn', 4500);
      return;
    }

    // confirm "all"
    if (parsed.value === 'all') {
      const ok = confirm(`Request ALL ${type.toUpperCase()} from faction vault?\n\n(If you didn’t mean it, hit Cancel.)`);
      if (!ok) return;
    }

    const payload = {
      amount_requested: parsed.value,
      type: type,                    // "cash" | "points"
      timeout: timeout,       // number
    };

    toast('Submitting request…', `${type} • ${parsed.value === 'all' ? 'ALL' : fmt(parsed.value)} • timeout ${timeout}m`, 'ok', 3500);

    const res = await createVaultRequest(payload);
    if (!res.ok) {
      setErr(modal, res.err);
      toast('Request failed', `Status: ${res.status}\n${res.err}`, 'err', 9000);
      return;
    }

    const info = res.data || {};
    const id = info.id ?? '(no id)';
    const amount = info.amount ?? payload.amount_requested;
    toast('Request created ✅', `id: ${id}\namount: ${amount}`, 'ok', 6500);

    // Refresh vault so UI updates after request
    await refreshVaultDisplay(modal);
  }

  // ============================================================
  // Armoury page injection (near deposit money/points)
  // ============================================================
  function onArmouryPage() {
    if (!ARMOURY_MATCH.test(location.href)) return false;
    // hash might be different depending on TornTools, but we prefer armoury tab
    return (location.href.includes('tab=armoury') || location.hash.includes('armoury'));
  }

  function injectArmouryButton() {
    // We’ll locate a nearby anchor point by finding buttons containing "DEPOSIT MONEY" or "DEPOSIT POINTS"
    const depositMoneyBtn = [...document.querySelectorAll('button, input[type="button"], a')]
      .find(el => (el.textContent || el.value || '').trim().toUpperCase() === 'DEPOSIT MONEY');

    const depositPointsBtn = [...document.querySelectorAll('button, input[type="button"], a')]
      .find(el => (el.textContent || el.value || '').trim().toUpperCase() === 'DEPOSIT POINTS');

    const anchor = depositMoneyBtn || depositPointsBtn;
    if (!anchor) return;

    // choose container
    const container = anchor.closest('div') || anchor.parentElement;
    if (!container) return;

    if (document.getElementById('rx-armoury-withdraw-btn')) return;

    const btn = document.createElement('button');
    btn.id = 'rx-armoury-withdraw-btn';
    btn.type = 'button';
    btn.textContent = 'Withdraw (Tornium)';
    btn.style.cssText = `
      margin: 10px 0;
      padding: 8px 12px;
      border-radius: 10px;
      border: 1px solid rgba(255,255,255,0.18);
      background: rgba(10,10,10,0.85);
      color: #eaeaea;
      cursor: pointer;
      font: 12px/1 Arial, sans-serif;
    `;

    btn.addEventListener('click', () => openWithdrawModal({ source: 'armoury' }));

    // Put it either above the deposit money or below — we’ll do ABOVE if money exists, else above points
    container.insertBefore(btn, anchor);

    toast('Tornium Withdraw', 'Armoury button injected ✅', 'ok', 2800);
  }

  // ============================================================
  // Init + observers
  // ============================================================
  function bootObservers() {
    // Status icon: keep trying (SPA)
    const iconTick = () => {
      try { upsertStatusIcon(); } catch {}
    };
    iconTick();
    setInterval(iconTick, 2500);

    // Armoury injection: mutation observer + periodic check
    const tryArm = () => {
      if (!onArmouryPage()) return;
      try { injectArmouryButton(); } catch {}
    };
    tryArm();

    const mo = new MutationObserver(() => {
      iconTick();
      tryArm();
    });
    mo.observe(document.documentElement, { childList: true, subtree: true });

    // hash changes (SPA-ish)
    window.addEventListener('hashchange', () => setTimeout(tryArm, 350));
    window.addEventListener('popstate', () => setTimeout(tryArm, 350));
  }

  // ============================================================
  // Menu Commands
  // ============================================================
  safeGM.menu('Tornium Withdraw: Open UI', () => openWithdrawModal({ source: 'menu' }));
  safeGM.menu('Tornium Withdraw: Force Re-login', () => forceRelogin());
  safeGM.menu('Tornium Withdraw: Clear Token', async () => {
    await safeGM.del('access_token');
    toast('Tornium', 'Token cleared.', 'warn', 4500);
  });

  // ============================================================
  // MAIN
  // ============================================================
  (async function main() {
    await handleOAuthCallbackIfPresent();

    // Don’t auto-login. We only login when YOU click Force Re-login.
    const token = await getToken();
    if (!token) {
      toast('Tornium not connected', 'Open the UI (icon) → Force Re-login to connect.', 'warn', 6500);
    }

    bootObservers();
  })();

})();
