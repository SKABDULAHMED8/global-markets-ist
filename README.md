# Global Stock Markets — Live Status in IST ⏱️🇮🇳

A zero-backend, mobile-friendly **single HTML file** that shows **real-time open/close status** of major stock exchanges **converted to Indian Standard Time (IST)**.

- Live clock & **real-time** updates every second  
- **Accurate DST** handling via IANA time zones (Luxon)  
- **Local hours → IST hours** (with lunch breaks where applicable)  
- **“Next day”** label when IST close crosses midnight  
- **Countdown** until open/close + **session length**  
- **Index table** (countries + flags + IST open/close) and **Live table** (status, countdowns)

---

## 🚀 Run It (2 ways)

### 1) Open locally (fastest)
- Download this repo and open `markets-ist.html` in **Chrome** (mobile or desktop). That’s it.

### 2) Host on GitHub Pages
1. Push this repo to GitHub.
2. Go to **Settings → Pages** → set **Source** = `Deploy from a branch`, **Branch** = `main` (root).
3. Open: `https://<your-username>.github.io/<this-repo>/markets-ist.html`

---

## 📁 Project Structure

```
.
├── markets-ist.html   # App (HTML+CSS+JS) — runs entirely in your browser
└── README.md          # This guide
```

---

## 🧠 How It Works (Interview-style Overview)

**Goal:** Show global market sessions **in IST** with precise **DST** handling, **live status**, and **countdowns**, without any backend.

1. **Data model (simple & explicit)**
   - Each exchange has: flag, name, country, `tz` (IANA timezone), and `s` (sessions).  
   - Example: Tokyo with a lunch break:
     ```js
     { f:"🇯🇵", x:"JPX — Tokyo", tz:"Asia/Tokyo", s:[[9,0,11,30],[12,30,15,0]] }
     ```

2. **Time math with Luxon**
   - Sessions are built in the **exchange’s local timezone** → then converted to **IST** for display.
   - This avoids DST pitfalls and **NaN** values common with native `Date` across zones.

3. **Status engine**
   - If now is inside any session → **OPEN** and countdown to close.
   - If not, find the next open **today**, else target **tomorrow**’s first session.
   - Multiple sessions (e.g., lunch) are supported naturally.

4. **Cross‑midnight**
   - When an IST close falls after midnight, we append **“(next day)”**.

5. **Sorting**
   - Tables are sorted by the **first IST opening time** (earliest → latest), mirroring an Indian day.

6. **Performance**
   - No network calls beyond the tiny Luxon script. Re-renders every second.

---

## 📸 Inline Preview (the entire app)

> You can run this **directly** by saving it as `markets-ist.html` and opening in Chrome.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Global Markets — Live in IST</title>
<style>
  :root{--bg:#0b1320;--card:#121a2b;--text:#e2e8f0;--muted:#94a3b8;--good:#10b981;--bad:#ef4444;--accent:#60a5fa;--line:#1f2a44}
  html,body{background:var(--bg);color:var(--text);font-family:Inter,system-ui,Segoe UI,Roboto,Arial,sans-serif}
  .wrap{max-width:1000px;margin:14px auto;padding:10px}
  h1{font-size:18px;margin:0 0 6px}
  .clock{font-variant-numeric:tabular-nums;color:var(--accent);font-weight:600}
  .grid{display:grid;gap:10px}
  .card{background:var(--card);border:1px solid var(--line);border-radius:12px;padding:10px}
  .muted{color:var(--muted);font-size:12px}
  table{width:100%;border-collapse:collapse}
  th,td{border-bottom:1px solid var(--line);padding:8px 6px;vertical-align:top}
  th{color:var(--muted);font-size:12px;text-align:left}
  .flag{font-size:20px}
  .mono{font-variant-numeric:tabular-nums;font-family:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace}
  .status{display:inline-block;padding:2px 8px;border-radius:999px;font-size:11px;font-weight:700}
  .OPEN{background:rgba(16,185,129,.12);color:var(--good)}
  .CLOSED{background:rgba(239,68,68,.12);color:var(--bad)}
  .tag{display:inline-block;background:#0f172a;border:1px solid var(--line);color:var(--muted);padding:1px 6px;border-radius:6px;font-size:11px;margin-right:4px}
  @media (max-width:720px){
    .hide-sm{display:none}
    th:nth-child(6),td:nth-child(6){display:none} /* hide “Len (hrs)” on small screens */
  }
</style>
</head>
<body>
<div class="wrap grid">
  <div class="card" style="display:flex;justify-content:space-between;align-items:center;gap:8px">
    <div>
      <h1>🌏 Global Stock Exchanges — Live Status in IST</h1>
      <div class="muted">Regular sessions only. DST handled automatically. Lunch breaks shown where applicable.</div>
    </div>
    <div class="clock mono" id="clock">--:--:--</div>
  </div>

  <!-- INDEX: countries + times in IST (sorted by earliest open) -->
  <div class="card">
    <div style="display:flex;justify-content:space-between;align-items:center">
      <strong>Index — IST Timeline (Earliest Open → Latest)</strong>
      <span class="muted" id="idxUpd"></span>
    </div>
    <table id="indexTbl" aria-label="Index IST timeline">
      <thead><tr>
        <th>Flag</th><th>Exchange (Country)</th><th>Opens (IST)</th><th>Closes (IST)</th><th class="hide-sm">Len (hrs)</th>
      </tr></thead>
      <tbody id="idxBody"></tbody>
    </table>
  </div>

  <!-- LIVE table -->
  <div class="card">
    <div style="display:flex;justify-content:space-between;align-items:center">
      <strong>Live Status — Local → IST</strong>
      <span class="muted" id="liveUpd"></span>
    </div>
    <table id="liveTbl" aria-label="Live market table">
      <thead><tr>
        <th>Flag</th><th>Exchange</th><th class="hide-sm">Local Hours</th><th>IST Hours</th><th>Status</th><th class="hide-sm">Len (hrs)</th>
      </tr></thead>
      <tbody id="liveBody"></tbody>
    </table>
  </div>
</div>

<!-- Luxon (tiny, DST-safe) -->
<script src="https://cdn.jsdelivr.net/npm/luxon@3/build/global/luxon.min.js"></script>
<script>
(() => {
  const { DateTime, Duration, Interval } = luxon;
  const IST = "Asia/Kolkata";
  // Minimal, clear dataset (regular sessions). Lunch splits for JPX/HKEX/SSE.
  const M = [
    {f:"🇯🇵", x:"JPX — Tokyo", c:"Japan", tz:"Asia/Tokyo",    s:[[9,0,11,30],[12,30,15,0]]},
    {f:"🇰🇷", x:"KRX — Seoul", c:"South Korea", tz:"Asia/Seoul", s:[[9,0,15,30]]},
    {f:"🇦🇺", x:"ASX — Sydney",c:"Australia", tz:"Australia/Sydney", s:[[10,0,16,0]]},
    {f:"🇹🇼", x:"TWSE — Taipei",c:"Taiwan", tz:"Asia/Taipei",   s:[[9,0,13,30]]},
    {f:"🇭🇰", x:"HKEX — Hong Kong", c:"Hong Kong", tz:"Asia/Hong_Kong", s:[[9,30,12,0],[13,0,16,0]]},
    {f:"🇨🇳", x:"SSE — Shanghai",c:"China", tz:"Asia/Shanghai", s:[[9,30,11,30],[13,0,15,0]]},
    {f:"🇮🇳", x:"NSE/BSE — Mumbai",c:"India", tz:IST,           s:[[9,15,15,30]]},
    {f:"🇪🇺", x:"Euronext (Paris/AMS/BRU)", c:"Eurozone", tz:"Europe/Paris", s:[[9,0,17,30]]},
    {f:"🇿🇦", x:"JSE — Johannesburg",c:"South Africa", tz:"Africa/Johannesburg", s:[[9,0,17,0]]},
    {f:"🇬🇧", x:"LSE — London", c:"United Kingdom", tz:"Europe/London", s:[[8,0,16,30]]},
    {f:"🇩🇪", x:"Xetra — Frankfurt", c:"Germany", tz:"Europe/Berlin", s:[[9,0,17,30]]},
    {f:"🇨🇭", x:"SIX — Zurich", c:"Switzerland", tz:"Europe/Zurich", s:[[9,0,17,30]]},
    {f:"🇪🇸", x:"BME — Madrid", c:"Spain", tz:"Europe/Madrid",  s:[[9,0,17,30]]},
    {f:"🇮🇹", x:"Borsa Italiana — Milan", c:"Italy", tz:"Europe/Rome", s:[[9,0,17,30]]},
    {f:"🇨🇦", x:"TSX — Toronto",c:"Canada", tz:"America/Toronto", s:[[9,30,16,0]]},
    {f:"🇺🇸", x:"NYSE — New York",c:"USA", tz:"America/New_York", s:[[9,30,16,0]]},
    {f:"🇺🇸", x:"NASDAQ — New York",c:"USA", tz:"America/New_York", s:[[9,30,16,0]]},
  ];

  // Helpers
  const fmt = (dt, zone) => dt.setZone(zone).toFormat("h:mm a");
  const fmtIST = dt => dt.setZone(IST).toFormat("h:mm a");
  const pad = n => String(n).padStart(2,"0");
  const hhmm = ms => {
    const d = Duration.fromMillis(ms).shiftTo("hours","minutes").toObject();
    const h = Math.max(0, Math.floor(d.hours||0));
    const m = Math.max(0, Math.floor(d.minutes||0));
    return `${pad(h)}:${pad(m)}`;
  };
  const todayLocal = (zone, nowIST) => nowIST.setZone(zone);
  const mk = (y,m,d,hh,mm,zone) => DateTime.fromObject({year:y,month:m,day:d,hour:hh,minute:mm,second:0},{zone});
  const buildSegs = (rec, nowIST) => {
    const t = todayLocal(rec.tz, nowIST);
    return rec.s.map(a=>{
      const o = mk(t.year,t.month,t.day,a[0],a[1],rec.tz);
      let c = mk(t.year,t.month,t.day,a[2],a[3],rec.tz);
      if (c <= o) c = c.plus({days:1}); // safety
      return {o,c};
    });
  };
  const istSpan = (o,c) => {
    const oi = o.setZone(IST), ci = c.setZone(IST);
    const next = (ci.ordinal !== oi.ordinal || ci < oi) ? " (next day)" : "";
    return `${fmtIST(o)} – ${fmtIST(c)}${next}`;
  };
  const totalLen = segs => segs.reduce((ms,s)=>ms + Math.max(0, s.c.diff(s.o).toMillis()),0);

  const statusNow = (segs, nowLocal) => {
    for (const s of segs) {
      if (Interval.fromDateTimes(s.o,s.c).contains(nowLocal)) {
        return {st:"OPEN", until:s.c};
      }
    }
    const fut = segs.map(s=>s.o).filter(t=>t>nowLocal).sort((a,b)=>a-b);
    if (fut.length) return {st:"CLOSED", until:fut[0]};
    // otherwise tomorrow first open
    return {st:"CLOSED", until: segs[0].o.plus({days:1})};
  };

  function renderAll(){
    const nowIST = DateTime.now().setZone(IST);
    const clock = document.getElementById("clock");
    if (clock) clock.textContent = nowIST.toFormat("EEE d LLL yyyy • HH:mm:ss 'IST'");

    // INDEX: sort by first IST open
    const idxBody = document.getElementById("idxBody");
    const idxUpd = document.getElementById("idxUpd");
    const idxRows = M.map(r=>{
      const segs = buildSegs(r, nowIST);
      const first = segs[0].o.setZone(IST), last = segs[segs.length-1].c.setZone(IST);
      return {
        key: first.toMillis(),
        len: totalLen(segs),
        html: `<tr>
          <td class="flag">${r.f}</td>
          <td><strong>${r.x}</strong><br><span class="muted">${r.c}</span></td>
          <td class="mono">${fmtIST(segs[0].o)}</td>
          <td class="mono">${fmtIST(segs[segs.length-1].c)}${(last.ordinal!==first.ordinal||last<first)?' <span class="muted">(next day)</span>':''}</td>
          <td class="mono hide-sm">${(function(ms){const h=Math.floor(ms/3600000),m=Math.floor((ms%3600000)/60000);return (h+'').padStart(2,'0')+':'+(m+'').padStart(2,'0');})(totalLen(segs))}</td>
        </tr>`
      };
    }).sort((a,b)=>a.key-b.key);
    if (idxBody) idxBody.innerHTML = idxRows.map(r=>r.html).join("");
    if (idxUpd) idxUpd.textContent = "Updated " + nowIST.toFormat("HH:mm:ss");

    // LIVE table (also sorted by earliest IST open)
    const liveBody = document.getElementById("liveBody");
    const liveUpd = document.getElementById("liveUpd");
    const liveRows = M.map(r=>{
      const segs = buildSegs(r, nowIST);
      const nowLocal = nowIST.setZone(r.tz);
      const st = statusNow(segs, nowLocal);
      const untilIST = st.until.setZone(IST);
      const left = Math.max(0, st.until.diff(nowLocal).valueOf()); // ms
      const fmtHM = (dt, z) => dt.setZone(z).toFormat("h:mm a");
      const localStr = segs.map(s=>`${fmtHM(s.o,r.tz)} – ${fmtHM(s.c,r.tz)}`).join("  ·  ");
      const istStr   = segs.map(s=>istSpan(s.o,s.c)).join("<br>");
      const hhmm = ms => { const h=Math.floor(ms/3600000), m=Math.floor((ms%3600000)/60000); return (h+'').padStart(2,'0')+':'+(m+'').padStart(2,'0'); };
      return {
        key: segs[0].o.setZone(IST).toMillis(),
        html: `<tr>
          <td class="flag">${r.f}</td>
          <td><strong>${r.x}</strong><br><span class="muted">${r.c} · <span class="mono">${r.tz}</span></span></td>
          <td class="hide-sm mono">${localStr}</td>
          <td class="mono">${istStr}</td>
          <td class="mono">
            <span class="status ${st.st}">${st.st}</span><br>
            ${st.st==="OPEN" ? "closes in" : "opens in"} <strong>${hhmm(left)}</strong><br>
            <span class="muted">at ${untilIST.toFormat("h:mm a")} IST</span>
          </td>
          <td class="mono hide-sm">${(function(ms){const h=Math.floor(ms/3600000),m=Math.floor((ms%3600000)/60000);return (h+'').padStart(2,'0')+':'+(m+'').padStart(2,'0');})(totalLen(segs))}</td>
        </tr>`
      };
    }).sort((a,b)=>a.key-b.key);
    if (liveBody) liveBody.innerHTML = liveRows.map(r=>r.html).join("");
    if (liveUpd) liveUpd.textContent = "Updated " + nowIST.toFormat("HH:mm:ss");
  }

  // First paint + tick every second. Guard to avoid NaN if lib fails.
  if (!luxon?.DateTime) {
    alert("Time library failed to load. Check connection and reload.");
  } else {
    renderAll();
    setInterval(renderAll, 1000);
  }
})();
</script>
</body>
</html>

```

---

## 🛠️ Customize

- **Add an exchange:** Push a new object to `M` with its `tz` and `s` (sessions).  
- **Pre/After-market:** Add as extra segments in `s` and style differently.  
- **Refresh rate:** Change `setInterval(renderAll, 1000)`.

---

## 🧪 Talking Points for Demo/Interview

- **DST correctness** across UK/EU/US vs Indian fixed IST.  
- **Local → IST** conversion strategy prevents drift and NaN.  
- **Edge cases:** lunch breaks, cross‑midnight closures.  
- **No backend** and **no API keys** — safe to embed anywhere.

---

## 📜 License

MIT
