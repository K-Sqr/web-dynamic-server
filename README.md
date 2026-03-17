# Drug Use Analytics -- Dynamic Web Server

A server-rendered data visualization platform that transforms national substance use statistics into interactive, explorable charts. Built with Node.js and Express, the app ingests a CSV dataset and dynamically generates pages with Chart.js visualizations across three dimensions: **age group**, **drug type**, and **usage frequency**.

---

## Features

- **Three Exploration Modes** -- Browse data by age group (pie charts), by drug type across ages (pie charts), or by usage frequency across ages (bar charts)
- **Custom ETL Pipeline** -- Parses raw CSV data at startup, normalizes it into structured objects, and maps image assets to data entries
- **Server-Side Rendering** -- HTML pages generated dynamically using a custom ESM template engine (no client-side framework)
- **Interactive Charts** -- Chart.js pie and bar charts with tooltips, legends, and responsive sizing
- **Sequential Navigation** -- Prev/Next buttons allow browsing through all age groups or drug types in order
- **Sidebar Quick Navigation** -- Clickable sidebar lists for jumping directly to any age or drug type
- **Dark Theme UI** -- Polished dark interface with glassmorphism effects, radial gradients, and hover animations
- **Image Mapping** -- Automatically matches representative images to age groups and drug types from the filesystem
- **Responsive Layout** -- Adapts from desktop (sidebar + chart) to mobile (stacked) with CSS media queries
- **Error Handling** -- Custom 404 pages with contextual navigation for out-of-range or invalid data requests

---

## Tech Stack

| Technology | Purpose |
|---|---|
| [Node.js](https://nodejs.org/) | JavaScript runtime |
| [Express 5](https://expressjs.com/) | Web server and routing |
| [Chart.js](https://www.chartjs.org/) | Client-side data visualization (pie and bar charts) |
| Custom CSV Parser | Parses `drug-use-by-age.csv` into structured data |
| Custom ESM Templates | Server-side HTML generation using ES module `render()` functions |
| CSS (Dark Theme) | Hand-written dark UI with CSS variables, gradients, and glassmorphism |

---

## Data Source

The dataset (`data/drug-use-by-age.csv`) contains national substance use statistics across 17 age groups (ages 12 through 65+) and 13 substance categories:

> Alcohol, Marijuana, Cocaine, Crack, Heroin, Hallucinogen, Inhalant, Pain Reliever, OxyContin, Tranquilizer, Stimulant, Meth, Sedative

Each substance has two metrics: **usage percentage** (`_use`) and **frequency** (`_frequency`).

---

## Getting Started

### Prerequisites

- Node.js 18+

### Installation

```bash
git clone https://github.com/K-Sqr/web-dynamic-server.git
cd web-dynamic-server
npm install
```

### Run the Server

```bash
node server.mjs
```

Open [http://localhost:3000](http://localhost:3000)

---

## Project Structure

```
web-dynamic-server/
├── server.mjs              # Express server, CSV parsing, routing, template rendering
├── data/
│   └── drug-use-by-age.csv # Source dataset (17 age groups x 13 substances)
├── templates/
│   ├── base.mjs            # Base HTML layout (shared structure)
│   ├── age.mjs             # Age view template (pie chart + sidebar)
│   ├── drug_type.mjs       # Drug type view template (pie chart + sidebar)
│   ├── drug_frequency.mjs  # Frequency view template (bar chart)
│   └── error.mjs           # 404/error page template
├── static/
│   └── dark.css            # Dark theme stylesheet
├── img/
│   ├── AgePhotos/          # Representative images per age group
│   └── DrugPhotos/         # Representative images per substance
├── package.json
└── package-lock.json
```

---

## Routes

| Route | Description |
|---|---|
| `GET /` | Homepage with navigation links |
| `GET /age/:age` | Drug usage breakdown for a specific age group (pie chart) |
| `GET /drug_type/:type` | Usage of a specific drug across all age groups (pie chart) |
| `GET /drug_frequency/:type` | Usage frequency of a specific drug across all age groups (bar chart) |

---

## Contributors

- **Emmanuel Adekoya** ([@K-Sqr](https://github.com/K-Sqr))
- Abdiwahab
- Reece

---

## License

ISC
