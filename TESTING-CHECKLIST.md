# RaceDay Part 2 — manual endpoint verification checklist

Use Swagger (`/swagger`) after the API is running. Keep the same browser session between login and protected requests.

1. Register a Participant with a valid email and password of at least 8 characters. Expected: `201 Created`.
2. Register the same email again. Expected: `409 Conflict`.
3. Log in with the correct password. Expected: `200 OK`; session cookie should be retained.
4. Log in with a wrong password. Expected: `401 Unauthorized`.
5. Call `GET /api/users/me` without logging in. Expected: `401 Unauthorized`.
6. Log in as Participant and call `POST /api/events`. Expected: `403 Forbidden`.
7. Log in as Organiser and create an event. Expected: `201 Created`.
8. Attempt to edit an event owned by another Organiser. Expected: `403 Forbidden`.
9. View the event list and filter by city/event type. Expected: `200 OK` and matching events.
10. Add a category to an event as its Organiser. Expected: `201 Created`.
11. Enrol a Participant in a valid category. Expected: `201 Created`.
12. Enrol the same Participant in the same category again. Expected: `409 Conflict`.
13. Attempt enrolment as Organiser. Expected: `403 Forbidden`.
14. View enrolments as the owning Organiser. Expected: `200 OK`.
15. Record a result for a valid enrolment as the owning Organiser. Expected: `201 Created`.
16. View event results and the Participant's own results. Expected: `200 OK`.
17. Try to modify another user's enrolment or another organiser's result. Expected: `403 Forbidden`.
18. Log out and retry a protected endpoint. Expected: `401 Unauthorized`.

Record actual results and screenshots for the report. Do not claim a test passed unless you ran it and observed the expected response.

---

# Part 3 — MVC application checklist

Run `docker compose up --build`, open http://localhost:8080 and record what you actually observe.

1. Open `/` while logged out → redirected to the login page. Try `/MyEnrolments`, `/Organiser/Dashboard`, `/ManageEvents` → all redirect to login.
2. Register a new Participant with a short password → friendly validation message, no crash.
3. Log in as `participant@raceday.test` → Home lists upcoming events; filters (type, province, city, dates, search) change the list.
4. Open an event → banner/placeholder, details and categories shown.
5. Enrol in a category → redirected to My Enrolments, status **Pending** (amber).
6. Try to enrol in the same category again → friendly "Already enrolled" message.
7. As Participant open `/ManageEvents` → access-denied page.
8. My Results shows the seeded "Heritage Day 5K" with time and "3rd out of 3 finishers".
9. Profile: update details; upload a profile picture (and a `.txt` file → friendly error).
10. Log out → login page; browser Back then refresh a protected page → login page again.
11. Log in as `organiser@raceday.test` → Dashboard shows totals, per-event enrolments and upcoming dates.
12. Create an event with a banner image; open its public page → banner displayed (served from Azure Blob / Azurite via the API).
13. Add an age category and a distance category; try a negative fee → friendly error.
14. Enrolments page → click **Confirm payment** on the participant's enrolment → status turns **Confirmed** (green).
15. Results page for a future event is locked; for "Heritage Day 5K" correct a result; try a duplicate position → friendly error.
16. Delete an event with enrolments → it is cancelled (message explains); delete one without → removed.
17. Stop the `api` container and reload a page → friendly "service unavailable" message, no stack trace.
