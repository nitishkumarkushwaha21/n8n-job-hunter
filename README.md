# Fresher Job Alerts — n8n workflow

An n8n workflow that checks job boards and company career pages every hour, uses Google Gemini (free tier) to screen each new job, saves the good ones to a Google Sheet and emails them to you.

It is set up for a **fresher / new graduate in India** looking for **software engineering roles**, but every filter can be changed.

**Everything here is free:** n8n (self-hosted), Google Sheets, the Gemini API free tier and Gmail.

---

## Table of contents

1. [What it does](#1-what-it-does)
2. [What you need](#2-what-you-need)
3. [Install n8n on your computer](#3-install-n8n-on-your-computer)
   - [Option A: Docker (recommended)](#option-a-docker-recommended)
   - [Option B: Node.js](#option-b-nodejs)
4. [Create the Google Sheet](#4-create-the-google-sheet)
5. [Create the three credentials](#5-create-the-three-credentials)
6. [Import the workflow and connect credentials](#6-import-the-workflow-and-connect-credentials)
7. [Configure the Settings node](#7-configure-the-settings-node)
8. [Test and turn it on](#8-test-and-turn-it-on)
9. [Everyday use](#9-everyday-use)
10. [How the workflow works, node by node](#10-how-the-workflow-works-node-by-node)
11. [Customizing](#11-customizing)
12. [Troubleshooting](#12-troubleshooting)
13. [Before you push this repo](#13-before-you-push-this-repo)

---

## 1. What it does

Every hour:

1. **Collects jobs** from:
   - Y Combinator job board (India + remote pages)
   - We Work Remotely (programming RSS feed)
   - RemoteOK
   - Himalayas (entry-level, open to India)
   - Company career pages on Greenhouse, Lever and Ashby (Stripe, Databricks, Postman, MongoDB and more)
2. **Pre-filters for free** (no AI): the title must look like a software role, must not be senior, and the location must work from India.
3. **Skips jobs it has already checked**, using a `Seen` tab in your Google Sheet.
4. **Asks Gemini** to judge each new job: is it a real software role, is it open to freshers (≤ 1 year), can someone in India take it, and what is the salary in ₹/month.
5. **Applies your rules** in code and makes the final decision.
6. **Saves** every checked job to `Seen`, saves matches to `Jobs`, and **emails** you the matches.

**Default filters**

| Filter | Default |
|---|---|
| Roles | SDE / SWE, full-stack, frontend, backend, forward deployed, founding / product engineer and similar |
| Experience | Fresher up to 1 year. Jobs that don't mention experience are kept and marked "Not stated" |
| Location | In India (on-site, hybrid or remote), or remote and open to India / Asia / worldwide |
| Salary | At least ₹70,000/month (≈ 8.4 LPA). Jobs with no salary listed are kept |

---

## 2. What you need

- A computer (Windows, macOS or Linux) with **Docker Desktop** *or* **Node.js 20.19 – 24.x**
- A **Google account** (for Google Cloud, Google Sheets and Gemini)
- A **Gmail account** to send the alerts from, with **2-Step Verification** turned on
- About **30–45 minutes** for the first setup

> **The workflow only runs while your computer is on, awake and n8n is running.** Missed hours are not replayed, but nothing is lost: the next run finds every job that is still listed, because "new" is decided by the `Seen` tab, not by time.

---

## 3. Install n8n on your computer

Choose **one** option.

| | Docker (Option A) | Node.js (Option B) |
|---|---|---|
| Runs in the background | ✅ Yes | ❌ Only while the terminal is open |
| Starts again by itself | ✅ Whenever Docker starts | ❌ You start it each time |
| Updates | Only when you choose | Only when you choose |
| Kept apart from the rest of your computer | ✅ | ❌ |
| Best for | Day-to-day use | Quick tests, or if Docker is too heavy for your machine |

### Option A: Docker (recommended)

**A1. Install and open Docker Desktop**

1. Download Docker Desktop from [docker.com](https://www.docker.com/products/docker-desktop/) and install it.
2. Open Docker Desktop and wait until it shows **"Engine running"** (bottom-left).
3. *Optional:* **Settings ⚙️ → General → "Start Docker Desktop when you sign in to your computer"**.
   - Ticked: n8n starts automatically every time you log in.
   - Unticked: n8n runs only when you open Docker yourself. Good if you want to save RAM.

**A2. Create the n8n container (only once)**

Open **PowerShell** (Windows) or **Terminal** (macOS/Linux) and run this single line:

```bash
docker run -d --name n8n --restart unless-stopped -p 127.0.0.1:5678:5678 -e GENERIC_TIMEZONE=Asia/Kolkata -e TZ=Asia/Kolkata -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

What each part means:

| Part | Meaning |
|---|---|
| `-d` | Run in the background |
| `--name n8n` | Name the container `n8n` so you can start/stop it by name |
| `--restart unless-stopped` | Start n8n whenever Docker starts, unless you stopped it yourself |
| `-p 127.0.0.1:5678:5678` | Open n8n at `localhost:5678`, **only on your computer** (not visible to others on the same Wi-Fi) |
| `-e GENERIC_TIMEZONE=... -e TZ=...` | Use Indian time for schedules and timestamps |
| `-v n8n_data:/home/node/.n8n` | Keep your workflows and credentials in a Docker volume called `n8n_data`, so they survive updates |

The first time, Docker downloads n8n (a few minutes). When it finishes, it prints one long ID like `b42a0d9b0fc3...`. That means it worked.

**A3. Open n8n**

1. Go to **http://localhost:5678**. If it doesn't load, wait 30 seconds and refresh.
2. Fill in **"Set up owner account"**. This account exists only on your computer.
3. Skip the survey if you like.
4. *Optional:* n8n may email you a **free license key**. To activate it: **Settings → Usage and plan → Enter activation key**. The workflow does not need it.

**A4. Commands you'll use**

```bash
docker stop n8n             # pause n8n (frees most of its RAM)
docker start n8n            # start it again
docker logs -f n8n          # watch what n8n is doing (Ctrl + C to exit)
docker desktop stop         # close Docker completely (frees all its RAM)
docker desktop start        # open Docker again
```

If `docker desktop` isn't recognised, quit or open Docker from its taskbar / menu-bar icon instead.

**A5. Updating n8n later**

Your workflows and credentials live in the `n8n_data` volume, so this is safe:

```bash
docker pull docker.n8n.io/n8nio/n8n
docker rm -f n8n
```

Then run the **A2** command again.

> ⚠️ **Never run `docker volume rm n8n_data`.** It deletes your workflows, saved credentials and the key that unlocks them.

---

### Option B: Node.js

**B1. Check your Node.js version**

```bash
node -v
```

n8n needs **Node.js 20.19 to 24.x**. If you have an older or newer version, install an LTS version (22 or 24) from [nodejs.org](https://nodejs.org/). If you need several Node versions on one machine, use a version manager (`nvm` / `nvm-windows`).

**B2. Install n8n**

```bash
npm install -g n8n
```

Use the global install rather than `npx n8n`, so the version doesn't change on you by surprise.

**B3. Start n8n with Indian time**

Windows (PowerShell):

```powershell
$env:GENERIC_TIMEZONE="Asia/Kolkata"; n8n start
```

macOS / Linux:

```bash
GENERIC_TIMEZONE=Asia/Kolkata n8n start
```

Then open **http://localhost:5678** and create your owner account (same as A3).

**B4. Things to know**

- **Keep the terminal window open.** Closing it, or pressing `Ctrl + C`, stops n8n.
- **After a reboot**, run the B3 command again.
- Your data is stored in `C:\Users\<you>\.n8n` (Windows) or `~/.n8n` (macOS/Linux). Don't delete this folder.
- **Update:** `npm update -g n8n`

---

## 4. Create the Google Sheet

1. Create a new Google Sheet (any name, e.g. *Job Alerts*).
2. Rename the first tab to **`Jobs`** (exact spelling, capital J).
3. Add a second tab named **`Seen`** (capital S).
4. Leave both tabs **completely empty**. Column headers are created on the first run.
5. Copy the **sheet ID** from the URL:

   ```
   https://docs.google.com/spreadsheets/d/THIS_LONG_PART_IS_THE_ID/edit
   ```

You'll paste it into the Settings node in [step 7](#7-configure-the-settings-node).

**What the tabs will contain**

| Tab | Columns | Purpose |
|---|---|---|
| `Jobs` | Found (IST), Title, Company, Location, Salary, Experience, Source, Link, Why, Status | Every matching job. Use **Status** yourself to track applications (e.g. *Applied*, *Not interested*). |
| `Seen` | id, key, title, company, verdict, checked_at | Every job the AI has checked, matched or not, so nothing is checked or emailed twice. |

---

## 5. Create the three credentials

The workflow needs three logins. They are stored **inside your n8n**, never in the workflow file.

### 5.1 Google Sheets: service account

A service account is a "robot" Google account. It is used instead of a normal Google sign-in, which would expire every 7 days for personal projects.

**In Google Cloud:**

1. Go to [console.cloud.google.com](https://console.cloud.google.com/) and create a project (top bar → project picker → **New project**).
2. Go to **APIs & Services → Library** and enable:
   - **Google Sheets API**
   - **Google Drive API**
3. Go to **IAM & Admin → Service Accounts → Create service account**. Give it a name (e.g. `n8n-sheets`), click **Done**. No roles are needed.
4. Click the new service account → **Keys** tab → **Add key → Create new key → JSON**. A `.json` file downloads.
   > 🔒 This file is a password. Keep it private and **never put it in your GitHub repo**.
5. Open your Google Sheet → **Share** → paste the service account's email (looks like `n8n-sheets@your-project.iam.gserviceaccount.com`) → give it **Editor** → **Share**.
   > If you skip this, every Sheets step fails with a permission error.

**In n8n:**

1. Go to **Credentials** (left menu, or **Overview → Credentials**) → **Create credential** → **Google Service Account API**.
2. Open the downloaded `.json` file in Notepad / a text editor and copy:
   - `client_email` → **Service Account Email**
   - `private_key` → **Private Key**. Copy everything between the quotes, from `-----BEGIN PRIVATE KEY-----` to `-----END PRIVATE KEY-----\n`.
3. **Save**. n8n tests the connection and should show success.

### 5.2 Gemini API key (AI screening)

1. Go to [aistudio.google.com](https://aistudio.google.com/) → **Get API key** → **Create API key**. No card is needed for the free tier.
2. Copy the key.
3. In n8n: **Create credential → Google Gemini (PaLM) Api**. Paste the key into **API Key** and leave **Host** as it is.
4. **Save**.

### 5.3 Gmail SMTP (sending the email)

Two email addresses are involved:

| | What it is | Where it goes |
|---|---|---|
| **Sender** | The Gmail account that *sends* the alerts | SMTP credential **User** + `emailFrom` in Settings |
| **Receiver** | The inbox where you *read* the alerts | `emailTo` in Settings only |

They can be the same address or different ones. **The SMTP User and `emailFrom` must always be the same address.**

1. Sign in to the **sender** Gmail account.
2. Make sure **2-Step Verification** is on: [myaccount.google.com/security](https://myaccount.google.com/security).
3. Go to [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords), type a name (e.g. `n8n`) and click **Create**.
4. Copy the 16-letter app password. Google shows it only once.
5. In n8n: **Create credential → SMTP** and fill in:

   | Field | Value |
   |---|---|
   | User | the **sender** Gmail address |
   | Password | the 16-letter app password |
   | Host | `smtp.gmail.com` |
   | Port | `465` |
   | SSL/TLS | **On** |

6. **Save**.

---

## 6. Import the workflow and connect credentials

**6.1 Import**

1. In n8n, go to **Overview** → **Create workflow** (or **Build a workflow**).
2. Click **⋯** (top right) → **Import from File…**
3. Choose `Fresher-Job-Alerts.n8n.json` from this repo.
4. Press **Ctrl + S** (Cmd + S on Mac) to save.

**6.2 Connect each credential**

Imported nodes point at credentials that don't exist on your machine yet, so pick yours in each node. Double-click the node, choose from the **Credential** dropdown, close it.

| Node | Credential |
|---|---|
| **Read Seen jobs** | Google Service Account API |
| **Save to Seen tab** | Google Service Account API |
| **Save to Jobs tab** | Google Service Account API |
| **Gemini screen** | Google Gemini (PaLM) Api |
| **Send email** | SMTP |

Press **Ctrl + S** again.

---

## 7. Configure the Settings node

Double-click the **Settings** node. Everything you need to change is at the top of its code. You don't need to edit any other node.

### 7.1 Required

| Setting | What to put |
|---|---|
| `emailTo` | The inbox where you want to receive alerts |
| `emailFrom` | The **sender** Gmail address (same as the SMTP credential **User**) |
| `sheetId` | The sheet ID from [step 4](#4-create-the-google-sheet) |

### 7.2 AI screening

| Setting | Default | Meaning |
|---|---|---|
| `geminiModel` | `gemini-flash-lite-latest` | Gemini model to use. If you get a 404 error, set an exact model ID from AI Studio instead. |
| `geminiBatchSize` | `12` | Jobs sent to Gemini in one request |
| `maxNewJobsPerRun` | `96` | Maximum new jobs checked per run. The rest wait for the next hour. Protects your free quota. |

### 7.3 Your filters

| Setting | Default | Meaning |
|---|---|---|
| `maxYearsExperience` | `1` | Highest *minimum* experience a job may ask for |
| `includeUnclearExperience` | `true` | Keep jobs that don't mention experience (marked "Not stated") |
| `minMonthlyInr` | `70000` | Minimum salary in ₹ per month. 10 LPA ≈ 83,333 |
| `salaryCheck` | `'max'` | `'max'`: pass if the **top** of the range reaches the minimum. `'min'`: the **bottom** must reach it. |
| `usdToInr`, `eurToInr`, `gbpToInr` | `95`, `110`, `127` | Approximate rates used to convert foreign salaries. Update them now and then. |
| `roleKeywords` | list | The job title must match **at least one** (regular expressions, case-insensitive) |
| `excludeTitle` | list | The job title must match **none** of these |

> In the code, backslashes are doubled: `'\\bsde\\b'` means the regex `\bsde\b` (the word "sde" on its own).

### 7.4 Sources

| Setting | What it is |
|---|---|
| `ycPages` | Y Combinator job board pages to read |
| `wwrFeeds` | We Work Remotely RSS feeds |
| `remoteOk` | `true` / `false` to turn RemoteOK on or off |
| `himalayasQueries` | Search terms for Himalayas |
| `greenhouse`, `lever`, `ashby` | Company slugs (see [11.1](#111-add-a-company)) |

---

## 8. Test and turn it on

**8.1 Test run**

1. On the canvas, click **Execute workflow**.
2. Wait **1–3 minutes**. Gemini batches are sent 7 seconds apart.
3. Nodes turn **green ✓** as they finish.

**What to expect**

| Situation | What you see | Is it OK? |
|---|---|---|
| Very first run | Up to 96 jobs checked, rows in `Seen`, matches in `Jobs`, one email | ✅ |
| Later runs with no new jobs | Run stops after **Prepare new jobs** with **0 items**. No email. | ✅ Normal. Everything was already checked. |
| New jobs but none match | Rows added to `Seen` only. No email. | ✅ |
| A node is **red** | That step failed | See [Troubleshooting](#12-troubleshooting) |

To see why a run stopped early: open **Executions** → the run → **Logs** panel → click **Keep only new**. `0 items` means there was nothing new.

**8.2 Turn it on**

Click **Publish** (top right). On older n8n versions, flip the **Active** toggle instead. It now runs every hour while n8n is running.

---

## 9. Everyday use

| You want to… | Do this |
|---|---|
| Pause alerts while doing heavy work | `docker stop n8n` (Docker) or close the terminal (Node.js) |
| Resume | `docker start n8n` (Docker) or the B3 command (Node.js) |
| Check for jobs right now | Open the workflow → **Execute workflow** |
| See past runs | Workflow → **Executions** tab |
| Track applications | Fill in the **Status** column in the `Jobs` tab |
| Back up | Workflow → **⋯ → Download** after any big change |

**Moving from n8n Cloud?** Download the workflow from the cloud (**⋯ → Download**), import it locally ([step 6](#6-import-the-workflow-and-connect-credentials)), re-create the three credentials, then **Unpublish** the cloud copy. If both are running you get every email twice.

---

## 10. How the workflow works, node by node

The canvas has four groups.

### Group 1: Collect job listings

| Node | What it does |
|---|---|
| **Every hour** | Schedule trigger. Starts a run every hour. |
| **Settings** | Holds all your settings and passes them to the other nodes. |
| **Sources** | Builds the list of URLs to fetch from the settings: YC pages, WWR feeds, RemoteOK, Himalayas searches, and one URL per Greenhouse / Lever / Ashby company. |
| **Fetch sources** | Downloads every URL, 4 at a time, with a 30-second timeout and one retry. If a site fails, the run continues and the failure is noted. |
| **Parse sources** | Has one parser per source type. Turns every response into the same job format: id, title, company, location, salary, experience, description, url, postedAt. Long descriptions are trimmed to the parts about experience, pay and location. |
| **Pre-filter** | Free filtering before any AI: the title must match `roleKeywords` and must not match `excludeTitle`. The location must work from India (Indian city or "IN", anywhere / APAC / worldwide, or plain "Remote" not tied to another country). Duplicates within the same run are removed. |

### Group 2: Prepare new jobs

| Node | What it does |
|---|---|
| **Read Seen jobs** | Reads the whole `Seen` tab once. |
| **Keep only new** | Drops any job whose `id`, or company + title + location `key`, is already in `Seen`. Sorts newest first and keeps at most `maxNewJobsPerRun`. If nothing is new, the run ends here. |
| **Needs description?** | Greenhouse's job list has no descriptions. Greenhouse jobs go to the next node, all others skip ahead. |
| **Fetch job description** | Fetches the full description for each new Greenhouse job only. |
| **Attach description** | Cleans the HTML and attaches the important parts of the description to the job. |
| **All new jobs** | Merges both paths back into one list. |

### Group 3: Screen with AI

| Node | What it does |
|---|---|
| **Gemini batches** | Splits the jobs into groups of `geminiBatchSize` and writes the screening prompt, with a strict JSON schema for the answer. For every job Gemini returns: `role_match`, `experience_fit` (yes / no / unclear), `experience_text`, `location_ok`, salary min/max converted to ₹/month, and a short `reason`. |
| **Gemini screen** | Calls the Gemini API, one batch every 7 seconds to stay inside free-tier limits, with up to 3 tries. |

### Group 4: Record and notify matches

| Node | What it does |
|---|---|
| **Apply rules** | Reads Gemini's answers and makes the **final decision in code**: role matches **and** experience is OK **and** location is OK **and** salary passes. Jobs whose Gemini call failed are left out, so they are retried next hour. |
| **Seen rows → Save to Seen tab** | Appends every checked job, matched or rejected, with its verdict to `Seen`. |
| **Matching jobs → Save to Jobs tab** | Appends only matches to `Jobs`. |
| **Build email** | Builds one HTML email with a card per job (title, company, location, salary, experience, source, Gemini's reason). If any source failed during the run, it is listed at the bottom. Nothing is sent when there are no matches. |
| **Send email** | Sends the email through Gmail SMTP. |

---

## 11. Customizing

### 11.1 Add a company

1. Open the company's careers page and click any job.
2. Look at the job's URL:

   | URL looks like | List | Slug |
   |---|---|---|
   | `boards.greenhouse.io/acme/jobs/123` or `job-boards.greenhouse.io/acme/...` | `greenhouse` | `acme` |
   | `jobs.lever.co/acme/...` | `lever` | `acme` |
   | `jobs.ashbyhq.com/acme/...` | `ashby` | `acme` |

3. Add the slug to that list in **Settings**, e.g. `greenhouse: ['stripe', 'acme'],`.
4. Save. If a slug is wrong, the email footer lists it as a failed source.

### 11.2 Change the roles

- To **include** more titles, add patterns to `roleKeywords`, e.g. `'devops'`, `'\\bml engineer'`.
- To **exclude** titles, add patterns to `excludeTitle`.

**Known issue: "Member of Technical Staff" jobs are filtered out.** `roleKeywords` includes "member of technical staff", but `excludeTitle` contains `'\\bstaff\\b'`, which also matches those titles. To keep them, replace that entry in `excludeTitle` with:

```js
'(?<!technical )\\bstaff\\b',
```

### 11.3 Allow internships

Remove `'\\bintern(ship)?\\b'` and `'\\bapprentice'` from `excludeTitle`.

### 11.4 Change the schedule

Open **Every hour**. To run only from 7 AM to 11 PM, switch the rule to **Custom (Cron)** with:

```
0 7-23 * * *
```

The timezone comes from `GENERIC_TIMEZONE` (set to `Asia/Kolkata` in step 3). You can also set it per workflow under **⋯ → Settings → Timezone**.

---

## 12. Troubleshooting

| Problem | Cause and fix |
|---|---|
| `localhost:5678` doesn't open | Docker isn't running, or the container is stopped. Open Docker Desktop, then `docker start n8n`. Check `docker logs n8n`. |
| `port is already allocated` when creating the container | Something else is using port 5678. Use `-p 127.0.0.1:5679:5678` and open `localhost:5679`. |
| `Conflict. The container name "/n8n" is already in use` | The container already exists. Use `docker start n8n`, or `docker rm -f n8n` and create it again. Your data is safe in the volume. |
| Sheets node: *permission denied / caller does not have permission* | Share the sheet with the service account email as **Editor**, and check that the Sheets and Drive APIs are enabled. |
| Sheets node: *sheet not found* | Tab names must be exactly `Jobs` and `Seen`. Check `sheetId`. |
| Gemini: **404** | The model name isn't available. Set `geminiModel` to an exact ID from AI Studio. |
| Gemini: **429** / quota | Free-tier limit reached. Lower `maxNewJobsPerRun`, or wait. Failed jobs are retried next hour automatically. |
| Email: *authentication failed* | The SMTP **User** must be the sender Gmail address, the password must be an **app password** from that same account, and 2-Step Verification must be on. |
| Run succeeded but no email | Usually normal: no new jobs, or none matched. Check **Executions → Keep only new** and the `Seen` tab. |
| Every email arrives twice | Two copies are running (e.g. n8n Cloud and your laptop, or Docker and Node). Unpublish one. |
| A source keeps failing | Websites change their pages; the YC board is the most likely. Remove it from **Settings** or fix its parser in **Parse sources**. |
| Too many / too few jobs | Adjust `roleKeywords`, `excludeTitle`, `minMonthlyInr`, `includeUnclearExperience` or `salaryCheck`. |
| No runs overnight | Your computer was asleep or n8n was stopped. That's expected. The next run catches up on jobs that are still listed. |

**Sites not included:** LinkedIn, Naukri, Instahyre and Wellfound need a login or block automated access. Remotive asks API users to fetch at most 4 times a day, which doesn't fit an hourly check.

---

## 13. Before you push this repo

The exported workflow file contains **no passwords or API keys**. Those stay inside your n8n. It does contain:

- your email addresses (`emailTo`, `emailFrom`) and your `sheetId` in the **Settings** node,
- the names and internal IDs of your n8n credentials (harmless without your n8n).

For a public repo, replace the personal values in the JSON with placeholders before committing:

```js
emailTo: 'you@example.com',
emailFrom: 'sender@gmail.com',
sheetId: 'YOUR_GOOGLE_SHEET_ID',
```

**Never commit** the Google service-account `.json` key file, your Gemini key or your Gmail app password. Keep the key file outside the repo folder, or add it to `.gitignore`.

Suggested repo layout:

```
fresher-job-alerts/
├── README.md
├── Fresher-Job-Alerts.n8n.json
└── .gitignore
```
