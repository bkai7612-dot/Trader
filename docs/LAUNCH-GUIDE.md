# How to Launch PAIDROCKET: Simple Guide

> **Important:** the trading app is **not built yet**. Only the planning (Milestone 1) is done.
> Part 1 is what you can do **today**. Part 2 shows how launching will work **once the app exists**.
> Everything is **PAPER (pretend money)**. No real money is ever used.

---

## Part 1: Do this now (5 minutes)

These steps unblock the next stage.

### Step 1: Let Claude reach the exchange data

1. Open this Claude Code session on the web or in the app.
2. Click the **environment name** in the session's title bar.
3. Click **Edit**.
4. Find **Network access**.
5. Under **Allowed domains**, add these three lines:
   ```
   api.coinbase.com
   api.exchange.coinbase.com
   api.kraken.com
   ```
6. Leave **"Allow package managers"** ticked.
7. Click **Save**.

Official help: <https://code.claude.com/docs/en/cloud-environments#network-access>

### Step 2: Tell Claude to continue

Type:

```
Network is open. Continue with Milestone 2.
```

That's it for now. Claude will check exchange fees and minimum order sizes, then build the app.

> **You do NOT need:** an exchange account, API keys, passwords or money. Never paste a password or seed phrase into the chat.

---

## Part 2: Launching the app (after it's built, Milestones 3–6)

These commands are **planned** and **won't work yet**. This guide will be updated with the real ones when they exist.

### Step 1: Get a computer that stays on

The app has to run all the time to watch prices. Use one of these:
- **Your own PC or laptop**, left on, or
- **A small cloud server**, typically a few pounds a month.

### Step 2: Install Python

- Download **Python 3.12 or newer** from <https://www.python.org/downloads/>.
- On Windows, tick **"Add Python to PATH"** during install.

### Step 3: Download the project

Open a terminal (on Windows: search for **"Terminal"** or **"PowerShell"**) and type:

```
git clone https://github.com/bkai7612-dot/Trader.git
cd Trader
```

### Step 4: Install and start (planned commands)

```
pip install -r requirements.txt
python -m paidrocket start
```

### Step 5: Open the dashboard

- Open your web browser and go to **http://127.0.0.1:8000**.
- You should see **"MODE: PAPER"** at the top and **£50.00** starting equity.

### Step 6: Daily use

| You want to… | Do this |
|---|---|
| Check progress | Open the dashboard |
| Pause trading | Click **Pause** |
| Stop everything right now | Click **Emergency Stop** |
| Turn it off | Press **Ctrl + C** in the terminal |

---

## Good to know

- 🧪 **Paper only.** No real money is used. Live trading is a separate step later and needs your explicit approval.
- 📉 **Losses will happen.** The app will show them honestly and never hide or reset them.
- 🎯 **£1,000 is a stretch goal, not a promise.** With the safety limits, growth will be slow.
- 🛑 **Stop-losses aren't guaranteed.** In a crash, prices can jump past them.
