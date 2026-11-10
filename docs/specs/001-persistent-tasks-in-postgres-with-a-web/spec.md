---
spec_id: 001-persistent-tasks-in-postgres-with-a-web
title: Persistent tasks in Postgres with a web task list
status: draft
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks in Postgres with a web task list

## Problem

The Task API keeps tasks only in its own memory, so every restart or redeploy of the API wipes the board, and the API cannot run more than one instance. The only way to see or create tasks is to call the API directly; taskboard9 has a web UI component but it has no task features yet. Storing tasks in a Postgres database and giving the web UI a task list with a create form turns taskboard9 into a board people can actually use and rely on.

## Stories

As a taskboard9 user, I want the tasks I create to still be there after the system restarts or is redeployed, so that I can rely on the board as my record of work.
**Repos:** taskboard9-api

As a taskboard9 user, I want to see all tasks in the web UI, so that I do not have to call the API to know what has been recorded.
**Repos:** taskboard9-web, taskboard9-api

As a taskboard9 user, I want to create a task with a title and an optional due date in the web UI, so that I can record work without any tooling.
**Repos:** taskboard9-web, taskboard9-api

## Acceptance criteria

- **AC-001.1** Given a task created with `POST /tasks` `{"title": "Buy milk", "due_date": "2026-12-01"}`, when every instance of the API is stopped and started again, then `GET /tasks` still contains that task with the same `id`, `title` and `due_date`.
- **AC-001.2** Given tasks "A", "B" and "C" were created in that order and the API was then restarted, when a task "D" is created and a client sends `GET /tasks`, then the response lists "A", "B", "C", "D" in that order.
- **AC-001.3** Given the database is unreachable, when a client sends `GET /tasks` or a valid `POST /tasks`, then the response is 503 with a JSON body holding a `detail` message and no task is created; and when the database is reachable again, both requests succeed within 30 seconds without the API being restarted.
- **AC-001.4** Given task "A" with due date 2026-12-01 and then task "B" without a due date were created through the API, when a person opens the web UI's start page, then it shows a list headed "Tasks" with "A" above "B", "A" showing the due date "2026-12-01" and "B" showing "No due date".
- **AC-001.5** Given no task exists, when a person opens the web UI's start page, then it shows the text "No tasks yet." in place of the list, together with the create form.
- **AC-001.6** Given the start page is open, when a person enters the title "Buy milk" and the due date 2026-12-01 and submits the form, then "Buy milk" with "2026-12-01" appears as the last entry of the list without the person reloading the page, the form fields are cleared, and the API's `GET /tasks` contains that task; submitting with the due date left empty creates the task with no due date.
- **AC-001.7** Given the start page is open, when a person submits the form with a title that is empty or whitespace only, or longer than 200 characters after trimming, then the page shows "Title is required." or "Title must be at most 200 characters." respectively at the title field, keeps the entered values, and no task is created.
- **AC-001.8** Given the API is unreachable or answers with a 5xx status, when a person opens the start page, then it shows "Tasks could not be loaded. Please try again later." instead of the list (not "No tasks yet."); and when a person submits a valid task, the page shows "The task could not be saved. Please try again." and keeps the entered values.

## Non-goals

- Editing, completing or deleting tasks, and a page for a single task.
- Sign-in, permissions and per-user task lists.
- Carrying over tasks that the running API holds in memory when the database takes over.
- Database backups and restore, and running the system in staging or prod (promotion is a separate step).
- Pagination, filtering, sorting other than creation order, and search, in the API and in the web UI.
- Live updates: tasks created by someone else appear after the page is reloaded.

## Assumptions

- **A1** Tasks are stored in one PostgreSQL database per environment, declared as a resource of the taskboard9 system and used only by taskboard9-api; the web UI reads and creates tasks only through the API (other option: the web UI reads the database directly; affects AC-001.1 to AC-001.8).
- **A2** The API contract of taskboard9-api spec 001 stays unchanged (endpoints, fields `id`, `title`, `due_date`, validation, 422 errors, oldest-first order); only its AC-001.7 ("empty after restart") is superseded by AC-001.1 here (other option: extend the task with fields such as `created_at`; affects AC-001.1, AC-001.2).
- **A3** The database starts empty; tasks held in memory by the running API at the switch are not carried over, as they are already lost on every restart today (other option: migrate them once; affects AC-001.1).
- **A4** When the database is unreachable the API answers 503 with a JSON `detail` message, stores nothing and recovers by itself within 30 seconds (other option: 500, or the API stays down until restarted; affects AC-001.3).
- **A5** The task board is the web UI's start page: the list and the create form are on the same page, under the heading "Tasks" (other option: separate list and create pages; affects AC-001.4 to AC-001.8).
- **A6** The web UI shows all tasks, oldest first like the API, with the due date as `YYYY-MM-DD` or "No due date" (other option: newest first, or a localised date format; affects AC-001.4, AC-001.6).
- **A7** The web form checks the title with the API's rules (trimmed, 1 to 200 characters) before sending; the due date is optional and picked as a calendar date (other option: rely on the API's 422 alone; affects AC-001.6, AC-001.7).
- **A8** The user-facing texts are "Tasks", "No due date", "No tasks yet.", "Title is required.", "Title must be at most 200 characters.", "Tasks could not be loaded. Please try again later." and "The task could not be saved. Please try again." (other option: other wording; affects AC-001.4, AC-001.5, AC-001.7, AC-001.8).
- **A9** No sign-in: everyone who can reach the web UI sees and creates all tasks, as anyone who can reach the API can today (other option: a login before staging or prod exposure; affects AC-001.4 to AC-001.8).

## Open questions

None.

## Changelog

- 2026-11-10 created
