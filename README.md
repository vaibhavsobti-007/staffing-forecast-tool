# 📊 AI-Powered Staffing Forecast Tool

> An intelligent staffing forecast tool that calculates **revenue, cost, and margin** across work numbers — with AI column mapping, formatted Excel output, and an interactive **what-if scenario analyser**.

![Tool Screenshot](https://img.shields.io/badge/status-active-brightgreen) ![n8n](https://img.shields.io/badge/n8n-workflow-orange) ![Ollama](https://img.shields.io/badge/Ollama-llama3.1%3A8b-blue) ![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## ✨ What It Does

Upload 4 Excel files → get a fully formatted forecast Excel report with revenue, cost, and margin breakdowns — plus an AI chat panel to run what-if scenarios on your data.

**Key capabilities:**
- 🤖 **AI column mapping** — uses a local LLM (Ollama/llama3.1:8b) to intelligently map your file's column names to standard fields, so it works with any spreadsheet format
- 💰 **Revenue & cost calculation** — bills by Client Bill Role (rate card) and costs by Consulting Band (band cost card), with vacation deductions
- 📅 **Monthly breakdown** — hours, revenue, cost and margin per WBS per month across the full project timeline
- 📊 **Formatted Excel output** — colour-coded 4-sheet report built in the browser (no server-side Excel library needed)
- 💬 **What-if scenario chat** — ask questions like *"If costs increase 10%, what's the margin impact on each work number?"* and get instant analysis

---

## 🏗️ Architecture

```
Browser (HTML file)
    │
    ├── Reads 4 Excel files → converts to CSV
    ├── POSTs CSV data to n8n webhook
    │
    └── n8n Workflow (6 nodes)
            │
            ├── 1. Webhook — receives POST
            ├── 2. Extract Headers — reads column names + sample rows
            ├── 3. Ollama (llama3.1:8b) — maps columns to standard fields
            ├── 4. Validate Mapping — validates critical fields, detects day rates
            ├── 5. Parse & Calculate — computes hours, revenue, cost, margin per WBS/month
            ├── 6. Build Report — structures data with totals
            └── 7. Respond with JSON — returns report to browser
                        │
                        └── Browser builds formatted Excel (SheetJS)
                            + unlocks What-if Scenario Analyser
```

**Key design decisions:**
- The LLM does **only column mapping** — all arithmetic is deterministic JavaScript
- Excel is built **client-side** using SheetJS with full cell-level styling
- The what-if engine runs **entirely in the browser** — no API calls required

---

## 📁 Files in This Repo

| File | Description |
|------|-------------|
| `staffing_forecast_v8.html` | Main application — open this in your browser |
| `Staffing_Forecast_v7_final.json` | n8n workflow — import this into n8n |
| `staffing_sheet_v8_usd.xlsx` | Sample staffing data (18 resources, 4 WBS) |
| `role_rate_card_v8_usd.xlsx` | Sample client bill rates (USD) |
| `band_cost_card_v8_usd.xlsx` | Sample internal band cost rates (USD) |
| `vacation_tracker_v8_usd.xlsx` | Sample vacation/leave data |

---

## 🚀 Setup & Usage

### Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| [Docker](https://docker.com) | Runs n8n | [docs.docker.com](https://docs.docker.com/get-docker/) |
| [Ollama](https://ollama.ai) | Runs the local LLM | [ollama.ai](https://ollama.ai) |
| A modern browser | Runs the HTML app | Chrome / Safari / Firefox |

---

### Step 1 — Start Ollama

```bash
# Pull the model (first time only, ~5GB download)
ollama pull llama3.1:8b

# Start Ollama with network access enabled
OLLAMA_HOST=0.0.0.0 OLLAMA_ORIGINS=* ollama serve
```

> **Mac users:** You can also run `brew services start ollama` but make sure `OLLAMA_HOST=0.0.0.0` is set so Docker can reach it.

---

### Step 2 — Start n8n in Docker

```bash
docker run -d \
  --name n8n \
  -p 5678:5678 \
  --add-host=host.docker.internal:host-gateway \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

Then open **http://localhost:5678** and create a free account.

> The `--add-host` flag is required so n8n (inside Docker) can reach Ollama (on your Mac/host).

---

### Step 3 — Import the n8n Workflow

1. In n8n, go to **Workflows → Import from file**
2. Select `Staffing_Forecast_v7_final.json`
3. Open the **"Ollama - Map Columns"** node
4. Update the URL to your machine's actual local IP address:
   ```
   http://YOUR_LOCAL_IP:11434/api/generate
   ```
   Find your IP with `ipconfig getifaddr en0` (Mac) or `hostname -I` (Linux)
5. Click **Publish** (top right)

---

### Step 4 — Open the App

1. Open `staffing_forecast_v8.html` in your browser (double-click or `File → Open`)
2. Upload your 4 Excel files (or use the sample files in this repo)
3. Set the webhook URL to:
   ```
   http://localhost:5678/webhook/staffing-forecast-v7
   ```
4. Click **⚡ Generate Forecast**

The tool will:
- Send your files to n8n
- Run AI column mapping via Ollama (takes 15–60 seconds)
- Calculate all metrics
- Download a formatted Excel report
- Unlock the what-if scenario chat panel

---

## 📂 Input File Format

The tool uses AI to map columns — so exact column names don't matter. It just needs files that contain roughly these fields:

### Staffing Sheet
| Required concept | Example column names |
|---|---|
| Person full name | `Employee Full Name`, `Staff Name`, `Resource` |
| Client Bill Role | `Client Bill Role`, `Billing Role`, `Role` |
| Consulting Band | `Consulting Band Role`, `Band`, `Grade` |
| Project start date | `Project Start Date`, `Start`, `From` |
| Project end date | `Project End Date`, `End`, `To` |
| WBS / charge code | `WBS/Charge Code`, `Work Number`, `Project Code` |
| Weekly hours | `Contracted Hrs/Wk`, `Hours/Week` |

### Role Rate Card
| Required concept | Example column names |
|---|---|
| Client Bill Role | `Client Bill Role`, `Role Name` |
| Revenue rate | `Hourly Rate`, `Standard Day Rate` (÷8 auto-detected) |

### Band Cost Card
| Required concept | Example column names |
|---|---|
| Band label | `Consulting Band Role`, `Band`, `Grade` |
| Cost rate | `Hourly Cost Rate`, `Daily Cost Rate` (÷8 auto-detected) |

### Vacation Tracker
| Required concept | Example column names |
|---|---|
| Person name | `Staff Member`, `Employee Name` |
| Leave start | `Leave From`, `Start Date` |
| Leave end | `Leave To`, `End Date` |

---

## 📊 Excel Output — 4 Sheets

| Sheet | Contents |
|---|---|
| **Summary** | Portfolio totals + one row per WBS with hours, revenue, cost, margin, margin % |
| **Resource Detail** | Per-resource breakdown within each WBS — rates, hours, revenue, cost, margin, vacation |
| **Monthly Breakdown** | Hours, revenue, cost, margin per WBS per calendar month |
| **Column Mapping** | Shows exactly how the AI mapped your file columns to standard fields |

Color coding:
- 🟦 **Navy** — title / header rows
- 🟩 **Green** — portfolio/grand totals
- 🔵 **Teal** — WBS group headers
- 🔴 **Red text** — negative margins
- 🟢 **Green bg** — successfully mapped columns
- 🔴 **Red bg** — unmapped columns

---

## 💬 What-if Scenario Analyser

After generating your forecast, the right panel unlocks. You can ask questions in plain English:

| Question | What it does |
|---|---|
| `If costs increase 10%, what's the margin impact?` | Recalculates margin = revenue − (cost × 1.10) per WBS |
| `If revenue rates increase 15% on WBS-102?` | Shows before/after revenue and margin for that WBS |
| `Costs +10% and rates +5% — net impact?` | Combined scenario across all WBS |
| `Show sensitivity table for -20% to +20% cost changes` | Full matrix of margin outcomes |
| `Which work number has the best margin?` | Ranks WBS by margin % |
| `Which resources are most costly?` | Top 10 resources by total internal cost |
| `Remove the most expensive resource from WBS-101` | Models headcount reduction impact |
| `Show monthly breakdown for WBS-103` | Monthly revenue and margin % trend |

The engine runs **100% locally in your browser** — no internet connection or API key needed.

---

## 🔧 Troubleshooting

| Problem | Fix |
|---|---|
| `NetworkError — cannot reach n8n` | Make sure n8n Docker container is running: `docker start n8n` |
| `Ollama offline` (red dot) | Run `OLLAMA_HOST=0.0.0.0 OLLAMA_ORIGINS=* ollama serve` |
| Workflow times out | Ollama can take 1–3 min on first run. Wait and retry. |
| `Column mapping failed` | Check the Ollama URL in the n8n node uses your actual local IP, not `localhost` |
| Revenue/cost shows 0 | Open n8n executions, check the Validate Mapping node output — look for mapping warnings |
| Excel opens without colour | Make sure you're using the latest `staffing_forecast_v8.html` — older versions had a `cellStyles` bug |

---

## 🧱 Tech Stack

| Component | Technology |
|---|---|
| Workflow automation | [n8n](https://n8n.io) (self-hosted via Docker) |
| Local LLM | [Ollama](https://ollama.ai) + [llama3.1:8b](https://ollama.ai/library/llama3.1) |
| Excel parsing (input) | [SheetJS / xlsx](https://sheetjs.com) |
| Excel generation (output) | SheetJS with cell-level styling (`cellStyles: true`) |
| Frontend | Vanilla HTML/CSS/JS — no framework, no build step |
| What-if engine | Custom JavaScript scenario calculator |

---

## 🗺️ Roadmap / Ideas

- [ ] Support for multiple WBS allocations per person (partial % allocation)
- [ ] Export what-if scenario results to Excel
- [ ] Risk flagging (resources near end date, low margin WBS, high vacation periods)
- [ ] PDF report generation
- [ ] Support for Google Sheets as input
- [ ] Multi-currency support

---

## 📄 License

MIT — free to use, modify, and distribute.

---

## 🙏 Built With

- [n8n](https://n8n.io) — open-source workflow automation
- [Ollama](https://ollama.ai) — local LLM inference
- [SheetJS](https://sheetjs.com) — Excel read/write in the browser
- Built iteratively with [Claude](https://claude.ai) by Anthropic
