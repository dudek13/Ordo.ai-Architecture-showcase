# Ordo.ai-Architecture-showcase

> **Note:** The source code for this project is maintained in a private repository due to active commercialization efforts. This repository serves strictly as an Architectural Showcase demonstrating the system design, data flow, and technology stack.

**Ordo.ai** is a hybrid, high-performance system designed to assist managers in automating employee work schedules. It combines the flexible natural language understanding of Large Language Models (acting as an AI Preprocessor) with an ultra-fast, 100% deterministic algorithmic engine written in Python (**Solver**) and a transactional PostgreSQL database.

---

## Key Features & Solved Problems

- **Zero Hallucinations & Full Determinism**: The language model does not generate the schedule directly. Its sole responsibility is extracting intent from raw text. The actual schedule is computed by a mathematical-logical Python Solver in `< 0.1s`.
- **Labor Law & Business Rule Compliance**:
  * Fatigue prevention (maximum of 5 consecutive working days, followed by a mandatory rest day).
  * Automatic exclusion of non-trading Sundays (assigned the `'W'` off-day symbol).
  * Integration of approved leaves stored in the database (assigned the `'U'` vacation symbol).
  * Guaranteed shift staffing levels (e.g., exactly 2 employees for the morning shift **R** and 2 employees for the afternoon shift **P**).
- **Fair Workload Distribution**: The algorithm dynamically tracks the total number of days worked per month and prioritizes assigning shifts to employees with the lowest current workload.
- **Natural Language Processing (AI Preprocessor)**: Managers can enter free-form notes in plain language (e.g., *"Iwona is on vacation from October 10-15, Agnieszka needs the morning shift on Friday"*), which the AI translates into structured JSON rules.
- **End-to-End Data Lifecycle**: From entering natural language notes in the Streamlit frontend, through Pydantic validation in FastAPI and atomic transactions in PostgreSQL, all the way to generating an official, print-ready PDF schedule.

---

## Technology Stack

| Layer | Technology / Library | Role in the System |
| :--- | :--- | :--- |
| **Frontend** | `Streamlit`, `Pandas` | User interface, natural language input, tabular schedule preview, staffing statistics |
| **Backend API** | `FastAPI`, `Uvicorn`, `Pydantic` | Asynchronous REST API server, data & type validation, workflow orchestration |
| **AI Preprocessor** | `Groq API` (`llama-3.3-70b-versatile`) | Extracting manager intent from natural language into clean, structured JSON |
| **Engine / Solver** | `Python 3.12` (Custom Constraint Engine) | Deterministic schedule generation, enforcing labor regulations and staffing targets |
| **Database** | `PostgreSQL` / `Supabase`, `psycopg2` | Persistent storage for employees, roles, vacations, and saved shifts |
| **PDF Reporting** | `Jinja2`, `WeasyPrint`, `HTML5/CSS3` | HTML template rendering and compilation into an official print-ready PDF document |

---

## System Data Flow

```text
[USER / STREAMLIT (frontend.py)]
       │
       ├─► 1. Enter natural language notes + select Year/Month
       │     │
       │     └─► POST /generate_schedule/ (FastAPI main.py)
       │           │
       │           ├─► A. Call ai_agent.py (preprocess_input)
       │           │     └─► Groq API (Llama-3.3-70B) ──► Returns clean JSON
       │           │
       │           └─► B. Call solver.py (generate_schedule)
       │                 ├─► Fetches employees and vacations from DB (database.py)
       │                 ├─► Applies hard constraints from AI (forced_shifts)
       │                 └─► Generates schedule in < 0.1s and returns dictionary
       │
       ├─► 2. Preview DataFrame table + Staffing statistics on screen
       │
       ├─► 3. Click "💾 Save Schedule to Database"
       │     │
       │     └─► POST /save_schedule/ (FastAPI main.py)
       │           └─► Maps shift symbols ('R' -> 06:00-14:00, 'P' -> 14:00-22:00)
       │               and commits via an atomic transaction to PostgreSQL
       │
       └─► 4. Click "📄 Download PDF Schedule"
             │
             └─► GET /raporty/harmonogram/{year}/{month} (FastAPI main.py)
                   └─► Fetches saved shifts from DB ──► Jinja2 Template ──► WeasyPrint ──► PDF File
```

---

## Modular Architecture Breakdown

### 1. `frontend.py` (Streamlit Interface)
Handles interaction with the manager. It features a sidebar for year/month selection, an `st.text_area` input field for custom scheduling notes, and action buttons. It renders the generated schedule as a `pandas.DataFrame` table along with an automatic summary count of shift types (`R` – Morning, `P` – Afternoon, `W` – Day Off, `U` – Vacation).

### 2. `main.py` & `schemas.py` (FastAPI + Pydantic)
REST API server exposing secure, type-validated endpoints:
- **`POST /generate_schedule/`**: Accepts a `ScheduleRequest(year, month, instructions)` payload. If custom instructions are provided, it triggers the AI preprocessor first and passes the resulting constraints to the Solver.
- **`POST /save_schedule/`**: Accepts the generated schedule, maps shift symbols to exact timestamps (start and end times), and performs a bulk insert into the `zmiana` (shift) table with automatic transaction `rollback` on error.
- **`GET /raporty/harmonogram/{year}/{month}`**: Retrieves persisted shifts from the database, calculates total monthly hours per employee, populates an HTML template, and compiles it via `WeasyPrint` into a downloadable PDF file.

### 3. `ai_agent.py` & `prompts.py` (AI Preprocessor)
A lightweight module responsible for interpreting everyday language. It uses a strict system prompt defined in `prompts.py` that forces the Llama-3.3-70B model to return exclusively a valid JSON object containing two arrays:
- `urlopy` (vacations): a list of objects containing the employee's full name and date range (`data_start`, `data_koniec`).
- `sztywne_zmiany` (fixed shifts): a list of objects containing the employee's name, specific date, and forced shift symbol/time.

### 4. `solver.py` (Deterministic Scheduling Engine)
The core scheduling algorithm of the application, operating on a three-stage logic:
1. **State Initialization (`build_employee_tracker`)**: Fetches employee and vacation records from the database, building local state trackers for total days worked (`days_working`) and consecutive workdays (`workdays_in_a_row`).
2. **Constraint Parsing (`forced_shifts`)**: Converts AI-extracted rules into hard constraints within the `forced_shifts[(emp_id, date)] = symbol` dictionary.
3. **Day-by-Day Allocation Loop**:
   * *Step A*: Applies hard rules (`forced_shifts`, non-trading Sundays, database vacations, and the max 5 consecutive workdays limit).
   * *Step B*: Counts the staffing already allocated for a given day (how many employees are already assigned to morning `'R'` or afternoon `'P'` shifts).
   * *Step C*: Sorts the remaining available employees by their total days worked so far (ensuring fair workload distribution) and fills the remaining shift slots.

### 5. `database.py` (PostgreSQL / Supabase)
Database connectivity module built with `psycopg2` and environment variables loaded securely from `.env`. Ensures transactional integrity and executes parameterized SQL queries to prevent SQL Injection.

---

## Quick Start Guide

### 1. Prerequisites
- Python 3.12+
- Installed dependencies from `requirements.txt`:
  ```bash
  pip install fastapi uvicorn streamlit psycopg2-binary pandas requests pydantic weasyprint jinja2 groq python-dotenv
  ```
- A running PostgreSQL instance (e.g., on Supabase) with the required tables: `pracownik` (employee), `rola` (role), `urlop` (vacation), `zmiana` (shift).
- A Groq API key and database credentials configured in a `.env` file:
  ```env
  GROQ_API_KEY=your_groq_api_key
  DB_HOST=your_postgres_host
  DB_PORT=5432
  DB_NAME=postgres
  DB_USER=postgres
  DB_PASSWORD=your_password
  ```

### 2. Run the API Server (FastAPI)
In your first terminal window, start the Uvicorn server:
```bash
uvicorn main:app --reload
```
The API will be available at: `http://127.0.0.1:8000` (Swagger UI documentation: `http://127.0.0.1:8000/docs`).

### 3. Run the Frontend (Streamlit)
In your second terminal window, launch the Streamlit application:
```bash
streamlit run frontend.py
```
The application will open automatically in your browser at `http://localhost:8501`.

---

## Example Usage Scenario

1. **Open the Streamlit interface** in your browser.
2. In the sidebar, select **Year: 2026** and **Month: 10**.
3. In the *Custom Manager Notes* field, enter:
   > *"Iwona is on vacation from October 12 to October 15. Agnieszka must work the morning shift on Friday 16.10."*
4. Click **`🚀 Generate Schedule (Python Solver)`**.
   * *Behind the scenes*: FastAPI receives the request, the AI translates the text into JSON constraints, and the Solver builds a complete monthly schedule in 0.05 seconds—respecting the requested vacations, Agnieszka's morning shift, non-trading Sundays, and the 5-consecutive-workdays limit.
5. Click **`💾 Save Schedule to PostgreSQL`**.
   * *Behind the scenes*: The `'R'` and `'P'` shift symbols are converted into exact timestamps (`06:00-14:00` and `14:00-22:00`) and safely committed to the `zmiana` table.
6. Click **`📄 Download PDF Schedule`**.
   * *Behind the scenes*: A cleanly formatted, print-ready PDF report is generated and downloaded, ready to be posted on the staff noticeboard.
