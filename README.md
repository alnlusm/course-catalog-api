# Course Catalog API

Backend for the course catalog on FastAPI: the same six courses as in the Next.js project, described with a Pydantic model.

## How to run

```bash
python -m venv .venv
# Windows (PowerShell): .venv\Scripts\Activate.ps1
# macOS / Linux:        source .venv/bin/activate
pip install -r requirements.txt
fastapi dev main.py
```

The server starts at http://127.0.0.1:8000, interactive docs are at http://127.0.0.1:8000/docs.

## Project structure

| File | Purpose |
| --- | --- |
| `models.py` | Pydantic model `Course` |
| `data.py` | the six courses, `get_all_courses()` and `find_course()` |
| `main.py` | the FastAPI app, routes and the `pagination` dependency |

## Routes

| Route | Description |
| --- | --- |
| `GET /` | health check |
| `GET /courses` | course list; query parameters `is_elective`, `sort` (`popular` / `title`), `page`, `page_size` |
| `GET /courses/{course_id}` | one course, or 404 `{"detail": "Course not found"}` |

## What was verified

| Request | Result |
| --- | --- |
| `GET /` | `{"message": "Course Catalog API is running"}` |
| `GET /courses` | 6 courses: ai-integration, modern-frontend, web-security, backend-fastapi, databases-postgresql, api-design |
| `GET /courses?is_elective=true` | 2 courses: ai-integration, api-design |
| `GET /courses?is_elective=false` | 4 courses: modern-frontend, web-security, backend-fastapi, databases-postgresql |
| `GET /courses?sort=title` | ai-integration, api-design, backend-fastapi, modern-frontend, databases-postgresql, web-security |
| `GET /courses?page=1&page_size=2` | ai-integration, modern-frontend |
| `GET /courses?page=2&page_size=2` | web-security, backend-fastapi |
| `GET /courses?page=3&page_size=2` | databases-postgresql, api-design |
| `GET /courses?page=4&page_size=2` | `[]` |
| `GET /courses/web-security` | the Web Security Essentials course |
| `GET /courses/nope` | 404, `{"detail": "Course not found"}` |
| `/courses/web-security` with `course_id: int` (temporary) | 422, `Input should be a valid integer` |
| `/docs` | exactly three routes, `Course` in Schemas |
