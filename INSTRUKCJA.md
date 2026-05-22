# Weather Station — Frontend (instrukcja budowy)

## Cel
Statyczna strona HTML z wykresami danych z ESP32-S3.
Dane pobierane z Supabase REST API. Deploy na Vercel.

---

## Stack
- Czysty HTML + CSS + vanilla JS (zero bundlerów, zero frameworków)
- Chart.js 4.4.1 (CDN)
- Supabase REST API (fetch, bez SDK)

---

## Struktura projektu

```
weather-frontend/
└── index.html          ← jeden plik, wszystko w środku
```

Jeden plik. Żadnych node_modules. Żadnego build stepu.

---

## Supabase — dane do uzupełnienia

```js
const SUPABASE_URL      = 'https://izudnxavhfuuxdnyckdh.supabase.co';
const SUPABASE_ANON_KEY = 'placeholder';   // publishable key
const TABLE             = 'readings';
```

### Schemat tabeli `readings`
| kolumna        | typ           | opis                        |
|----------------|---------------|-----------------------------|
| id             | BIGSERIAL PK  | auto                        |
| recorded_at    | TIMESTAMPTZ   | DEFAULT now()               |
| temp_c         | NUMERIC(5,1)  | temperatura BMP280          |
| pressure_hpa   | NUMERIC(7,1)  | ciśnienie BMP280            |
| humidity_pct   | NUMERIC(5,1)  | wilgotność DHT11            |

### Endpoint (GET)
```
GET {SUPABASE_URL}/rest/v1/readings
  ?recorded_at=gte.{ISO_DATE}
  &order=recorded_at.asc
  &select=recorded_at,temp_c,pressure_hpa,humidity_pct

Headers:
  apikey: {SUPABASE_ANON_KEY}
  Authorization: Bearer {SUPABASE_ANON_KEY}
```
Zwraca tablicę JSON-ów. Odpowiedź 200 = OK.

---

## Wygląd i UX

### Motyw
- Ciemny (dark-only), tło `#0f1117`, surface `#1a1d27`
- Akcenty: amber `#f59e0b` (temp), blue `#4f8ef7` (wilgotność), teal `#2dd4bf` (ciśnienie)
- Font: `system-ui`

### Layout (góra → dół)
1. **Header** — tytuł "Stacja Pogodowa ESP32-S3" + wskaźnik statusu (dot + tekst)
2. **Range bar** — przyciski: `24h | 48h | 7 dni | 30 dni`
3. **Cards** — 4 karty w gridzie:
   - Temperatura (amber, duża cyfra, °C)
   - Wilgotność (blue, %)
   - Ciśnienie (teal, hPa)
   - Ostatni sync (czas + licznik odczytów)
4. **Wykresy** — 3 osobne Chart.js line charts:
   - Temperatura
   - Wilgotność
   - Ciśnienie

### Status dot (lewy górny róg headera)
- `load` → amber, pulsuje (animacja)
- `ok`   → teal, stały
- `err`  → red, stały

### Karty — subtitle
Pod główną wartością: `min X · max Y jednostka` — wyliczane z aktualnego zakresu danych.

### Wykresy — Chart.js config
```js
type: 'line'
borderWidth: 2
pointRadius: 0          // bez kropek na każdym punkcie
pointHoverRadius: 4     // ale hover działa
fill: true              // wypełnienie pod linią, kolor + '18' (alpha ~10%)
tension: 0.3            // lekkie wygładzenie
animation: false        // bez animacji przy update
interaction: { mode: 'index', intersect: false }  // tooltip na całą kolumnę
maxTicksLimit: { x: 8, y: 5 }
```

### Oś X — formatowanie czasu
- Zakres ≤ 48h → `HH:MM`
- Zakres > 48h → `DD.MM HH:MM`
- Locale: `pl-PL`

### Auto-refresh
Co 5 minut (`setInterval`) → `loadData(activeHours)`.

---

## Logika JS

```
init:
  makeChart() × 3
  loadData(24)
  setInterval(loadData, 5min)

loadData(hours):
  since = now - hours * 3600s
  fetch Supabase → rows[]
  if rows.length === 0 → setStatus('err') + show #empty
  else → renderData(rows, hours)

renderData(rows, hours):
  labels = rows.map(formatTime)
  update 3 charts
  update 4 cards (last row = latest values)
  update min/max subtitles

range button click:
  activeHours = btn.dataset.h
  clearInterval, loadData, setInterval
```

---

## Responsywność
- Cards: `grid-template-columns: repeat(auto-fit, minmax(160px, 1fr))`
- Wykresy: `height: 180px` mobile, `200px` na ≥640px (`@media`)
- Header: `flex-wrap: wrap`

---

## Deploy (Vercel)
1. Repozytorium GitHub z jednym plikiem `index.html`
2. Vercel → New Project → Import repo → Deploy
3. Zero konfiguracji — Vercel wykrywa statyczną stronę automatycznie
4. Gotowy URL: `https://nazwa.vercel.app`

---

## Co NIE jest potrzebne
- Node.js / npm
- package.json
- Żadne env variables po stronie Vercel (klucze są w JS — publishable key jest publiczny by design)
- Żaden backend — Supabase REST API odpytywane bezpośrednio z przeglądarki
