# Towson Study Planner

A Streamlit app for turning course deadlines, workload, priorities, and weekly availability into a practical study schedule. The interface uses Towson-inspired black and gold, supports English and Spanish, and includes adjustable text sizing. Individual student accounts keep planner data separate. AI features use a server-side Google AI key. Students do not enter or store provider keys. API usage and limits are shared across users of the configured provider account.

Students enter their major or intended program when creating an account and can set their academic classification and degree type in **Degree tracker**. The welcome page offers curriculum planning chat before any courses are added. The chat retrieves relevant excerpts from the selected catalog and can use saved courses, degree entries, syllabi, and assignments to recommend future-semester paths appropriate to the student's classification and program.

## Run locally

```bash
python -m pip install -r requirements.txt
cp .streamlit/secrets.toml.example .streamlit/secrets.toml
streamlit run streamlit_app.py
```

Before starting locally, add a Google AI API key to `.streamlit/secrets.toml` as `GOOGLE_API_KEY`. Streamlit deployments can set this key in app secrets; the `GOOGLE_API_KEY` or `GEMINI_API_KEY` server environment variable is also supported. `GEMINI_API_KEY` remains an alternative name for `GOOGLE_API_KEY`. Never commit or expose API keys in source code or client-facing UI.

For local development, copy `.streamlit/secrets.toml.example` to `.streamlit/secrets.toml` and replace the placeholder value. The real secrets file is ignored by Git. Revoke and replace any key that has been shared in chat or committed.

The app creates a SQLite database at `~/.student_ai_study_planner/planner.sqlite3`. Set `STUDY_PLANNER_DB` to a path on persistent, private storage when deploying. Usernames and salted password hashes are stored there; syllabus text, course records, and chat history are stored per account. The database file is restricted to the app's operating-system user but is not encrypted, so protect and back up the storage appropriately.

The campus illustration is served locally from `static/`, so the page background does not depend on an external image host. Streamlit static-file serving is enabled in `.streamlit/config.toml`.

Place the authorized Towson curriculum text files `Undergraduate.txt` and `Graduated.txt` in `catalogs/`; matching undergraduate/graduate PDFs are also supported. Text files are split into searchable excerpts and cached in memory; PDF excerpts retain page numbers for citations. Catalog files are excluded from Git and must be provided separately in each deployment. The selected degree type chooses which catalog supplies recommendations. Relevant excerpts, the student's major, classification, degree type, and included course context are sent to Google AI when the student submits a curriculum chat question. Scheduling questions also include the student's available study hours, open tasks, and generated study sessions. Without a matching catalog, the chat must say it cannot verify specific degree requirements.

Students create an account from the login screen by entering a username, major or intended program, and password. Passwords must be at least 8 characters. Existing accounts without a saved major can add one from the welcome page. Use a private, persistent database volume for deployments; each app instance must point to the same database if accounts are shared across instances.

## Workflow

1. Add courses and set their typical difficulty and personal importance.
2. Add assignments, exams, and milestones with deadlines and estimated hours, or upload a PDF, Markdown, or text syllabus. Save one syllabus per class in **Courses** so course chat can use it; replace that saved file whenever the class syllabus changes. The Courses page lists every class and keeps its saved extraction—overview, weekly topics, grading, and suggested tasks—together in an expandable class card. Google AI analyzes the syllabus; choose the course first, then review the proposed tasks on the **Review syllabus results** screen and accept them to add them to your planner. PDF extraction reads selectable text; scanned PDFs need OCR first.
3. Set available study hours for each weekday and a preferred time of day.
4. Review the priority queue and the rolling 14-day schedule. Each scheduled block is up to one hour and names a concrete activity.
5. Update completed hours, task status, deadlines, or availability. The schedule is recalculated from the current data on each app update.
6. Use **Degree tracker** to set a target graduation year, enter completed courses and grades, and plan future courses by term. Review completed/planned credits, edit the transcript, and import or download it as CSV.

The transcript and syllabus data are private to the signed-in account in the app's SQLite database. Download the transcript CSV as an additional backup.

## Prioritization

Tasks are ranked using a 0-100 score: 45% deadline urgency, 35% importance, and 20% difficulty. Urgency is calculated as `1 / (days until deadline + 1)`, so due-today and overdue work rises to the top. The schedule fits work into the available daily capacity, limits each task to one session per day, and reports work that does not fit in the next 14 days.

Syllabus review sends up to 30,000 characters of extracted document text to Google's `gemini-3.5-flash-lite` model using the shared server-side API key. A single request times out after 75 seconds so failed analyses return well before two minutes. Replacing a class's saved syllabus updates course-chat context and clears the previous analysis; review it again to refresh topics, grading, and task suggestions.

The **Evaluation** view runs overlapping-deadline and limited-availability scenarios through the same scheduler. It reports work completed by deadlines, tasks schedulable within the planning window, schedule conflicts, daily workload balance, and checks urgency ordering and completion-based replanning.

The sidebar **Menu** dropdown switches between the planner pages. The **Course and curriculum planning chat** is available on the welcome page before any courses are added and remains available from the menu. It uses Google AI and streams answers. Direct catalog lookups retrieve fewer passages; longer planning questions include more context. Catalog passages are selected using the student's major, classification, degree type, and question, and include the catalog name and page when available. Scheduling questions include the student's availability, open tasks, and generated study sessions. Chat history is stored in the student's account database. Uploaded syllabi can provide additional course-specific context. The app does not send transcript grades; it includes course codes, titles, terms, and statuses from degree entries to help plan. AI suggestions should be checked against the original catalog and syllabus. Syllabus analysis and the planner help chat also use Google AI.