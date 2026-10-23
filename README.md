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
| `models.py` | `CamelModel`, `Course`, `EnrollmentIn`, `Enrollment` |
| `data.py` | the six courses, `get_all_courses()` and `find_course()` |
| `main.py` | the FastAPI app, routes and the `pagination` dependency |

## Routes

| Route | Description |
| --- | --- |
| `GET /` | health check |
| `GET /courses` | course list; query parameters `is_elective`, `sort` (`popular` / `title`), `page`, `page_size` |
| `GET /courses/{course_id}` | one course, or 404 `{"detail": "Course not found"}` |
| `POST /courses/{course_id}/like` | +1 like, returns the updated course |
| `POST /enrollments` | enroll a student, 201 |
| `GET /enrollments` | all enrollments |

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

## What changed (Lab 5)

**CORS.** `CORSMiddleware` allows exactly one origin: `http://localhost:3000` (scheme, host and port, no path, no trailing slash). The Next.js pages call the API from the browser (the filter, the like button, the enrollment form), and the browser only lets JavaScript read a response from another origin when the server sends `access-control-allow-origin` for that origin. A request with `Origin: http://evil.example` still gets 200, but without `access-control-allow-origin`.

**camelCase in JSON.** All models inherit from `CamelModel` (`alias_generator=to_camel`, `validate_by_name=True`, `validate_by_alias=True`). Python code keeps `is_elective`, JSON has `isElective`, matching the TypeScript `Course` type. Query parameters are not affected: the filter is still `?is_elective=true`.

**New routes** (likes and enrollments live in memory and reset when the server restarts):

| Request | Response |
| --- | --- |
| `POST /courses/web-security/like` | 200, the course with `"likes": 22` (was 21) |
| `POST /courses/nope/like` | 404, `{"detail": "Course not found"}` |
| `POST /enrollments` `{"courseId": "web-security", "studentName": "Aigerim"}` | 201, `{"courseId": "web-security", "studentName": "Aigerim", "id": 1}` |
| `POST /enrollments` with `"studentName": "A"` | 422, `loc: ["body", "studentName"]`, `String should have at least 2 characters` |
| `POST /enrollments` with `"courseId": "nope"` | 404, `Course not found` |
| `GET /enrollments` | `[{"courseId": "web-security", "studentName": "Aigerim", "id": 1}]` |

Preflight check (`OPTIONS /enrollments` with `Origin: http://localhost:3000`, `Access-Control-Request-Method: POST`, `Access-Control-Request-Headers: content-type`): 200, `access-control-allow-origin: http://localhost:3000`, `access-control-allow-headers: content-type`, `access-control-max-age: 600`.
