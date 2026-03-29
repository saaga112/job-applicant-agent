# Multi-Agent Automated Job Application System

A fully automated job application pipeline built with **CrewAI** that parses your resume, searches for jobs across multiple platforms, tailors your resume and cover letter for each position, and submits applications automatically.

## Architecture

```
┌─────────────── RESEARCH CREW (Sequential) ───────────────┐
│                                                           │
│  Resume Parser ──► Job Searcher ──► Job Matcher           │
│  (PDF/DOCX)       (4 platforms)    (score & rank)         │
│                                                           │
└───────────────────────┬───────────────────────────────────┘
                        │ matched jobs
                        ▼
┌─────────────── APPLICATION CREW (per job) ───────────────┐
│                                                           │
│  Resume Tailor ──► Cover Letter Writer ──► Job Applicant  │
│  (ATS keywords)    (personalized)          (auto-submit)  │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### 6 Agents

| Agent | What it does |
|-------|-------------|
| **Resume Parser** | Extracts structured profile from your PDF/DOCX resume |
| **Job Searcher** | Scrapes LinkedIn, Indeed, Glassdoor, ZipRecruiter via Playwright |
| **Job Matcher** | Scores candidate-job fit (0.0–1.0), ranks, recommends Apply/Maybe/Skip |
| **Resume Tailor** | Rewrites your resume with ATS-optimized keywords per job |
| **Cover Letter Writer** | Generates a personalized cover letter per job |
| **Job Applicant** | Fills and submits application forms via browser automation |

### Supported Platforms

- LinkedIn (Easy Apply + standard)
- Indeed
- Glassdoor
- ZipRecruiter
- Greenhouse ATS
- Generic application forms (auto-detected)

## Prerequisites

- **Python 3.12+**
- **uv** package manager ([install guide](https://docs.astral.sh/uv/getting-started/installation/))
- **OpenAI API key** (used by CrewAI agents)

## Setup

### 1. Clone the repo

```bash
git clone https://github.com/saaga112/job-applicant-agent.git
cd job-applicant-agent
```

### 2. Install dependencies

```bash
uv sync
```

This installs all 160+ packages (CrewAI, Gradio, Playwright, OpenAI, etc.) into an isolated `.venv`.

### 3. Install browser for Playwright

```bash
uv run playwright install chromium
```

### 4. Set your OpenAI API key

```bash
export OPENAI_API_KEY="sk-your-key-here"
```

Or create a `.env` file in the project root:

```
OPENAI_API_KEY=sk-your-key-here
```

### 5. Add your resume

Place your resume (PDF or DOCX) in the `resumes/` folder:

```bash
mkdir -p resumes
cp /path/to/your/resume.pdf resumes/
```

## Usage

### Option 1: Gradio Web UI (Recommended)

```bash
cd src
uv run python -m job_applicant.app
```

Open **http://localhost:7860** in your browser.

**4 tabs:**

| Tab | Purpose |
|-----|---------|
| **Setup** | Upload resume, enter search keywords, pick locations & platforms, set minimum match score |
| **Job Matches** | View ranked jobs with fit scores, matching/missing skills |
| **Applications** | Track status — submitted, failed, or blocked (with screenshots) |
| **Logs** | Real-time agent activity output |

### Option 2: Command Line

```bash
cd src

# Basic — uses default keywords
uv run python -m job_applicant.main ../resumes/resume.pdf

# Custom keywords and locations
uv run python -m job_applicant.main ../resumes/resume.pdf \
  "python developer,backend engineer,data engineer" \
  "Remote,San Francisco CA,New York NY"
```

## What Happens When You Run It

1. **Resume Parser** reads your PDF/DOCX and extracts a structured profile (skills, experience, education)
2. **Job Searcher** scrapes all selected platforms with your keywords and locations
3. **Job Matcher** scores every job against your profile and filters by your minimum score
4. For each matched job:
   - **Resume Tailor** rewrites your resume to emphasize relevant skills and ATS keywords
   - **Cover Letter Writer** generates a personalized cover letter
   - **Job Applicant** navigates to the application page and submits via browser automation
5. All results are saved to the `output/` folder

## Output Files

After a run, check the `output/` directory:

```
output/
├── candidate_profile.json          # Your parsed resume (structured)
├── matched_jobs.json               # All matched jobs with scores
├── Google_SWE_resume.md            # Tailored resume (one per job)
├── Google_SWE_cover_letter.md      # Cover letter (one per job)
├── screenshots/                    # Proof-of-submission screenshots
│   ├── applied_linkedin_1234.png
│   ├── blocked_captcha_5678.png
│   └── ...
└── applications.db                 # SQLite database tracking all applications
```

## Project Structure

```
job-applicant-agent/
├── src/job_applicant/
│   ├── app.py                  # Gradio UI (4-tab dashboard)
│   ├── main.py                 # CLI pipeline orchestrator
│   ├── models.py               # Pydantic data models
│   ├── state.py                # SQLite state tracker (deduplication)
│   ├── config/
│   │   ├── agents.yaml         # 6 agent role/goal/backstory configs
│   │   └── tasks.yaml          # 6 task descriptions and workflows
│   ├── crews/
│   │   ├── research_crew.py    # Research crew (parse → search → match)
│   │   └── application_crew.py # Application crew (tailor → letter → apply)
│   └── tools/
│       ├── resume_parser_tool.py   # PDF/DOCX text extraction
│       ├── job_search_tool.py      # Playwright multi-platform scraper
│       ├── job_apply_tool.py       # Playwright form filler + submitter
│       └── file_writer_tool.py     # Save generated documents
├── resumes/                    # Your resume goes here
├── output/                     # Generated files land here
├── pyproject.toml              # Dependencies
├── .python-version             # Python 3.12
└── .gitignore
```

## Edge Cases & Safety

| Scenario | How it's handled |
|----------|-----------------|
| **CAPTCHA detected** | Marks application as BLOCKED, takes screenshot, logs for manual follow-up |
| **Login required** | Detects login walls, marks as BLOCKED with platform name |
| **Rate limiting** | Randomized delays (2–5s), max 5 pages per platform, exponential backoff on 429 |
| **Duplicate jobs** | URL hash deduplication in SQLite — never applies to the same job twice |
| **Resume parse failure** | Tries pypdf → falls back to raw text → clear error message |
| **Network failure** | 3 retries with backoff, state saved before each step |
| **Unknown form layout** | Generic handler auto-detects name/email/phone/resume/cover letter fields |

## Configuration

### Adjusting agents

Edit `src/job_applicant/config/agents.yaml` to change agent roles, goals, or LLM models:

```yaml
resume_tailor:
  role: Professional Resume Writer
  goal: Rewrite resume to maximize ATS score...
  llm: openai/gpt-4o  # Change to gpt-4o-mini for lower cost
```

### Adjusting tasks

Edit `src/job_applicant/config/tasks.yaml` to modify task descriptions, expected outputs, or agent assignments.

### Minimum match score

Default is `0.5` (applies to jobs scoring 50%+ match). Adjust in the Gradio UI slider or via CLI:

```python
# In main.py or when calling run()
run(resume_path="...", keywords=[...], min_match_score=0.7)  # Only apply to 70%+ matches
```

## Tech Stack

- **[CrewAI](https://github.com/crewAIInc/crewAI)** — Multi-agent orchestration
- **[Playwright](https://playwright.dev/python/)** — Browser automation for scraping and form filling
- **[Gradio](https://gradio.app/)** — Web UI dashboard
- **[OpenAI GPT-4o](https://openai.com/)** — LLM powering the agents
- **[Pydantic](https://docs.pydantic.dev/)** — Data validation and structured outputs
- **[pypdf](https://pypdf.readthedocs.io/)** / **[python-docx](https://python-docx.readthedocs.io/)** — Resume parsing
- **SQLite** — Application state tracking and deduplication

## License

MIT
