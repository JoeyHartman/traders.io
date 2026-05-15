
# Traders.io — Claude Code Project Context

## Wat is dit project?

Traders.io is een **forex trading education platform** waar traders tegen elkaar battelen in real-time prijsvoorspelling challenges. Het concept is een "Duolingo voor technische analyse" — snelle feedback loops, veel herhaling, patroonherkenning door volume.

Het business model is een funnel:
1. **Game** (nu live) — gratis, viral, bouwt email lijst
2. **Community** — Discord/Telegram met beste spelers
3. **Non-KYC CFD DEX Broker** — de uiteindelijke monetisatie (later)

---

## Tech Stack — Volledig Overzicht

### Frontend — traders.io (GitHub Pages)
- **Repo:** `github.com/JoeyHartman/traders.io` (public)
- **Hosting:** GitHub Pages → traders.io (custom domain via Namecheap)
- **Bestanden:**
  - `index.html` — homepage
  - `about.html` — about pagina
  - `contact.html` — contact + formulier
  - `privacy.html` — GDPR privacy policy
  - `terms.html` — terms of use
  - `battle.html` — de game zelf (hoofdproduct)
  - `charting_library/` — TradingView Advanced Charts library (lokaal gehost)
  - `datafeeds/` — TradingView UDF datafeed helper

### Backend — Datafeed Server (Railway)
- **Repo:** `github.com/JoeyHartman/traders-datafeed` (private)
- **URL:** `https://traders-datafeed-production.up.railway.app`
- **Stack:** Python Flask + Gunicorn
- **Bestand:** `traders-datafeed-server.py`
- **Functie:** Implementeert TradingView UDF protocol, haalt data op van Twelve Data API
- **Environment variable:** `TWELVE_DATA_API_KEY = e4131a3de90040c2a8ee85e12d225310`
- **Start command:** `gunicorn traders-datafeed-server:app --bind 0.0.0.0:$PORT`

### Hedge Engine Server (Railway — apart project)
- **URL:** `https://ami-production-11af.up.railway.app`
- **Functie:** Bestaande hedge engine voor Joey's eigen forex trading (los van traders.io)
- **Niet aanraken** — staat los van de game

### Data Provider
- **Twelve Data API** — historische OHLCV forex data
- **API Key:** `e4131a3de90040c2a8ee85e12d225310`
- **Gratis tier:** 800 requests/dag
- **Endpoint:** `https://api.twelvedata.com/time_series?symbol=EUR/USD&interval=5min&outputsize=500&apikey=KEY`

### Chart Library
- **TradingView Advanced Charts** (goedgekeurd door Simon Morda, TradingView)
- **Library path:** `charting_library/` in de traders.io repo (geen leading slash!)
- **Datafeed:** UDF compatible, wijst naar Railway datafeed server
- **Licentie:** Alleen voor traders.io, niet redistributable

---

## Infrastructuur

### Domain & DNS
- **Registrar:** Namecheap
- **DNS type:** BasicDNS
- **A Records:** GitHub Pages IPs (185.199.108-111.153)
- **CNAME:** www → joeyhartman.github.io
- **MX Records:** Namecheap Private Email (joey@traders.io)

### Email
- **Provider:** Namecheap Private Email
- **Adres:** joey@traders.io

---

## De Game — Battle Arena (battle.html)

### Game Flow
```
Lobby (kies pair, chart TF, expiry)
  ↓
FIND MATCH → data laden (Twelve Data via Railway)
  ↓
FASE 1: ANALYSE (15 seconden) — chart zichtbaar, buttons locked
  ↓
FASE 2: PREDICT (10 seconden) — UP/DOWN buttons actief
  ↓
FASE 3: REPLAY — expiry candles één voor één getoond
  ↓
RESULT → WIN/LOSS scherm → REMATCH
```

### Belangrijke variabelen

```javascript
// CONFIG
CONFIG = {
  DATAFEED_URL: 'https://traders-datafeed-production.up.railway.app',
  TWELVE_DATA_KEY: 'e4131a3de90040c2a8ee85e12d225310',
  ANALYSE_SECONDS: 15,
  DECIDE_SECONDS: 10,
  REPLAY_SPEED_MS: 600,
  CANDLES_HISTORY: 100,
  CANDLES_FETCH: 500,
}

// STATE (key fields)
state = {
  symbol: 'EURUSD',        // 'EURUSD', 'GBPUSD', etc
  pair:   'EUR/USD',       // display naam
  tf:     '1',             // TV resolution: '1','5','15','60'
  expiry: 5,               // bars vooruit: 3/5/10/20/35/50
  phase:  'analyse',       // 'analyse'|'vote-open'|'replaying'
  dataCache: {},           // "EURUSD_1" → [{o,h,l,c}]
  cursorIndex: 0,
  visibleCandles: [],      // 100 historische candles
  revealCandles: [],       // expiry candles (verborgen)
  battleCount: 0,
  _replayIdx: 0,
}
```

### TradingView Widget Config (huidige staat)
```javascript
tvWidget = new TradingView.widget({
  container:    'tv-chart-container',
  library_path: 'charting_library/',   // geen leading slash!
  datafeed:     createTVDatafeed(),
  symbol:       symbol,
  interval:     resolution,
  theme:        'dark',
  autosize:     true,
  toolbar_bg:   '#0d1420',
  disabled_features: [
    'header_symbol_search', 'header_compare', 'header_screenshot',
    'header_undo_redo', 'use_localstorage_for_settings',
    'timeframes_toolbar', 'volume_force_overlay',
    'show_logo_on_all_charts', 'display_market_status',
  ],
  enabled_features: [
    'study_templates', 'side_toolbar_in_fullscreen_mode', 'drawing_templates',
  ],
});
```

### Symbol resolution (session fix)
```javascript
// CORRECT format — was '24x5' (broken), nu:
session: '0000-2400:1234567',
timezone: 'Europe/Amsterdam',
```

---

## Bekende Issues (te fixen)

### 1. Datum/tijd zichtbaar op x-as ⚠️ PRIORITEIT
Traders zien echte datums → kunnen data live opzoeken → oneerlijke edge.
- **Oplossing A:** Toevoegen aan `disabled_features`: `'timeframes_toolbar'` (al gedaan), plus `hide_left_toolbar_by_default`
- **Oplossing B:** Tijdstempels vervangen door index-gebaseerde tijd in de datafeed (data komt als `time: index` ipv unix timestamp)
- **Voorkeur:** Oplossing B — meest waterdicht

### 2. Chart te klein — groot zwart vlak eronder ⚠️ PRIORITEIT
`#tv-chart-container` heeft `min-height: 260px` maar vult niet de beschikbare ruimte.
- **Fix:** 
```css
#tv-chart-container {
  flex: 1;
  height: 0;          /* flex hack zodat hij groeit */
  min-height: 300px;
}
.chart-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  min-height: 0;      /* belangrijk voor flex children */
}
```

### 3. Study templates 404 (minor)
`charting_library/undefined/undefined/study_templates` → 404
Niet blokkerend, maar lelijk in console.
- **Fix:** `save_load_adapter` configureren of `study_templates` uit `enabled_features` halen

---

## Datafeed Server — UDF Endpoints

```
GET /config
GET /symbols?symbol=EURUSD
GET /search?query=EUR
GET /history?symbol=EURUSD&resolution=5&from=UNIX&to=UNIX&countback=300
GET /time
```

---

## Ondersteunde Forex Pairs
EURUSD, GBPUSD, USDJPY, AUDUSD, USDCHF, USDCAD, NZDUSD, EURGBP, EURJPY, GBPJPY

---

## Design Systeem

```css
/* Kleuren */
--bg:      #080c12;
--surface: #0d1420;
--surface2:#111d2e;
--border:  #1a2840;
--up:      #00e676;   /* bullish groen */
--down:    #ff3d5a;   /* bearish rood */
--gold:    #ffd740;   /* timer warning */
--accent:  #00b4ff;   /* blauw */
--text:    #e4eeff;
--muted:   #4a6080;

/* Fonts */
--font-display: 'Barlow Condensed', sans-serif;
--font-mono:    'JetBrains Mono', monospace;
```

---

## Volgende Stappen (backlog)

### Nu direct
- [ ] Datum/tijd verbergen op x-as
- [ ] Chart hoogte fixen
- [ ] Joey's UX ideeën voor de battle bespreken en implementeren

### Binnenkort
- [ ] Leaderboard met echte data (nu fake)
- [ ] User accounts / persistente scores
- [ ] Telegram Mini App versie

### Later — DEX Broker
- Non-KYC CFD DEX op Arbitrum One
- Solidity smart contracts: CFDEngine, LPVault, FeeDistributor
- TradingView Trading Platform library voor broker UI
- Kosten: $20-40K MVP

---

## Juridisch
- Privacy Policy: traders.io/privacy.html (GDPR, NL)
- Terms of Use: traders.io/terms.html
- Educational platform, geen financieel advies
- Geen echt geld (virtual coins)
- TradingView licentie: alleen traders.io

---

## Contacten
- **Simon Morda** — TradingView Customer Success, Advanced Charts toegang verleend
- **Álvaro Roo** — TradingView, initieel contact

---

## Over Joey (context voor Claude)
- Forex day trader, 5m/1m charts, chart pattern strategies
- Bestaande hedge engine op Railway (Flask/Python/Make.com/TradingView webhooks)
- Facebook groep: 200K traders (distributiekanaal voor launch)
- Geen technische achtergrond — stap-voor-stap uitleg gewenst
- Werkt vanuit Nederland (Zwolle)
- Voorkeur voor pragmatische, directe aanpak
