# Agentic Financial Research Copilot

An intelligent, agentic investment assistant powered by **Gemini 2.5 Flash**, real‑time market data, web trends, and long‑term portfolio memory.

> This project simulates a proactive wealth manager that can analyze your current portfolio, discover new opportunities, propose risk‑aware allocations, and remember your choices across sessions.

---

## What This Project Does

This project builds an **Agentic Financial Research Copilot** that can:

- Analyze your **current portfolio** (live prices, total value, profit/loss).
- Suggest **new investments** based on:
  - Web trends (`search_market_trends` via DuckDuckGo)
  - Fundamentals (`get_asset_fundamentals`)
  - Technicals & risk (`calculate_technical_indicators`)
- Construct **risk‑aware portfolios** with `allocate_portfolio`.
- Maintain **long‑term memory** in `user_portfolio.json` so it remembers:
  - Your holdings (symbols & shares)
  - Your cash balance
  - Your approximate cost basis
- Operate in a **human‑like flow**:
  1. Analyze → 2. Suggest (draft) → 3. Refine (if needed) → 4. Execute (save to JSON on your “yes”).

---

## How This Differs From a Regular Financial Script

Unlike a simple “stock analyzer” that only shows static metrics, this project:

- Uses **natural language** for all interactions, not rigid inputs.
- Combines **web search + quantitative tools** (“trend hunter” + fundamentals + volatility).
- Keeps **stateful memory** of your portfolio and builds on past decisions.
- Provides an **“Underrated Stocks to Consider”** section:
  - Highlights 1–2 high‑potential but higher‑risk names not in your core plan.
  - Explains why they’re interesting and why they’re not in the main allocation.
- Asks for **confirmation** before committing changes to your portfolio JSON, mirroring how a real advisor would propose a plan, discuss changes, then execute.

> This is not just a static analyst; it behaves like a conversational portfolio partner that learns from your past portfolio and helps guide future decisions.

---

## Key Features

- **Natural language interface** (“I have ₹50k, build a safe AI portfolio”)
- **Real‑time data** via `yfinance` (prices, history, fundamentals, technicals)
- **Web trend discovery** using DuckDuckGo search for emerging / underrated stocks
- **Risk engine** (annualized volatility, trend analysis, basic risk labelling)
- **Portfolio construction** with risk‑aware weights (not just equal weight)
- **Long‑term memory** in `user_portfolio.json` (holdings, cash, cost basis)
- **Stateful flow**: suggest → refine → confirm → execute (and then save)
- **“Underrated stocks to consider”** section with reasoning and education

---

## Project Overview

This copilot is designed to behave like a **junior portfolio manager + research analyst** in one:

- If you ask about your **current portfolio**, it:
  - Loads your saved holdings from JSON
  - Fetches live prices for each asset
  - Computes current value, cost basis, and P/L
  - Analyzes risk and trend for each holding
- If you ask for **investment ideas**:
  - Searches the web for trending / high‑potential themes
  - Extracts candidate tickers and verifies them with fundamentals & technicals
  - Builds a core portfolio plus a list of “underrated” watchlist names
- If you ask to **invest X amount**:
  - Proposes a draft allocation
  - Lets you request changes
  - Only when you say “Yes / confirm” does it write to `user_portfolio.json`

Everything runs inside a single notebook / script using an agentic orchestration pattern.

---

## Architecture & Flow

The system uses a **Master Orchestrator Agent** plus several specialized tools.

### High‑Level Flow

```mermaid
graph TD
    A[Start] --> B{User Input};
    B --> C[Master Orchestrator Agent];

    C --> D{What does the user want?};

    D -->|Analyze current portfolio / future outlook| E[Load & Value Portfolio<br/>get_portfolio_state];
    D -->|Ask about a stock / sector / idea| F[Research & Trend Discovery<br/>search_market_trends<br/>get_asset_fundamentals<br/>calculate_technical_indicators];
    D -->|Invest X amount / change allocation| G[Draft Portfolio Plan<br/>allocate_portfolio];

    E --> H[Risk & Outlook Analysis<br/>LLM Reasoning];
    F --> H;
    G --> I{User happy with plan?};

    I -->|No – refine| G;
    I -->|Yes – execute| J[Update Portfolio Memory<br/>update_portfolio_memory];

    H --> K[Build Structured Response<br/>Core Analysis + Underrated Picks + Glossary];
    J --> E;  %% after execution, next analysis will use updated portfolio

    K --> L[Display Answer to User];
    L --> M[End];
```

### Main Tools / Components

- `search_market_trends(query)`  
  Uses DuckDuckGo search to find trending / emerging investment ideas (e.g., “undervalued AI stocks 2025”).

- `get_asset_price_history(symbol, ...)`  
  Fetches historical prices for basic charting / trend context.

- `get_asset_fundamentals(symbol)`  
  Gathers key metrics like P/E, market cap, sector, etc.

- `calculate_technical_indicators(symbol)`  
  Computes returns, annualized volatility, moving averages, and simple trend signals.

- `allocate_portfolio(tickers, total_amount, weights=None)`  
  Builds a **draft allocation** (shares, value, weights).  
  Does **not** save; used for “what‑if” planning.

- `get_portfolio_state()`  
  Loads `user_portfolio.json`, fetches **live prices**, and returns:

  - holdings (symbol, shares, current price, value)
  - cash
  - total portfolio value
  - cost basis
  - profit / loss and % return

- `update_portfolio_memory(action, details)`  
  Applies changes to the saved portfolio:

  - `deposit_cash`
  - `buy_asset`
  - `batch_update` (merge new allocation into existing holdings)
  - `set_portfolio` (hard overwrite when user asks to “sell everything and rebuild”)

- **Master Agent (Gemini 2.5 Flash)**  
  Orchestrates all of the above based on user intent and conversation state.

---

## Conversation Modes (Master Orchestrator Logic)

The Master Agent switches between four logical “modes”:

### 1. Analysis Mode

Triggered by queries like:

> “How is my portfolio doing?”  
> “What are the chances my portfolio will grow in the future?”

Actions:

- Call `get_portfolio_state`
- If empty → respond:  
  _“This looks like your first investment! Let’s build a foundation.”_
- If not empty:
  - Show **Portfolio Status**: current value vs cost basis, P/L %
  - For each holding, call fundamentals + technicals tools
  - Flag high‑volatility or weak‑trend assets
  - Provide a plain‑English “future outlook” based on risk and trend

---

### 2. Advisory Mode

Triggered by queries like:

> “I have ₹50,000 to invest in tech/AI.”  
> “Suggest some growth stocks in clean energy.”

Actions:

1. **Check memory** with `get_portfolio_state` (don’t overload existing risk).
2. **Discovery** with `search_market_trends` for new / underrated ideas.
3. **Verification** with `get_asset_fundamentals` and `calculate_technical_indicators`.
4. **Selection** of:
   - 3–5 **core** assets (balanced risk/reward)
   - 1–2 **underrated** “watchlist” assets (high potential, higher risk)
5. **Plan drafting** with `allocate_portfolio` (no saving yet).

Output (strict structure):

1. **Portfolio Status**

   - If existing portfolio: current value, P/L, brief summary.
   - If first time: explicitly say so.

2. **Market Insight**

   - E.g., “Based on recent search trends, I focused on AI and cloud infrastructure…”

3. **Core Portfolio Analysis**

   - Fundamentals table (symbol, sector, P/E, market cap, etc.)
   - Technicals table (volatility, recent trend)
   - Short explanation of each choice and its role in the portfolio.

4. **Proposed Allocation**

   - Table with: Asset | Price | Shares | Value | Weight %

5. **Underrated Stocks to Consider**

   - 1–2 names **not** in the main allocation
   - Why they have high growth potential
   - Why they were excluded from the core (e.g., too volatile, small cap)

6. **Glossary**

   - Simple definitions of any technical terms used (e.g., volatility, P/E, diversification).

7. **Confirmation Prompt**
   - E.g., _“Does this plan look good? Say ‘yes’ to execute or tell me what to change.”_

---

### 3. Refinement Mode

Triggered by queries like:

> “Remove BTC from the plan.”  
> “Can we reduce Nvidia and add more Microsoft?”

Actions:

- Adjust the previous **draft** allocation (mentally / via new `allocate_portfolio` call).
- Re‑generate the **Advisory Mode** output format with the updated plan.
- Ask for confirmation again.

---

### 4. Execution Mode

Triggered only when user explicitly confirms:

> “Yes, go ahead.”  
> “Confirm this plan.”

Actions:

- Use `update_portfolio_memory` to:
  - **Merge** new investments (`batch_update`) into existing holdings, or
  - **Overwrite** (`set_portfolio`) when user wants a full rebuild.
- Confirm back to user with updated **Portfolio Status** summary.

---

## Why This Is Not a “Regular” Financial Analyst Script

Most “financial analysis” projects on GitHub are:

- Single‑purpose scripts (e.g., download prices and plot a chart)
- Static dashboards (e.g., Streamlit app with fixed metrics)
- Stateless: they forget everything about you between runs

This project is fundamentally different in three ways:

### 1. Agentic, Not Just Analytic

- **Regular tools:** You decide everything, tools only show numbers.
- **This copilot:**
  - **Proposes** ideas based on web trends and risk
  - **Explains** why each asset is chosen or excluded
  - **Asks for confirmation** and then executes simulated trades
  - **Suggests underrated stocks** beyond what you initially asked, to broaden your horizon

### 2. Long‑Term, Personalized Memory

- **Regular tools:** Every run is a clean slate.
- **This copilot:**
  - Stores your holdings and cost basis in `user_portfolio.json`
  - Re‑evaluates your existing positions each time with **live prices**
  - Tailors new suggestions around your current exposure  
    (e.g., “You’re already heavy in AI, let’s diversify into healthcare.”)

### 3. “Trust but Verify” Workflow

- **Regular LLM demos:** Hallucinate tickers or repeat headlines without checking data.
- **This copilot:**
  - Finds ideas via web search, **then** validates them with fundamentals & technicals
  - Filters out extreme volatility or unrealistic valuations unless you opt for high risk
  - Uses a clear, repeatable **plan → confirm → execute** loop

In short, this is not a static analyst; it’s an **assistive, conversational portfolio partner** that learns your situation over time.

---

## Setup & Installation

### Prerequisites

- Python 3.10+
- Google API key for **Gemini 2.5 Flash**
- Basic familiarity with running Jupyter notebooks or Python scripts

### Installation

1.  **Clone the repository:**

    ```bash
    git clone https://github.com/YUHESHWAR/Google-Capstone.git
    ```

2.  **Create a virtual environment and install dependencies:**

    ```bash
    python -m venv venv
    source venv/bin/activate  # On Windows use `venv\Scripts\activate`
    pip install -r requirements.txt
    ```

    _(Note: A `requirements.txt` file is expected in the project root containing necessary libraries.)_

3.  **Set up your Google API Key:**
    Create a `.env` file in the project root and add your Gemini API key:

    ```
    GOOGLE_API_KEY="YOUR_GEMINI_API_KEY"
    ```

    _(Ensure you have enabled the Gemini 2.5 Flash model for your API key.)_

4.  **Initialize User Portfolio (Optional):**
    If you don't have an existing `user_portfolio.json`, the system will likely create one on first use. You can also create an empty one:

    ```
    # user_portfolio.json
    {}
    ```

5.  **Run the Copilot:**
    Launch Jupyter Notebook or JupyterLab and open `FinanceAssistant.ipynb`. Run the cells sequentially to initialize the agent and start the interaction.
    ```bash
    jupyter notebook FinanceAssistant.ipynb
    ```
