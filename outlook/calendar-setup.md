# Outlook: Shared PTO Calendar Setup

Approved PTO requests become all-day events on a calendar owned by a shared mailbox. This doc covers creating that mailbox, giving the service account access, controlling who can see the calendar, and how the flow builds each event.

| Item | Placeholder |
|---|---|
| Shared mailbox | `pto-calendar@example.com` |
| Service account | `svc-automation@example.com` |

---

# 1. Create the shared mailbox

How you create the mailbox depends on your environment:

| Environment | Where to create it |
|---|---|
| **Hybrid** (on-premises Active Directory synced to Microsoft 365) | Create it on-premises and let it sync to Microsoft 365. |
| **Cloud-only** (Exchange Online) | Microsoft 365 admin center → **Teams & groups → Shared mailboxes**, or the Exchange admin center. |

> **The mailbox must be hosted in Exchange Online.**
> Power Automate's Outlook connector works with cloud mailboxes only. A purely on-premises Exchange environment, with no Microsoft 365 tenant, isn't supported by this process. In a hybrid setup, wherever you create the mailbox, confirm it lives in Exchange Online before building the flow.

---

# 2. Give the service account access

The service account needs two separate permissions. Mailbox delegation does not grant calendar editing, so set up both.

# Delegate access to the mailbox

Lets the service account send the approval and rejection emails from the shared mailbox.

1. Open the **Exchange admin center** → **Recipients → Mailboxes**.
2. Select `pto-calendar@example.com` → **Delegation**.
3. Add `svc-automation@example.com` under **Read and manage (Full Access)** and **Send as**.

# Editor access to the calendar

Lets the service account create events. Run this in Exchange Online PowerShell:

```powershell
Connect-ExchangeOnline

Add-MailboxFolderPermission `
  -Identity "pto-calendar@example.com:\Calendar" `
  -User "svc-automation@example.com" `
  -AccessRights Editor
```

Verify it:

```powershell
Get-MailboxFolderPermission -Identity "pto-calendar@example.com:\Calendar"
```

The service account should be listed with **Editor**.

---

# 3. Control who can see the calendar

Access is restricted to **IT, HR and managers**. Other roles, such as office managers or front desk staff, or end-users depending on your environment can be added on request, with approval from the relevant departments.

Grant viewers access the same way:

```powershell
Add-MailboxFolderPermission `
  -Identity "pto-calendar@example.com:\Calendar" `
  -User "viewer@example.com" `
  -AccessRights Reviewer
```

> **Why access is restricted:** each event's body includes the employee's PTO reason and Notes. Anyone who can open the calendar can read them.

# Adding the calendar in Outlook

Once a person has permission, they can open the calendar themselves:

- **Outlook on the web / new Outlook:** Calendar → **Add calendar → Add from directory** → search for the shared mailbox.
- **Classic Outlook:** Calendar → **Add → Open Shared Calendar** → enter the shared mailbox name.

---

# 4. How the flow builds each event

The flow's **Create event (V4)** action (see [`power-automate/`](../power-automate/)) creates one all-day event per approved request.

| Setting | Value |
|---|---|
| Calendar id | The shared mailbox's calendar |
| Subject | `Display Name` - `Reason` |
| Start time | `Start Date` |
| End time | `addDays(…)` expression (End Date + 1 day) |
| Time zone | Your organization's time zone |
| Is all day event | Yes |

**Example:** Jane Doe requests vacation from March 3 to March 5. The event title is `Jane Doe - Vacation`, it starts March 3, and its end is set to March 6. Because Outlook all-day events use an **exclusive** end date, the calendar shows March 3–5. See [the +1 day rule](../docs/runbook.md#calendar-end-date-the-1-day-rule).

# Event body

```
[Display Name] - PTO
• Reason: [Reason]
• Start Date: [Start Date]
• End Date: [End Date]
• # of Days: [# of Days]
• # of Hours: [# of Hours]
• Notes: [Notes]
```

`Display Name` comes from Get user profile (V2). Every other value comes from the form.

All other advanced parameters (Show as, reminders, location, attendees) are left at their defaults.
