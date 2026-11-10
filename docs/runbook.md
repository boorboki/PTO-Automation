# Runbook: PTO Request Automation

This runbook covers how the PTO automation works, what to do when it fails, and how to change it safely. Employees request time off in Microsoft Forms, Power Automate routes the request to their manager, and approved requests become all-day events on the shared PTO calendar in Outlook.

| Owner | Scope | Time zone | Status |
|---|---|---|---|
| IT | Full-day PTO only | Edit as applicable | Production · v1.1 |

---

# 1. How a request flows

1. **Employee submits the form.** The employee fills in the Request For Time Off Form. Their name and email come from their signed-in Microsoft/work account, so they don't type them.
2. **Flow collects the details.** Power Automate pulls the answers, looks up the employee's profile and manager, and emails the employee a confirmation that the request was received.
3. **Manager approves or rejects.** The manager gets an approval request (Approve / Reject). See the [Manager guide](manager-guide.md) for how managers respond.
4. **Approved.** The employee gets an approval email, and an all-day PTO event is created on the `pto-calendar@example.com` calendar. The event title includes the PTO reason.
5. **Rejected.** The employee gets a rejection email. No calendar event is created and the flow ends.

> **If the manager never responds**
> The run waits on the approval until it times out. No event is created, and the employee has to submit the form again. See [When the flow fails](#4-when-the-flow-fails).
<!-- TODO: Confirm whether a time-out triggers the failure notice described in Section 4 and v1.1, and state the timeout duration here. -->

# Components

| Component | Role |
|---|---|
| Microsoft Forms | The request form employees fill in. |
| Power Automate | Runs the workflow and approval logic. Built in the cloud Modern Designer, not legacy. |
| Outlook / Microsoft Graph | Creates the calendar event. |
| Shared mailbox + service account | Sends the notification emails and holds calendar access. |

---

# 2. The request form

**Form name:** Request For Time Off Form

| Field | Type | Required | Notes |
|---|---|---|---|
| Reason | Dropdown | Yes | Why the time is requested. Also added to the calendar event title. |
| Start Date | Date | Yes | First day of PTO. |
| End Date | Date | Yes | Can come back empty for single-day PTO (see below). |
| # of Days | Text | Yes | Free-text box. |
| # of Hours | Text | Yes | Free-text box. |
| Notes | Long text | No | For Employee, Manager, HR reference only. |

> **Empty End Date is expected**
> When Start Date and End Date are the same day, Forms may return no End Date at all. The flow handles this by falling back to the Start Date, so don't treat it as a bug.
<!-- TODO: Clarify how a Required field can come back empty, or note that the fallback is defensive. -->

---

# 3. Flow logic

| Step | Action | What it does |
|---|---|---|
| Trigger | When a new response is submitted | Starts the flow on each new form response. |
| Data | Get response details | Reads the answers and the Submission Time. |
| Data | Get user profile (V2) | Gets the employee's display name and details. |
| Data | Get manager (V2) | Finds the approver. |
| Approval | Start and wait for an approval | Approve / Reject, assigned to the manager. |
| Calendar | Create event (V4) | On approval, creates the all-day event in Central Time. |

# Calendar end date: the +1 day rule

Outlook all-day events use an exclusive end date: the end date marks where the event stops and isn't shown on the calendar. So the flow adds one day to the End Date. This does not add an extra visible day. The End Time expression in **Create event (V4)** is:

```
addDays(
  coalesce(
    triggerOutputs()?['body/EndDate'],
    triggerOutputs()?['body/StartDate']
  ),
  1
)
```

`coalesce` uses the Start Date when End Date is empty, and `addDays(…, 1)` sets the exclusive end.

---

# 4. When the flow fails

Any failure sends an error email. It includes the employee name, employee email, Submission Time (when the form was submitted) and the flow name.

1. **Find the failed run.** In Power Automate, open the flow's Run History and match the run using the employee email and the Submission Time from the error email.
2. **Is it a time-out?** This is the most common failure: the manager didn't respond to the approval in time. **Fix:** if the request wasn't already handled verbally, ask the employee to submit the form again.
3. **Is the manager missing?** The run fails at **Get manager (V2)** when the employee has no manager assigned. **Fix:** add the manager to the employee's Microsoft 365 profile, wait for the change to sync, then ask the employee to submit the form again.
4. **Neither? Check Create event (V4).** Confirm calendar permissions, that the service account is active, and that the Outlook connector is authenticated.

> **There's no automatic resubmit**
> Failed requests are never retried, so both known fixes end with the employee resubmitting. For anything else, find the cause before re-running, or you may repeat the failure or create a duplicate event.

# Quick troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| Approval timed out | No manager response | Employee resubmits (if not handled verbally) |
| Fails at Get manager | No manager assigned | Assign, wait for sync, employee resubmits |
| Fails at Create event | Calendar access | Check permissions, account, connector |
| No event, no error | Request rejected | Nothing to do |

---

# 5. Making changes

| Risk | Change |
|---|---|
| **Safe** | Email body text · manager assignments · notification recipients |
| **Test first** | Date logic · form date fields · calendar permissions |
| **High risk** | Renaming or deleting form questions · the End Date expression · End Date +1 logic · time zone setting in Create event (V4) · changing a date field's Required setting |

> **What high-risk changes can break**
> PTO showing the wrong number of days, calendar events not being created at all, or the flow failing when it runs.

# Required tests after any change

Run every test below before putting the flow back into production.

| Test case | Expected result |
|---|---|
| Single-day PTO | One all-day event shows on the correct day. |
| Multi-day PTO | Event covers exactly the requested range. |
| Month boundary | Event spans the correct dates across months. |
| Year boundary | Event spans the correct dates across years. |
| Start Date = today | Event is created successfully. |

# Change management

- Record every change in the [Changelog](../CHANGELOG.md).
- Test with a non-production or test user account.
- Make one change at a time.
- Re-run all the tests above after each update.

---

# 6. Ownership & access

| Item | Details |
|---|---|
| Flow owner | IT |
| Service account | `svc-automation@example.com` (dedicated automation account) |
| Mailbox access | Delegated access to `pto-calendar@example.com` |
| Calendar access | Editor on the `pto-calendar@example.com` calendar |

New maintainers should read this runbook before making any changes.

---

# 7. Limits & roadmap

These limits are intentional and aren't defects. They can be revisited if business needs change.

| Not automated, by design | Backlog |
|---|---|
| Half-day or hourly PTO (HR handles these manually) | Half-day PTO support |
| PTO balance checks | PTO balance tracking |
| Overlap detection between employees | Overlapping PTO detection |
| Past-dated PTO | Teams-based approvals |
