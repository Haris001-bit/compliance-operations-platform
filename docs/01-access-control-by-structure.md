# Deep dive: access control and data handling by structure

[Back to README](../README.md)

Detail on [§4](../README.md#4-access-three-levels-one-flag-three-checks), [§5](../README.md#5-reads-under-policy-writes-through-one-door) and [§6](../README.md#6-data-the-platform-is-built-not-to-hold) of the README.

---

## Structural versus conventional

A protection is **structural** when breaking it requires a deliberate, reviewable change: to the schema, to which database client a module imports, or to the contract with an integration. It is **conventional** when breaking it requires only forgetting.

| Protection | Conventional version | Structural version |
|---|---|---|
| Staff see only their own records | Every page remembers to filter by the current user | A read policy on the table; the filter lives in the database |
| The privileged key stays on the server | "Don't use that client in components" | A server-only module that no client component imports |
| No personal data in logs | "Be careful what you log" | Log entries carry IDs and nothing else |
| Documents stay private | Hard-to-guess paths | Private bucket, short signed links, UUID filenames |
| Third parties never see student data | "Don't sync that field" | Student records are not in the platform at all |

The last row is the strongest form. The cheapest protection for a category of data is not holding it. Student records live in a separate system of record and are not stored, duplicated or maintained here, which means no component in this platform needs a rule about them.

---

## Who can do what

The operations, mapped to the three ranked levels:

| Operation | Basic | Advanced | Admin |
|---|---|---|---|
| Read own compliance records and documents | Yes | Yes | Yes |
| Upload or update a compliance document | Yes | Yes | Yes |
| Read everyone's records and the aggregate summary | No | Yes | Yes |
| Create a plan, validate and submit its tasks | No | Yes | Yes |
| Delete a compliance record or a plan | No | No | Yes |
| Manage credential types and runtime settings | No | No | Yes |
| Create, import and re-sync staff | No | No | Yes |
| Trigger compliance notifications manually | No | No | Yes |
| Manage scheduled team notifications | No | No | Yes |

The capability flag grants exactly the last row and nothing else. It is a flag, not a fourth level, because it doesn't sit on the rank: holding it opens that one area and nothing above it.

Two rules keep the table honest. **Level is checked as a rank, not a list**, so "at least advanced" is one comparison and can't drift between handlers that each list the allowed levels slightly differently. **Job function never appears in this table.** It drives which credentials apply to a person, and that is all it drives.

---

## Three checks, and what each is for

| Layer | Runs | Catches | Can't catch |
|---|---|---|---|
| Edge route guard | Before render | Navigation to an area outside your level | Direct calls to handlers |
| Server page | Before any fetch | A render that slips past the guard | Writes, which don't go through pages |
| API handler | On every call | Everything, including calls nothing in the interface makes | Nothing it doesn't check |

Each layer fails in a different way, which is why they are independent rather than layered on trust. A guard driven by path patterns can be wrong about a path. A page check only protects that page. The handler check is the one that holds for writes, so every handler authenticates and authorises on its own and returns a consistent error shape when it refuses.

The page layer has one more structural property. Server components do all the data fetching and assembly; client components handle interactivity only. In the compliance dashboard, the client component filters and searches what the server already decided to send and does no fetching of its own, so there is no browser-side query path that could be widened by accident.

The scheduled job is the one route that has no session at all. It is let past the session guard on purpose and authenticates itself with a bearer secret supplied by the scheduler, returning 401 before any data access. A route without a session has to be its own guard.

---

## Three clients

| Client | Identity | Row-level security | Import rule |
|---|---|---|---|
| Cookie-bound server client | The signed-in user | Enforced | Server code |
| Browser client | The signed-in user, public key | Enforced | Client components |
| Service-role client | None; no session, no token refresh | Bypassed | Server code only, never a client component |

The service-role client is configured to hold no session and refresh no token, so it never carries a user's identity by accident, and nothing about it exists in the browser. The boundary is held by module layout: it lives in its own server module and no client component imports it.

### Reads

Two read policies cover compliance records, and the visible set is their union:

```
own rows:       row.person  = person_of(current_user)
elevated read:  level_of(current_user) in { advanced, admin }
```

That sketch is deliberately generic, but the shape is the point. A basic user's query and an admin's query can be byte-for-byte identical and return different rows. Filtering is not a step any page performs, so it is not a step any page can skip.

### Writes

Writes do not rely on row policies: all mutations go through the service-role client inside handlers that have already authorised the caller.

| Approach | Strength | Weakness |
|---|---|---|
| Write policies per table | The database refuses bad writes on its own | Rules spread across tables, in a language that is awkward to test and review |
| One server-side door | Write rules sit next to validation and audit logging | The database accepts any write the server makes |

I took the second and kept the door narrow. Where one update had to cross a restriction (writing a CRM reference onto another staff member's record, which the self-update restriction would otherwise block), it goes through a single privileged database function that does exactly that one thing. **A narrow function that can do one thing is a better shape than a general key that can do everything**.

---

## The audit log

Every compliance write, every server error and every manual notification trigger writes an entry. Entries reference people and credential types by ID and contain no names, no email addresses and no credential type names.

The credential type exclusion is the one that looks excessive until you think about it. A type name next to a person reference is a statement about that person, and an audit log is retained for as long as the reporting obligations require. Keeping the log to IDs means reading it meaningfully requires joining against tables already protected by policy, so the log inherits that protection rather than needing its own.

---

## Documents

```
upload   -> stored under a UUID filename in a private bucket
         -> path recorded on the credential row
view     -> server checks: own record, or advanced and above
         -> server mints a signed link valid for 5 minutes
replace  -> new file stored, old file deleted
delete   -> admin only, two-step confirmation, file and row both removed
```

Three structural choices are doing the work. The bucket has no public access, so there is no address that works without the server. The filename is a UUID, so a path or a storage listing says nothing about whose document it is. The link is minted per request and expires quickly, so a forwarded link is not lasting access.

---

## What is still convention

One protection here is still convention: the status derivation is copied into every file that handles compliance data, so it stays correct only while every copy is edited together, and moving it into one shared module would make it structural.

The repository's standing instructions also carry the incident protocol: if anything may have exposed protected student data, stop, alert leadership, log it, and do not recover silently. Writing that into the repository rather than a separate policy document puts it where people are working when something goes wrong.
