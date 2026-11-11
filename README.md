# RaceDay — Part 3 (MVC web front end + Azure Blob Storage + Docker)

RaceDay is a platform for running, walking and cycling events.

| Project | What it is |
|---|---|
| `RaceDay.Api` | The Part 2 REST API (ASP.NET Core minimal API, EF Core, SQL Server, session-cookie auth) — **extended in Part 3** with Azure Blob Storage image uploads, a dashboard endpoint, result/enrolment improvements and DTO responses |
| `RaceDay.Web` | **New in Part 3** — ASP.NET Core MVC app (Bootstrap 5) that consumes the API for *all* data and file operations |
| `RaceDay.Tests` | xUnit tests (auth hashing, UI formatting, image rules, mapping, file storage) |
| `Part1-docs` | ERD, SQL script and endpoint plan from Part 1 |

```
Browser ──► RaceDay.Web (MVC, session) ──HTTP + forwarded API session cookie──► RaceDay.Api ──► SQL Server
                                                                                      └──────► Azure Blob Storage
```
The MVC project has **no database access, no business rules and no Azure code**. Images are uploaded to the API (`multipart/form-data`) and served back through the API (`/api/files/...`); the MVC app only proxies the bytes (`/Files/Image?path=...`).

---

## Quick start — one command (Docker)

Requires Docker Desktop (or Docker Engine + Compose).

```bash
docker compose up --build
```

Then open **http://localhost:8080** (MVC app). Swagger for the API is at http://localhost:5100/swagger.

This starts SQL Server, the Azurite Azure Blob emulator, the API (creates the schema and seeds demo data) and the MVC app. The first start takes a minute while SQL Server initialises; the API waits and retries automatically.

**Demo accounts** (seeded because `Seed__DemoData=true` in `docker-compose.yml`; password `Password123!`):

| Role | Email |
|---|---|
| Organiser | `organiser@raceday.test` |
| Participant | `participant@raceday.test` |

Run **only the MVC container** against an API that is already running on your machine (port 5100):

```bash
docker build -f RaceDay.Web/Dockerfile -t raceday-web .
docker run --rm -p 8080:8080 --add-host=host.docker.internal:host-gateway -e Api__BaseUrl=http://host.docker.internal:5100 raceday-web
```

Stop everything with `docker compose down` (add `-v` to also delete the database and uploaded images).

## Run locally without Docker

Requirements: .NET 8 SDK, SQL Server LocalDB (or change the connection string).

```bash
dotnet restore RaceDay.sln
dotnet run --project RaceDay.Api    # http://localhost:5100  (creates the DB on first start)
dotnet run --project RaceDay.Web    # http://localhost:5200  (in a second terminal)
```

Set `"Seed": { "DemoData": true }` in `RaceDay.Api/appsettings.json` (or `Seed__DemoData=true`) to get the demo accounts/events above. Without a blob connection string the API falls back to local-disk storage (`App_Data/uploads`) and logs a warning — for the real thing use Azure (below) or Azurite.

### Azure Blob Storage configuration
Only the **API** needs this:
```json
"BlobStorage": { "ConnectionString": "<azure storage connection string>", "ContainerName": "raceday-images" }
```
Use user-secrets or environment variables (`BlobStorage__ConnectionString`) — never commit real keys. The container is created automatically as **private**; images are only reachable through the API. For local testing: `docker run -p 10000:10000 mcr.microsoft.com/azure-storage/azurite azurite-blob --blobHost 0.0.0.0 --skipApiVersionCheck` and use the Azurite connection string shown in `docker-compose.yml` (with `BlobEndpoint=http://127.0.0.1:10000/devstoreaccount1`).

---

## Requirements checklist (Part 3 brief)

| # | Requirement | Where |
|---|---|---|
| 1 | MVC app consuming the Part 2 API, no DB access / duplicated logic | `RaceDay.Web/Services/ApiClient.cs` is the only data gateway; controllers only call it |
| 2 | Register / log in via the API | `AccountController` → `/api/auth/register`, `/api/auth/login` |
| 2 | Authenticated state + role kept with session management | `SessionUser` in MVC session; the API session cookie is stored server-side in that session and forwarded on each call |
| 2 | Distinct interface & navigation per role | Layout nav differs for Participant and Organiser; separate controllers/views |
| 2 | Unauthenticated users can't reach protected pages | Global `SessionAuthFilter` (only login/register/error pages are `[AllowAnonymous]`); wrong role → access-denied page |
| 2 | End session and redirect | **Log out** → API logout + session cleared → login page |
| 3 | Organiser dashboard (events, enrolments per event, upcoming dates) | `Organiser/Dashboard` ← `GET /api/organiser/dashboard` |
| 3 | Create / edit / delete events (name, description, date, location, distance, type) | `ManageEvents` (Index, Form) |
| 3 | Event banner on create/edit | Banner field → `POST /api/events/{id}/banner` |
| 3 | Category management (age or distance) | `ManageCategories` |
| 3 | Enrolments page (participant, category, status) | `ManageEnrolments` (incl. *Confirm payment*) |
| 3 | Results entry (finish time + position) published | `ManageResults` → `POST /api/events/{id}/results`, `PUT /api/results/{id}` |
| 4 | Participant home with upcoming events + filters | `Home/Index` (search, type, province, city, date range) |
| 4 | Full event info + categories | `Events/Details` |
| 4 | Enrol + choose category, recorded immediately | Details → `POST /api/events/{id}/enrolments` |
| 4 | Personal view of enrolments + status | `MyEnrolments` |
| 4 | My Results ("47th out of 312 finishers") | `MyResults` ← `GET /api/users/me/results` |
| 4 | Profile view/update + profile picture | `Profile` → `PUT /api/users/me`, `POST /api/users/me/picture` |
| 5 | Azure Blob Storage in the API only; banner shown on event page | `RaceDay.Api/FileStorage.cs` (`AzureBlobFileStorage`), banner on `Events/Details` |
| 6 | Runs in Docker; well-structured Dockerfile; one-command start; same behaviour as local | `RaceDay.Web/Dockerfile` (multi-stage, non-root), `docker-compose.yml`, config via env vars |
| 7 | Branded, sporty UI, consistent Bootstrap layout, role nav, colour-coded statuses, graceful invalid-input handling | `wwwroot/css/site.css`, Razor views, validation + friendly error pages |

Enrolment colours: **Confirmed** = green (active and payment received), **Pending** = amber (active, awaiting payment), **Cancelled** = red, **Completed** = navy. A new enrolment is *Pending* until the organiser clicks **Confirm payment** on the Enrolments page.

## API changes made in Part 3
* `POST/DELETE /api/events/{id}/banner`, `POST/DELETE /api/users/me/picture`, `GET /api/files/{blob}` (Azure Blob Storage)
* `GET /api/organiser/dashboard`; `GET /api/events` gains `search` and `upcomingOnly`
* Events gain `DistanceKm` and banner; profiles gain a picture (columns are added automatically to existing Part 2 databases)
* Responses are DTOs (the Part 2 endpoints returned entities, which exposed password hashes and caused JSON cycles)
* Results: only once the event date has arrived, unique finishing position per category, `DisplayStatus` on enrolments, `MyResults` includes total finishers
* Validation tightened (event/category/profile fields, participants may only cancel their own enrolment, cancelled enrolments can be re-activated)

## Tests
```bash
dotnet test RaceDay.Tests/RaceDay.Tests.csproj
```
Manual test script: see `TESTING-CHECKLIST.md` (Part 2 API checks + Part 3 UI checks).

## Project layout
```
RaceDay.sln
docker-compose.yml
RaceDay.Api/   Program.cs  Models.cs  Dtos.cs  FileStorage.cs  DbInitializer.cs  Dockerfile
RaceDay.Web/   Controllers/  Views/  Services/ApiClient.cs  Filters/  Helpers/  Models/  wwwroot/  Dockerfile
RaceDay.Tests/ xUnit tests
Part1-docs/    ERD, SQL, endpoint plan
```

## Known limitations / notes
* Passwords for the demo accounts and the SQL `sa` password in `docker-compose.yml` are for local development only.
* Anyone can register as an Organiser (kept from Part 2 for the coursework); a real deployment would need an approval flow.
* Weather and route endpoints from Part 2 remain placeholders.
* Bootstrap, Bootstrap Icons, jQuery and Google Fonts load from CDNs, so the browser needs internet access.

## Before you submit
* Build and run it yourself (`docker compose up --build`) and click through both roles.
* Push your own commits (at least 20 meaningful ones) and add the green GitHub Actions screenshot and your unlisted video link here.
* Submit your GitHub repository link.
