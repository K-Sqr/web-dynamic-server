# Drug Use Analytics -- Codebase Walkthrough

This document explains every part of the project so you can walk an interviewer through it as if you built it.

---

## Table of Contents

1. [What This Project Is](#what-this-project-is)
2. [How It Works (End to End)](#how-it-works-end-to-end)
3. [The Server (server.mjs)](#the-server)
4. [The Template Engine](#the-template-engine)
5. [The Data Pipeline](#the-data-pipeline)
6. [The Views](#the-views)
7. [The Styling](#the-styling)
8. [Design Decisions and Why](#design-decisions-and-why)

---

## What This Project Is

A dynamic web server that reads a CSV file of national drug usage statistics, parses it into structured data, and serves interactive web pages with Chart.js visualizations. Users can explore the data across three dimensions:

1. **By Age** -- "What substances does a 19-year-old use most?" (pie chart)
2. **By Drug Type** -- "Which age groups use marijuana the most?" (pie chart)
3. **By Frequency** -- "How often does each age group use cocaine?" (bar chart)

There's no frontend framework. The server generates full HTML pages on every request (server-side rendering).

---

## How It Works (End to End)

```
Startup:
  1. server.mjs reads data/drug-use-by-age.csv
  2. Custom parseCSV() splits it into rows of objects
  3. init() extracts unique ages, drug types, and maps images
  4. Data stored in a global DATA object

Request:
  1. User visits /age/19
  2. Express route handler finds the row where age === "19"
  3. Extracts drug usage percentages from that row
  4. Calls renderTemplate("age", { data }) to generate HTML
  5. Template builds a full HTML page with Chart.js pie chart
  6. HTML string returned to browser
  7. Browser renders the page and Chart.js draws the chart
```

---

## The Server

### File: `server.mjs`

This is the entire backend -- one file, ~395 lines. Here's how it's organized:

### Imports and Setup (Lines 1-19)

```javascript
import express from "express";
import fs from "fs";
import path from "path";
import url from "url";

const app = express();
const PORT = process.env.PORT || 3000;

app.use("/static", express.static(path.join(__dirname, "static")));
app.use("/static/img", express.static(path.join(__dirname, "img")));
```

- Uses **ESM modules** (`import` syntax, `.mjs` extension) instead of CommonJS (`require`)
- `__dirname` doesn't exist in ESM, so it's manually derived from `import.meta.url` -- this is a common ESM pattern
- **Static file serving:** Express serves the `static/` folder (CSS) and `img/` folder (photos) as static assets so the browser can load them directly

### CSV Parser (Lines 22-31)

```javascript
function parseCSV(text) {
  const lines = text.split(/\r?\n/).filter(l => l.trim() !== "");
  const headers = lines.shift().split(",").map(h => h.trim());
  return lines.map(line => {
    const vals = line.split(",").map(v => v.trim());
    const obj = {};
    for (let i = 0; i < headers.length; i++) obj[headers[i]] = vals[i] ?? "";
    return obj;
  });
}
```

**What it does:** Takes raw CSV text and converts it into an array of objects.

**How it works:**
1. Split text into lines, removing empty lines
2. First line becomes the headers array: `["age", "n", "alcohol_use", "alcohol_frequency", ...]`
3. Each subsequent line becomes an object: `{ age: "19", n: "2223", alcohol_use: "64.6", ... }`

**Why a custom parser instead of a library?** The `csv-parser` package is in `package.json` but isn't used -- the custom parser is simpler for this flat CSV (no quoted fields, no commas within values). It avoids the overhead of streaming and works synchronously since the file is small (18 rows).

### Data Initialization (Lines 34-121)

The `init()` function runs once at startup before the server starts listening.

**Step 1: Parse CSV**
```javascript
const csvPath = path.join(__dirname, "data", "drug-use-by-age.csv");
const txt = fs.readFileSync(csvPath, "utf8");
const rows = parseCSV(txt);
DATA.rows = rows;
```

**Step 2: Extract unique ages and drug types**
```javascript
const ages = Array.from(new Set(rows.map(r => String(r.age))));
// ages = ["12", "13", "14", ..., "65+"]

const types = Object.keys(first).filter(k => k.endsWith("_use")).map(k => k.replace(/_use$/, ""));
// types = ["alcohol", "marijuana", "cocaine", "crack", "heroin", ...]
```

The drug types are derived from the CSV column names. Any column ending in `_use` is treated as a drug type. This means if the CSV gets new columns, the code automatically picks them up.

**Step 3: Build image maps**

The function scans the `img/AgePhotos/` and `img/DrugPhotos/` directories for image files and maps them to data entries.

For ages: looks for files named like `Age19.jpg`, `Age22-23.jpg`, etc.
For drugs: looks for files whose name contains the drug type (e.g., `Marijuana.webp` maps to the `marijuana` type).

This image mapping means adding a new photo just requires dropping it in the right folder with the right name -- no code changes needed.

### Template System (Lines 141-151)

```javascript
const templateCache = new Map();
async function renderTemplate(name, data) {
  if (!templateCache.has(name)) {
    const modPath = path.join(__dirname, "templates", `${name}.mjs`);
    const modUrl = url.pathToFileURL(modPath).href;
    const mod = await import(modUrl);
    templateCache.set(name, mod);
  }
  const mod = templateCache.get(name);
  return mod.render(data);
}
```

**What it does:** Loads a template file (ESM module), caches it, and calls its `render()` function with data.

**How it works:**
1. First call: dynamically imports the template module (e.g., `templates/age.mjs`)
2. Caches the module so subsequent requests don't re-import
3. Calls `mod.render(data)` which returns an HTML string

**This is a custom template engine.** Instead of using EJS, Handlebars, or Pug, each template is a JavaScript file that exports a `render()` function. The function uses template literals (backtick strings) to build HTML. This approach has zero dependencies and gives full programmatic control (loops, conditionals, JSON embedding).

### Routes (Lines 153-371)

There are 7 routes:

**1. `GET /` -- Homepage**
Returns a hardcoded HTML string with three navigation buttons (By Age, By Drug Type, By Frequency). Uses the dark theme CSS.

**2. `GET /age` -- Redirect**
Redirects to `/age/{first_age}` (e.g., `/age/12`). This way the `/age` URL always works.

**3. `GET /age/:age` -- Age Detail Page**
This is the most complex route. It:
- Finds the matching data row (supports exact matches like "19" and range matches like "22-23")
- If the age is out of range (below 12 or above 65+), renders a custom error page
- Extracts usage percentages for every drug type into a `weights` object
- Renders the `age` template with the data, prev/next navigation, and image references

**4. `GET /drug_type` -- Redirect**
Redirects to `/drug_type/{first_type}` (e.g., `/drug_type/alcohol`).

**5. `GET /drug_type/:type` -- Drug Type Detail Page**
- Validates the drug type exists in the data
- Builds a `countsByAge` object: for each age group, what percentage uses this drug
- Renders the `drug_type` template with a pie chart of usage across ages

**6. `GET /drug_frequency` -- Redirect**
Redirects to `/drug_frequency/{first_type}`.

**7. `GET /drug_frequency/:type` -- Frequency Detail Page**
- Similar to drug_type, but pulls `_frequency` columns instead of `_use`
- Renders a bar chart (not pie) since frequency is a continuous value, not a proportion

**Navigation helpers:**
```javascript
function sequentialNeighborsForAge(ageStr) {
  const idx = DATA.agesSorted.indexOf(String(ageStr));
  return {
    prev: idx > 0 ? DATA.agesSorted[idx - 1] : null,
    next: idx < DATA.agesSorted.length - 1 ? DATA.agesSorted[idx + 1] : null
  };
}
```
These functions compute the prev/next entries for navigation buttons. They work by finding the current entry's index in the sorted array and returning the adjacent elements.

### Server Startup (Lines 373-395)

```javascript
await init();
const server = app.listen(PORT, () => console.log(`Server running on http://localhost:${PORT}`));
```

`init()` must complete before the server starts listening -- this ensures the CSV is fully parsed and all data is ready before any requests come in. If `init()` fails (e.g., CSV file missing), the server exits with a helpful error message.

---

## The Template Engine

Each template is in `templates/` and follows the same pattern:

```javascript
// templates/age.mjs
export function render({ title, age, countsByDrug, prev, next, nav }) {
  // Build HTML strings for each section
  const chartCard = `<div class="card">...</div>`;
  const infoCard = `<div class="card">...</div>`;
  
  // Build the Chart.js script
  const inner = `...
    <script>
      new Chart(ctx, {
        type: 'pie',
        data: {
          labels: ${JSON.stringify(labels)},
          datasets: [{ data: ${JSON.stringify(data)} }]
        }
      });
    </script>`;
  
  // Wrap in full HTML document
  const page = `<!doctype html>...<main>{{INNER}}</main>...`
    .replace("{{INNER}}", inner);
  
  return page;
}
```

**Key patterns in the templates:**

1. **Data is serialized into the HTML.** The server converts JavaScript objects into JSON and embeds them directly into `<script>` tags: `labels: ${JSON.stringify(labels)}`. When the browser loads the page, Chart.js reads these inline values.

2. **No AJAX calls.** The chart data doesn't come from an API -- it's baked into the HTML. This means pages work offline after loading (no network dependency for data).

3. **Placeholder images.** If an age group or drug type has no image, the template generates an inline SVG data URL as a placeholder: a gray rectangle with "No image available" text. No broken image icons.

4. **Min/Max statistics.** Each template calculates and displays the minimum and maximum values in the dataset, showing which drug/age has the highest and lowest usage.

### Templates Breakdown

| Template | Chart Type | X-Axis | Y-Axis | Purpose |
|---|---|---|---|---|
| `age.mjs` | Pie | Drug types | Usage % | "What does this age group use?" |
| `drug_type.mjs` | Pie | Age groups | Usage % | "Which ages use this drug?" |
| `drug_frequency.mjs` | Bar | Age groups | Frequency | "How often does each age use this?" |
| `error.mjs` | None | -- | -- | Custom 404 page |
| `base.mjs` | None | -- | -- | Shared layout (not currently used by dark theme pages) |

**Why pie for usage and bar for frequency?**
- Usage data is proportional (percentages that represent parts of a whole) -- pie charts show this well
- Frequency data is a scale (how many times per year) -- bar charts show magnitude comparisons better

---

## The Data Pipeline

This is an **ETL (Extract, Transform, Load)** pipeline:

### Extract
Read the raw CSV file from disk using `fs.readFileSync()`.

### Transform
1. **Parse:** Split CSV into headers + rows, create objects
2. **Derive:** Extract unique ages and drug types from column names
3. **Map images:** Scan filesystem directories and match images to data entries by naming convention
4. **Build navigation:** Create sorted arrays for prev/next navigation

### Load
Store everything in the global `DATA` object so route handlers can access it instantly (no database queries, no file reads on each request).

**Why this approach?**
- The dataset is small (18 rows, ~2KB). Loading it all into memory at startup is efficient.
- No database needed. Adding SQLite would be overkill for 18 rows of static data.
- Data never changes at runtime, so there's no need for cache invalidation.

---

## The Views

### Age View (`/age/19`)
```
┌─────────────────────────────────────────────┐
│ Header: "Age 19"                            │
├───────────┬─────────────────────────────────┤
│ Sidebar   │  Pie Chart: Drug Usage (%)      │
│ Ages:     │  [alcohol: 64.6%, marijuana:    │
│ 12 ←      │   33.4%, cocaine: 4.1%, ...]    │
│ 13        ├─────────────────────────────────┤
│ 14        │  Info Card: Min/Max stats       │
│ ...       ├─────────────────────────────────┤
│ → 19 ←    │  Age Group Image               │
│ ...       ├─────────────────────────────────┤
│ 65+       │  [← Prev] [Next →] [Home]      │
└───────────┴─────────────────────────────────┘
```

### Drug Type View (`/drug_type/marijuana`)
Same layout as age view, but sidebar lists drug types and pie chart shows usage across age groups.

### Frequency View (`/drug_frequency/cocaine`)
Bar chart showing how frequently each age group uses the substance. No sidebar (simpler layout).

---

## The Styling

### File: `static/dark.css`

The dark theme uses **CSS custom properties** (variables) for consistent theming:

```css
:root {
  --bg: #0f1115;
  --panel: #161a22;
  --text: #e6e6ea;
  --accent: #7aa2f7;
  --radius: 14px;
  --shadow: 0 8px 24px rgba(0,0,0,0.35);
}
```

**Visual effects:**
- **Radial gradients on body:** Creates a subtle blue/teal glow behind content
- **Glassmorphism on header:** `backdrop-filter: blur(6px)` with semi-transparent background
- **Card depth:** Linear gradient backgrounds + box shadows make cards feel layered
- **Hover animations:** Buttons lift (`translateY(-2px)`) and shadows deepen on hover
- **Responsive grid:** 2-column on desktop, single column on mobile via `@media(min-width:900px)`

---

## Design Decisions and Why

### Why Express 5 instead of a newer framework (Fastify, Hono)?
Express is the most widely known Node.js framework. For a course project, using Express means anyone reviewing the code (professors, TAs) immediately understands the structure. Express 5 adds native `async` route handler support, which this project uses.

### Why server-side rendering instead of React/Vue?
The project requirements called for a dynamic web server -- the server must generate the HTML. This is the traditional server-rendered web architecture (like PHP, Django, Rails). No client-side framework needed because the interaction model is simple: navigate between pages, view charts.

### Why Chart.js instead of D3?
Chart.js is declarative -- you pass data and options, and it draws the chart. D3 is imperative -- you manually position every element. For standard pie and bar charts, Chart.js is faster to implement and produces clean results with less code. D3 would be better for custom, non-standard visualizations.

### Why a custom template engine instead of EJS/Pug?
EJS and Pug are in the project's dependencies (EJS) but the server uses custom ESM template modules instead. The advantage: template functions are plain JavaScript, so you get full IDE support (autocomplete, type checking), no special syntax to learn, and the ability to use any JavaScript logic directly. The trade-off is that HTML structure is embedded in JS strings, which is harder to read.

### Why no database?
The dataset is 18 rows of static data. Loading it into memory on startup is the right call -- it's faster than querying a database on every request, and the data never changes. A database would add complexity (schema, migrations, connection management) with no benefit for this use case.

### Why the image mapping system?
The `init()` function scans the filesystem to find images rather than hardcoding paths. This means:
- Adding new images requires zero code changes -- just drop the file in the right folder
- Naming conventions (`Age19.jpg`, `Marijuana.webp`) make the mapping self-documenting
- Fallback logic handles missing images gracefully (SVG placeholder)

---

## Resume Bullet Points for This Project

Use these on your resume:

**Drug Usage Analytics Web Server** | Node.js, Express, Chart.js, CSV

- Built a server-rendered data visualization platform that ingests national substance use data and generates interactive Chart.js charts across three exploration dimensions (age, drug type, frequency).
- Implemented a custom ETL pipeline to parse, normalize, and transform CSV data into structured objects at startup, with automatic filesystem-based image mapping.
- Designed a custom ESM template engine for server-side HTML generation, embedding serialized data directly into pages for zero-latency chart rendering.
