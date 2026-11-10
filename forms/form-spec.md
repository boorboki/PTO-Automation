# Microsoft Forms: Request For Time Off Form

This is the form employees use to request PTO. Each submission triggers the Power Automate flow (see [`power-automate/`](../power-automate/)). Build the form first: the flow's trigger needs an existing form to select.

---

# Form settings

Open the form, then **⋯ (More form settings) → Settings**.

| Setting | Value | Why it matters |
|---|---|---|
| Who can fill out this form | **Only people in my organization can respond** | Requires sign-in, so every response is tied to a work account. |
| Record name | **On** | **Required.** Captures the responder's name and email, which the flow uses to look up the employee's profile and manager. Without it, the flow can't find an approver. |
| One response per person | **Off** | Employees submit a new response for every PTO request, and resubmit after a time-out or manager change. Turning this on would block both. |
| Accept responses | **On** | |
| Start date / End date | **Off** | The form stays open year-round. |

---

# Layout

The form has one section, headed **Employee To Complete**, with six questions.

| # | Question | Type | Required | Notes |
|---|---|---|---|---|
| 1 | Reason | Choice (dropdown) | Yes | Why the time is requested. Also added to the calendar event title. |
| 2 | Start Date | Date | Yes | First day of PTO. Entered as M/d/yyyy. |
| 3 | End Date | Date | Yes | Last day of PTO. Entered as M/d/yyyy. |
| 4 | # of Days | Text | Yes | Free-text box. |
| 5 | # of Hours | Text | Yes | Free-text box. |
| 6 | Notes | Text (long answer) | No | For employee, manager, and HR reference only. |

# Reason options

<!-- TODO: Replace with your organization's PTO reasons. -->

| Option |
|---|
| Vacation |
| Sick |
| Personal |
| Bereavement |
| Jury Duty |

The selected reason appears in the calendar event title, so keep the options applicable to your organization's standards/short.

---

# How the flow uses the form

| Form data | Used by | Purpose |
|---|---|---|
| Responder's email (from Record name) | Get user profile (V2), Get manager (V2) | Identifies the employee and their approver |
| Reason | Create event (V4) | Calendar event title |
| Start Date, End Date | Create event (V4) | Event start and end (see the +1 day rule in the [runbook](../docs/runbook.md#calendar-end-date-the-1-day-rule)) |
| # of Days, # of Hours, Notes | Approval and notification emails | Shown to the manager and employee for reference |
| Submission Time | Error email | Used to find a failed run in Run History |

---

# Things that break the flow

> **Don't rename or delete questions after the flow is built.**
> The flow references each question by its internal ID, and the questions appear by name in the flow's dynamic content. Renaming or deleting a question can make those references empty or break the flow. Treat any question change as **high risk** (see [Making changes](../docs/runbook.md#5-making-changes)) and re-run the required tests afterward.

> **Empty End Date is expected.**
> When Start Date and End Date are the same day, Forms may return no End Date. The flow falls back to the Start Date with `coalesce()`, so this isn't a bug.

> **Don't turn on "One response per person."**
> Employees need to submit multiple requests over time.

---

# Building it from scratch

1. Go to [forms.office.com](https://forms.office.com) **as the service account** and create a **New Form** titled **Request For Time Off Form**.
2. Add a section titled **Employee To Complete**.
3. Add the six questions in the table above, in order, with the same names, types and Required settings.
4. Add your Reason options to question 1 and set it to display as a **dropdown** (⋯ on the question → Drop-down).
5. Apply the settings in [Form settings](#form-settings).
6. Submit one test response to confirm the form works, then build the flow.
