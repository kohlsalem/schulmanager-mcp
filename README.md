# Schulmanager Online MCP Server

A local [MCP](https://modelcontextprotocol.io/) (Model Context Protocol) server for accessing [Schulmanager Online](https://schulmanager-online.de/) data. It gives LLMs such as Claude direct access to timetables, homework, exams, grades, parent letters and more.

> **Note:** This is an unofficial community project with no affiliation to Schulmanager Online GmbH. Use at your own risk.

> **About this fork:** Based on [kohlsalem/schulmanager-mcp](https://github.com/kohlsalem/schulmanager-mcp). Adds full letter content including PDF attachments, tracking of processed letters, an email summary tool, support for multiple accounts, and Windows setup instructions. See [What's new in this fork](#whats-new-in-this-fork).

## Features

| Tool | Description |
|------|-------------|
| `schulmanager_daily_report` | **Parent briefing**: complete overview per child — new messages, tomorrow's timetable, exams (7 days), homework (7 days) |
| `schulmanager_get_students` | List all children/students on the account |
| `schulmanager_get_schedule` | Timetable for a selectable period, including substitutions |
| `schulmanager_get_homework` | Current homework |
| `schulmanager_get_exams` | Upcoming exams and class tests |
| `schulmanager_get_grades` | Grades per subject (if enabled by the school) |
| `schulmanager_get_letters` | Parent letters and notifications (titles and dates) |
| `schulmanager_get_letter` | **New:** full content of one letter, including text extracted from PDF attachments |
| `schulmanager_get_new_letters` | **New:** letters not yet processed on this account |
| `schulmanager_mark_letters_processed` | **New:** mark letters as processed after a successful run |
| `schulmanager_send_summary_email` | **New:** send a summary email to preconfigured recipients via Gmail SMTP |
| `schulmanager_get_institution` | School information |
| `schulmanager_raw_call` | Arbitrary API call for endpoints not covered above |

## Requirements

- Python 3.11+
- A parent or student account on [Schulmanager Online](https://login.schulmanager-online.de/)

## Installation

### macOS / Linux

```bash
git clone https://github.com/<your-username>/schulmanager-mcp.git
cd schulmanager-mcp

python3 -m venv .venv
source .venv/bin/activate
pip install -e .
```

### Windows (cmd)

```
git clone https://github.com/<your-username>/schulmanager-mcp.git
cd /d C:\path\to\schulmanager-mcp

python -m venv .venv
.venv\Scripts\activate
pip install -e .
```

If `python --version` prints only `Python` without a version number, you are hitting the Microsoft Store stub. Install Python from [python.org](https://www.python.org/) (tick "Add python.exe to PATH") and disable the `python.exe` / `python3.exe` aliases under *Settings → Apps → Advanced app settings → App execution aliases*.

### Dependencies

`pyproject.toml` pins `mcp<2`: the server uses `FastMCP`, which was renamed in MCP SDK 2.x. Additional dependencies for this fork: `pypdf` (PDF text extraction) and `python-dotenv` (`.env` loading for the standalone report). If you install packages manually, install them into the virtual environment:

```
.venv/bin/python -m pip install pypdf python-dotenv "mcp<2"        # macOS / Linux
.venv\Scripts\python.exe -m pip install pypdf python-dotenv "mcp<2"  # Windows
```

## Configuration

### Environment variables

| Variable | Required | Description |
|----------|----------|-------------|
| `SCHULMANAGER_EMAIL` | yes | Schulmanager login |
| `SCHULMANAGER_PASSWORD` | yes | Schulmanager password |
| `SCHOOL_KEY` | for letter tracking | Short unique key per account, e.g. `school-a`. Used for the state file and IDs in calendar/task entries |
| `GMAIL_ADDRESS` | for email | Gmail account used as sender |
| `GMAIL_APP_PASSWORD` | for email | Gmail [app password](https://myaccount.google.com/apppasswords) (requires 2-step verification), without spaces |
| `SUMMARY_TO` | for email | Comma-separated recipients. Defaults to `GMAIL_ADDRESS` |

### Option A: `.env` file (single account, standalone report)

```bash
cp .env.example .env
```

Edit `.env`:

```
SCHULMANAGER_EMAIL=your-email@example.com
SCHULMANAGER_PASSWORD=your-password
```

### Option B: environment variables in the MCP client config (recommended)

#### Claude Desktop

Open *Settings → Developer → Edit Config* and add the server to `claude_desktop_config.json`. If the file already contains other keys (e.g. `preferences`), add `mcpServers` as another top-level key — the file must remain a single JSON object.

macOS / Linux:

```json
{
  "mcpServers": {
    "schulmanager": {
      "command": "/absolute/path/to/schulmanager-mcp/.venv/bin/python",
      "args": ["-m", "schulmanager_mcp.server"],
      "env": {
        "SCHULMANAGER_EMAIL": "your-email@example.com",
        "SCHULMANAGER_PASSWORD": "your-password"
      }
    }
  }
}
```

Windows (backslashes must be doubled in JSON):

```json
{
  "mcpServers": {
    "schulmanager": {
      "command": "C:\\path\\to\\schulmanager-mcp\\.venv\\Scripts\\python.exe",
      "args": ["-m", "schulmanager_mcp.server"],
      "env": {
        "SCHULMANAGER_EMAIL": "your-email@example.com",
        "SCHULMANAGER_PASSWORD": "your-password"
      }
    }
  }
}
```

Fully quit Claude Desktop (tray icon → Quit) and start it again. Claude Desktop also writes its own settings to this file, so edit it while the app is closed.

#### Claude Code

```bash
claude mcp add --env SCHULMANAGER_EMAIL=your-email@example.com \
  --env SCHULMANAGER_PASSWORD=your-password \
  schulmanager -- /absolute/path/to/schulmanager-mcp/.venv/bin/python -m schulmanager_mcp.server
```

### Multiple accounts (children at different schools)

Schulmanager allows only children from the same school in one account, so children at different schools need separate accounts. Run one server instance per account, each with its own credentials and `SCHOOL_KEY`.

If both accounts use the same email address, they must have **different passwords** — otherwise the login asks you to choose a school, which automated logins cannot do.

```json
{
  "mcpServers": {
    "school-a": {
      "command": "C:\\path\\to\\schulmanager-mcp\\.venv\\Scripts\\python.exe",
      "args": ["-m", "schulmanager_mcp.server"],
      "env": {
        "SCHULMANAGER_EMAIL": "parent@example.com",
        "SCHULMANAGER_PASSWORD": "password-a",
        "SCHOOL_KEY": "school-a",
        "GMAIL_ADDRESS": "sender@gmail.com",
        "GMAIL_APP_PASSWORD": "xxxxxxxxxxxxxxxx",
        "SUMMARY_TO": "parent1@example.com, parent2@example.com"
      }
    },
    "school-b": {
      "command": "C:\\path\\to\\schulmanager-mcp\\.venv\\Scripts\\python.exe",
      "args": ["-m", "schulmanager_mcp.server"],
      "env": {
        "SCHULMANAGER_EMAIL": "parent@example.com",
        "SCHULMANAGER_PASSWORD": "password-b",
        "SCHOOL_KEY": "school-b"
      }
    }
  }
}
```

Name each server after the child or school so the model can tell them apart. Email settings are only needed on the instance that sends the summary.

## Usage

Ask the model in any client where the server is connected:

```
> Give me the parent briefing
> What homework does my child have?
> Show me next week's timetable
> Which exams are coming up in the next 4 weeks?
> Are there new parent letters? Translate them and list all dates and to-dos.
> What's on the timetable tomorrow?
```

### Standalone daily report

The parent briefing can also be printed directly in the terminal, without MCP:

```bash
./daily_report.sh             # macOS / Linux
```

```
.venv\Scripts\python.exe daily_report.py    # Windows
```

`daily_report.py` reads credentials from `.env` (via `python-dotenv`) or from the environment. It prints a briefing per child: unread letters, tomorrow's timetable, exams and homework for the next 7 days.

## What's new in this fork

### Full letter content and PDF attachments

`schulmanager_get_letter(letter_id)` returns the title, dates, reply deadline, survey options and the full letter text (HTML converted to plain text). PDF attachments are downloaded and their text is appended under a heading per file. Scanned PDFs without a text layer are marked as such.

### Processed-letter tracking

`schulmanager_get_new_letters` returns only letters that have not been processed yet; `schulmanager_mark_letters_processed` records their IDs in `state/processed_<SCHOOL_KEY>.json`. Unlike a "last 24 hours" filter, nothing is lost when a scheduled run is skipped (e.g. the computer was off), and nothing is summarized twice.

Before the first scheduled run, mark old letters as processed once, otherwise the first summary will include the whole history:

```
> For each school: call schulmanager_get_new_letters and mark all letters
> older than 7 days as processed.
```

### Email summary

`schulmanager_send_summary_email(subject, body)` sends a plain-text email via Gmail SMTP (SSL, port 465). Recipients come from `SUMMARY_TO` in the server config and cannot be set by the model.

### Daily summary workflow

A typical scheduled run (e.g. a Claude Cowork scheduled task that requires the local computer, since the MCP server runs locally):

1. `schulmanager_get_new_letters` for each account
2. `schulmanager_get_letter` for each new letter (including PDFs)
3. Translate and summarize; extract events and to-dos and resolve relative deadlines ("by the next lesson") using `schulmanager_get_schedule`
4. Add events to a calendar and to-dos to a task list (e.g. via the Google Calendar and Todoist connectors), using IDs like `SM-ID: school-a-<letterId>-<n>` in descriptions to avoid duplicates
5. `schulmanager_send_summary_email`
6. `schulmanager_mark_letters_processed` — **only after** the email was sent, so nothing is lost if a run fails

#### Example prompt

[`daily_prompt.example.md`](daily_prompt.example.md) contains a complete prompt for this workflow. Copy it to `daily_prompt.md` (ignored by git), adjust server names, children's names, output language and calendar/task list names, and point your scheduled task at it:

```
Read the file C:\path\to\schulmanager-mcp\daily_prompt.md and follow the instructions in it.
```

Keeping the prompt in a file means you can change the workflow without editing the scheduled task.

#### Example output

A summary email produced by the example prompt (fictional data):

```
Subject: School summary for 06.10.

== CHILD A ==
New letters:
- Parents' council election: online vote until Fri 09.10, 13:00.
  QR code and password are on the paper slip brought home on 02.10.
- PE information: bring indoor sports shoes from next week.

== CHILD B ==
New letters: none

== HOMEWORK ==
Child A, for tomorrow:
- English: vocabulary p. 171 (noisy – holy)
- Music: learn the song lyrics
Child A, later this week:
- Thu 08.10 Maths: worksheet "Linear functions"
Child B: the school does not publish homework in Schulmanager.

== TO DO (overdue and next 7 days) ==
- [Child A] Vote in the parents' council election — by Fri 09.10, 13:00
- [Child A] Buy A4 ring binder and squared paper for Nutrition & Health
  — by Tue 07.10 (next Nutrition & Health lesson per timetable)

== EVENTS IN THE NEXT 7 DAYS ==
- Wed 07.10, 19:00 [Child B] Parents' evening, class 5b
- Fri 09.10 [Child A] Maths test
```

### Fixes

- Pinned `mcp<2` (FastMCP was removed in MCP SDK 2.x)
- Fixed a crash in the daily report when a lesson has `room: null` (e.g. PE lessons without a room)

## Technical details

### Authentication

The server uses the unofficial Schulmanager Online API:

1. **Get salt** via `POST /api/get-salt`
2. **Hash password** with PBKDF2-SHA512 (99,999 iterations, 512-byte output)
3. **Log in** via `POST /api/login` — returns a JWT
4. **Fetch data** via `POST /api/calls` with a Bearer token

### API endpoints

All data requests go through the central `/api/calls` endpoint as batch requests:

| Module | Endpoint | Description |
|--------|----------|-------------|
| `schedules` | `get-actual-lessons` | Timetable including substitutions |
| `classbook` | `get-homework` | Homework |
| `exams` | `get-exams` | Exams and class tests |
| `grades` | `get-grades` | Grades |
| `letters` | `get-letters` | Parent letters (metadata only) |
| `letters` | `poqa` | Letter content: ORM-style query, `action: findByPk` on model `modules/letters/letter`, with `include: attachments` |
| `main` | `get-institution` | School information |

Not every school enables every module. If a school does not use homework in Schulmanager, `get-homework` may return HTTP 500 instead of an empty list.

### Attachments

The attachment's `file` field is a JSON string (`[institutionId, module, hash, key, contentType, size, filename]`). The download URL is its base64 encoding:

```
GET https://login.schulmanager-online.de/download-file/<base64(file)>
```

No authorization header is required.

The `schulmanager_raw_call` tool can be used to call any other endpoint.

## Security notes

- Email recipients are fixed in the server config. The model cannot send mail to other addresses, even if a letter contains such instructions.
- Use a dedicated Gmail (and calendar/task) account and an app password rather than your main account and password.
- Credentials live in the MCP client config or in `.env`. Never commit them. `.env`, `state/` and your personal daily prompt should be in `.gitignore`.
- Letter contents are untrusted input. When building automations, instruct the model to treat them as data, not instructions.

## Acknowledgements

- [kohlsalem/schulmanager-mcp](https://github.com/kohlsalem/schulmanager-mcp) — the original MCP server this fork is based on

The API structure was derived from existing open-source projects, in particular:

- [rwunsch/schulmanager-online-hass](https://github.com/rwunsch/schulmanager-online-hass) — Home Assistant integration
- [SchmueI/Schulmanager-API](https://github.com/SchmueI/Schulmanager-API) — Python scraping client

## License

This project is released under the [Unlicense](LICENSE) — public domain, without any warranty.
