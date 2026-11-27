# course-catalog-api

Backend of the course catalog: FastAPI, SQLAlchemy 2.1, PostgreSQL, Alembic and
GraphQL on Strawberry as a second entrance (Labs 4–8, "Advanced Web Technologies").

## How to run

You need Python 3.11 or newer and PostgreSQL 16 or newer.

Once, as the `postgres` superuser, create the role and the database:

```sql
CREATE ROLE catalog_app LOGIN PASSWORD 'change-me';
CREATE DATABASE course_catalog OWNER catalog_app;
```

Then, from the project folder:

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\Activate.ps1
pip install -r requirements.txt
cp .env.example .env               # Windows: copy .env.example .env
alembic upgrade head
psql -h localhost -U catalog_app -d course_catalog -v ON_ERROR_STOP=1 -f seed.sql
fastapi dev main.py
```

With the server running, the checks are run in a second window:

```bash
python try_errors.py      # 5 of 5 as expected
python try_graphql.py     # 8 of 8 as expected
python try_n_plus_1.py    # 3 and 3 SQL queries
python measure.py         # REST 5 requests / GraphQL 1 request
```

REST docs: http://localhost:8000/docs. GraphiQL: http://localhost:8000/graphql.

The password `change-me` is for training; real projects don't commit passwords.

## What changed in the API

- **REST errors are Problem Details.** The rules and their errors live in
  `services.py`: it commits, catches `IntegrityError`, looks up the constraint name
  and raises `ServiceError` with a code (`EMAIL_TAKEN`, `ALREADY_ENROLLED`,
  `COURSE_NOT_FOUND`, `STUDENT_NOT_FOUND`). `problem.py` turns the code into an
  answer with the content type `application/problem+json` and the fields `type`,
  `title`, `status`, `detail`, `instance`. The `type` field names the kind of error
  (`https://example.com/problems/already-enrolled`), so the client can switch on it.
  Starlette errors (unknown address, wrong method) use `about:blank`. `main.py`
  no longer knows about `HTTPException` or `IntegrityError`.
- **`/graphql`.** A Strawberry router next to REST, with the same database session
  per request (taken through `Depends(get_db)` in the context). Types `Course`,
  `Student`, `Enrollment`; queries `courses`, `course`, `students`, `student`,
  `enrollments`. A GET from a browser opens GraphiQL — an editor with hints and
  schema docs built from introspection; an ordinary client sends POST with JSON
  `{"query", "variables"}` and gets JSON back.
- **Mutations and `userErrors`.** `enroll(input: EnrollInput!)` and
  `likeCourse(id: ID!)` return a payload with the object and a `userErrors` list.
  An expected rule error (already enrolled, unknown course or student) comes in
  `data` with the same code as in `services.py`. Everything the client didn't
  foresee — a typo in the query, a bad argument, a server failure — comes in
  `errors`. Both cases have HTTP status 200.
- **`X-SQL-Queries` header.** `counter.py` counts SQL queries of each HTTP request
  (SQLAlchemy `before_cursor_execute` event + a `ContextVar`) and writes the number
  into a response header. It is how we see N+1 without reading the SQL log.
- **Protection.** Depth no more than 5 (`QueryDepthLimiter`), no more than 10
  aliases (`MaxAliasesLimiter`), `limit` must be 1..100 (`check_limit`), `id` must
  be a number (`parse_id`), the text of unexpected errors is hidden behind
  `Unexpected error.` (`MaskErrors`). Introspection is on by default and is turned
  off with `GRAPHQL_INTROSPECTION=false` in `.env` (restart the server after
  changing it).
- **New REST routes.** `GET /students/{id}` (404 `student-not-found` if missing) and
  `GET /students/{id}/enrollments`, used by the REST version of "My courses".

## Example queries

The "My courses" query (in the GraphiQL Variables panel: `{"id": "2"}`):

```graphql
query MyCourses($id: ID!) {
  student(id: $id) {
    name
    enrollments {
      id
      course { title credits level }
    }
  }
}
```

The `enroll` mutation (Variables:
`{"input": {"courseId": "web-security", "studentId": "3"}}`):

```graphql
mutation Enroll($input: EnrollInput!) {
  enroll(input: $input) {
    enrollment { id enrolledAt course { title } }
    userErrors { code message }
  }
}
```

The first time it succeeds:

```json
{"data": {"enroll": {"enrollment": {"id": "10", "enrolledAt": "2026-11-20T11:40:16.422855+05:00",
  "course": {"title": "Web Security Essentials"}}, "userErrors": []}}}
```

The same mutation again — a rule error inside `data`, status 200:

```json
{
  "data": {
    "enroll": {
      "enrollment": null,
      "userErrors": [
        {"code": "ALREADY_ENROLLED", "message": "Student is already enrolled in this course"}
      ]
    }
  }
}
```

## REST vs GraphQL comparison

The "My courses" screen of student 2; numbers from `measure.py`, starting data.

| Criterion | REST | GraphQL | Source |
|---|---|---|---|
| Requests and rounds per screen | 5 requests, 2 rounds (student + enrollments, then 3 courses) | 1 request, 1 round | `measure.py`, step 8 |
| Bytes in the responses | 1164 (full courses, names, dates) | 344 (only the fields asked for) | `measure.py`, step 8 |
| SQL queries per screen | 5 (1 per request) | 3 (with DataLoaders) | `measure.py`, `X-SQL-Queries` |
| SQL growth with more students | constant per route: the shape is known, one JOIN (`GET /enrollments` = 1) | without loaders 7 → 27 and 7 → 24 (3 → 20 students); with loaders 3 and 3 on both | `try_n_plus_1.py`, experiments 4–5 |
| Error in the query text | 404 for an unknown address, 422 for invalid parameters | 200, `data: null`, the message in `errors` with "Did you mean" | experiments 1 and 8 |
| Expected rule error (repeated enrollment) | 409, `application/problem+json`, type `.../already-enrolled` | 200, `userErrors[0].code = ALREADY_ENROLLED` | steps 1 and 4 |
| HTTP cache | GET by URL: browser, proxy and CDN can cache, ETag/Cache-Control work | POST to one URL: HTTP caches don't help; needs a client cache or persisted queries over GET | lecture section 14 |
| Protection from "expensive" queries | the route fixes the query and its limit; rate limit counts requests | the client builds the query: needs depth and alias limits, argument checks, error masking; 12 likes fit into one POST | experiments 6 and 7 |

**Conclusion.** For a screen that collects related data, like "My courses", GraphQL
is better: 1 request instead of 5, one round instead of two, 344 bytes instead of
1164, and with DataLoaders the SQL count stays 3 no matter how many students there
are. Public simple lists like `/courses` I would leave on REST: they are one SQL
query, they can be cached by URL, and errors come with honest HTTP statuses.
GraphQL also costs more to make safe: without protection one POST did 12 writes
and leaked SQL text to the client, so it only makes sense with the limits from
step 7.

## Experiment log

| # | Experiment | What I saw | Cause | How I fixed it |
|---|---|---|---|---|
| 1 | a typo in a field name, an argument name in `snake_case` | `titel`: status 200, `data: null`, `Cannot query field 'titel' on type 'Course'. Did you mean 'title', 'level', or 'likes'?`. `is_elective`: 200, `Unknown argument 'is_elective' on field 'Query.courses'. Did you mean 'isElective'?` | the engine validates the query against the schema before running it; Strawberry turns snake_case into camelCase in the schema. GraphQL reports errors in the body, not in the status | wrote `title` and `isElective`; the client must check `errors`, not the status |
| 2 | `\| None` removed from the `course` query, a nonexistent course | schema became `course(id: ID!): Course!`; `{ course(id: "nope") }` → `data: null`, `Cannot return null for non-nullable field Query.course.`; with aliases `a` (exists) and `b` (nope) the whole `data` was null, `a` was lost too | null propagation: a null in a non-null field goes up to the nearest nullable parent, here the whole `data` | put `\| None` back: `{"a": {"title": "API Design: REST vs GraphQL"}, "b": null}` |
| 3 | a mutation with the GET method | `GET /graphql?query={course...}` → 200 with data; `GET /graphql?query=mutation{likeCourse...}` → 400 `mutations are not allowed when using GET`; likes stayed 12 | GET must be safe: links can be opened, prefetched, cached; Strawberry accepts mutations only in POST | nothing to fix; restored data with `seed.sql` |
| 4 | `more_students.sql`: 3 and 20 students, the SQL counter | `GET /courses` → `x-sql-queries: 1`. 3 students: 7 and 7. 20 students: 27 and 24, status and speed in GraphiQL looked the same | lazy loading in resolvers: 1 + N (enrollments per student) + distinct courses = 1+3+3 and 1+20+6; second query 1+3+3 and 1+6+17 | step 6: DataLoaders → 3 and 3 on 20 students |
| 5 | a DataLoader batch function that doesn't sort rows by key | students query: `Received wrong number of results in dataloader, expected: 3, received: 6`. Enrollments query: no error, but enrollments 2, 4, 5, 6 showed someone else's course (2 → Backend Foundations instead of AI/LLM Integration) | the loader hands results out by position; with `ORDER BY id DESC` the 3 courses came in another order than the 3 keys, the length matched, so nothing complained. The silent case is worse | `git restore graphql_api.py`: rows are put into `groups` by `getattr(row, column.key)` and returned in key order |
| 6 | six dangerous queries without protection and with it | without: depth-6 cycle and 12 aliases executed (12 likes in one POST, 11 → 23, 24 SQL queries); `limit -1` and the 20-digit id returned psycopg errors with SQL text and parameters; `id "abc"` returned Python's `invalid literal for int()`. With: `'Cycle' exceeds maximum operation depth of 5`, `12 aliases found. Allowed: 10` (SQL queries: 0), `limit must be between 1 and 100`, `Invalid ID: 'abc'`, `Unexpected error.`; introspection still works | the client writes the query, so the server must limit it. Without protection the client learned the table and column names, SQL, parameters and that the server is Python/psycopg. REST hides it with a bare 500 | extensions + `check_limit`/`parse_id`/`should_mask`. Our own checks give a message the client can act on (`Invalid ID: 'abc'`), `Unexpected error.` doesn't. With `max_depth=2` the "My courses" query (depth 3) would break |
| 7 | introspection off, a typo in a field name | `__schema` → `GraphQL introspection has been disabled, but the requested query contained the field '__schema'.`; normal queries and try_graphql.py (8 of 8) work; `{ cours }` → `Did you mean 'course' or 'courses'?` | `DisableIntrospection` blocks only `__schema`/`__type`; validation hints still show real names | put `GRAPHQL_INTROSPECTION=true` back; access must be protected by permissions, not by hiding the schema |
| 8 | client failures: a typo, no `errors` check, the API stopped | typo: page 500, `Error [GraphQLRequestError]: Cannot query field 'titel' on type 'Course'...`. No check: `TypeError: Cannot destructure property 'student' of '(intermediate value)' as it is null.` API stopped: `TypeError: fetch failed` | GraphQL answers 200 even for errors, so `res.ok` is not enough; without the check the page sees `data: null` and fails on the consequence. When the API is down there is no answer at all | `git restore .` in course-catalog; `gql` keeps the `errors` check |

## Decisions

- **Rules in `services.py`.** The constraint names and their meaning are written in
  one place, and REST and GraphQL only translate the code: REST into a status and
  `type`, GraphQL into `userErrors`. If both layers checked everything, the same
  rule would be written twice and sooner or later the two copies would disagree
  (in experiments the same repeated enrollment gave `already-enrolled` in REST and
  `ALREADY_ENROLLED` in GraphQL from one `ServiceError`).
- **Loaders in `Context`.** A loader caches results by key; one loader for the whole
  app would keep old data after a like and could hand one user's data to another.
  `Context` is created for every request, so the cache lives only for one request.
  Rows are matched to keys by value because the database doesn't promise any
  order — experiment 5 showed that matching by position silently gives wrong
  courses.
- **Two layers of protection.** The cases we can foresee (`limit`, `id`) get a clear
  message the client can fix, like `Invalid ID: 'abc'`. Everything else may contain
  SQL or internal details, so `MaskErrors` replaces it with `Unexpected error.`,
  while the full traceback stays in the server window for the developer.
