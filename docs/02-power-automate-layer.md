# Deep dive: the Power Automate layer

[Back to README](../README.md)

Detail on [§10 of the README](../README.md#10-the-automation-layer).

---

## Why a separate layer

The mail and the task lists already live in the school's Microsoft 365 tenant, and putting mailbox polling and task creation there keeps Microsoft access in one place. The work splits along that line: **the app is the intake and validation layer, and Power Automate is the execution layer.**

The three flows share the tenant, an automation service account and their connections. The absence flow keeps no copy of the staff directory: it resolves people from the app's staff table at runtime, so adding someone, or changing who they report to, is a data change rather than a flow edit.

---

## Out-of-office detection

### Trigger

A shared mailbox is polled every minute. Each new email is split into its own run, and runs execute in parallel, so one slow classification never holds up the next message and each run's variables belong to that email alone. End to end, a notification goes out within about 60 seconds of the email arriving.

### The classification contract

GPT-4o reads the subject, sender and body against a prompt built from context about the mailbox, more than 20 recognition patterns, an explicit definition of what is not an absence, a closed set of types, and a strict two-message output.

The patterns exist because keyword matching misses the messages people actually send: a description of what has happened that never says the person will be out, an email with no subject line, a reason that implies absence without stating it. The negative definition carries equal weight: without it, anything that mentions time off reads as a notice of absence.

| Type | Covers |
|---|---|
| Full day out | Not coming in |
| Late arrival | Coming in, later than usual |
| Leaving early | Coming in, leaving before the end of the day |
| Advance leave | A future absence, notified ahead |
| None | Not an absence |

### Two audiences, two messages

| Audience | Channel | Contains |
|---|---|---|
| Staff group chats | Three groups on a group-messaging app | First name only, no reason |
| The person's supervisor | SMS through the school's CRM | Full name, reason, timing, any future dates |

This is information architecture rather than presentation. Colleagues need to know someone is out; only the supervisor needs to know why. The reason for an absence can be the most sensitive thing in the email, and the broad audience never needs it.

### Sanitise, parse, gate

Models wrap JSON in markdown code fences often enough that the flow strips them before parsing, every time. The cleaned output is parsed into a typed schema: an is-absence flag, the name, the type, the timing detail and the two messages. The sender's address is extracted from the raw From header and lowercased so it matches the directory reliably. If the email is not an absence, the run ends there, before any lookup or send.

### Directory lookups

```
sender address
  -> staff record        name, supervisor, job function, cover flag
  -> supervisor record   CRM contact reference
  -> SMS address
```

The supervisor's CRM contact reference is the one the app's sync writes back. That couples the two halves in a useful way: when a sync fails, the app leaves the reference null and shows a badge on the staff roster, and that badge is also the place to look if a supervisor isn't being reached.

### Fan-out and the role branch

The group message is posted to three groups in sequence, and the supervisor message then goes by SMS. If the absent person's role needs a different operational response, the supervisor's message carries an extra note for it, because some absences are a different severity from others.

Five systems take part in one run: Outlook, the model, the staff database, the group-messaging app and the CRM.

---

## Email triage

The triage flow serves one staff member's mailbox, polled every minute with one run per email, and makes three model calls:

| Call | Model | Output | Decides |
|---|---|---|---|
| Triage | GPT-4o | JSON | Label, draft eligibility, task eligibility |
| Draft | GPT-5 | HTML fragment | The reply |
| Tasks | GPT-5 | JSON | A flat list of atomic tasks |

### One pass, three decisions

The first call decides the label (Action Required, Urgent, FYI or Review), whether a draft is warranted and whether tasks are warranted, all at once. The labels are defined with mutual exclusion rules and explicit tiebreakers. Making the three decisions in one pass keeps them consistent with each other.

**Recipient position comes before content.** The prompt first establishes whether the mailbox owner is a direct recipient or CC-only. A CC-only email can never be Action Required and never produces a draft or a task. Most email automation treats every recipient the same, and that single check is meant to remove a large class of false positives.

### Tag everything, then branch

Every email is tagged with its label as an Outlook category through Microsoft Graph, before and regardless of any branching. The label is visible in Outlook with no manual effort, and the triage value lands even when nothing downstream runs.

### The draft

The draft call uses a stronger model and a stricter brief: the recipient's voice, one draft covering every action item, a visible placeholder for any fact not yet confirmed, and an HTML fragment with no subject or signature.

The placeholder rule is the one doing the safety work. A fluent draft that confirms a fact nobody has checked is a commitment the staff member didn't make. A visible placeholder forces a human decision at exactly the point where one is needed.

The flow then fetches the message and conversation identifiers, creates a properly threaded reply draft through Graph, and appends the stored signature. **It is saved as a draft and never sent automatically.**

### Tasks

The task call runs after the draft branch, so each task can link to the draft. It returns a flat list of atomic tasks as JSON. The email's received timestamp is passed in rather than generated, and **a due date that isn't stated stays null.**

After cleaning (fences stripped, newlines normalised), each task is created in To Do with its title, optional due date and notes. The notes include a deep link that opens the draft reply in Outlook on the web, falling back to the original email when no draft was made, and the task is tagged Action Required.

The timestamp and due-date rules address the same failure: a model asked to produce a date will produce a plausible one. Passing the real value in and forbidding invention is the only reliable answer.

---

## Strategic tasks

The plan tasks validated in the app arrive by webhook as a plan year and an array of tasks, each with a name, the assignee's email and display name, a due date as an Excel serial number, a repeat cadence, notes and a category.

Excel stores dates as a day count from its own epoch, with a known leap-year quirk, so the flow converts serials to ISO datetimes with date arithmetic, without an external library. The end of the plan year is derived from the plan year field and becomes the boundary for every recurrence. Each of five strategic categories maps to an owner label prefixed to the task title.

| Repeat | To Do pattern | Interval |
|---|---|---|
| None | One-off task, optional due date | n/a |
| Weekly | Weekly, day of week taken from the start date | 1 |
| Biweekly | Weekly | 2 |
| Monthly | Absolute monthly | 1 |
| Quarterly | Absolute monthly | 3 |

Tasks are created through Graph with request bodies composed conditionally, so a field like the due date is omitted entirely when absent rather than sent empty.

---

## Patterns across the flows

| Pattern | Why |
|---|---|
| Strip fences, then parse into a typed schema | Model output habits must not reach a parser |
| Gate early | A non-absence stops before any lookup; a CC-only email gets a label and nothing else |
| Resolve people from data at runtime | The directory has one owner, the app; the strategic flow's owner mapping is the exception to fix |
| Draft, never send | A wrong automated email costs more than a reviewed one saves |
| Never invent dates | Pass real values in; use null when there are none |
| Minimise per audience | Each recipient gets the detail their job needs and no more |
