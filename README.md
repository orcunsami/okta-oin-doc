# SOCRadar connector for Okta Workflows: user guide

**Connector version: 0.7.0**. This guide describes that version.

## What this connector does

SOCRadar collects credentials exposed on the dark web. This connector brings SOCRadar's botnet
exposure data into Okta Workflows. It also lets a flow close a SOCRadar alarm once the affected
account has been handled.

A typical flow runs on a schedule. It pulls newly exposed credentials and matches them against
your Okta users. It then acts on the accounts that match, for example by clearing their sessions
and suspending them. The connector supplies the data and the alarm action. The decision logic
lives in your flows.

| Card | Purpose |
|---|---|
| Get Botnet Data | Credentials stolen by infostealer malware and collected from botnet logs |
| Close Alarm | Change the status of a SOCRadar alarm |

---

## Set up the connection

The connector authenticates with a SOCRadar **API key**, sent in a request header. There is no
OAuth flow and no consent screen.

The connection also stores your **Company ID**. Every card builds its request path from it, so the
cards read the data of the SOCRadar company that the connection belongs to.

### Before you start

You need two values from the SOCRadar platform:

- **API key.** Generate it in the SOCRadar console with an account that has API access.
- **Company ID.** The number that identifies your SOCRadar company. It appears in the SOCRadar
  console URL after `/company/`.

### Create the connection

1. In the Workflows Console, open a flow and add a SOCRadar card.
2. Click the connection field on the card and choose **New Connection**.
3. Enter a name you will recognise, then the **API key** and the **Company ID**.
4. Save. Okta does not call SOCRadar when you save, so a wrong key is not reported at this step.
5. Test the connection with one Get Botnet Data call over a narrow window.
   - A wrong API key fails the card with HTTP `401` and no response body.
   - An empty Company ID fails the card the same way, with HTTP `404`.
   - Either failure stops the flow, unless the card sits inside an **If Error** block.
   - A request that SOCRadar accepts returns a body. Input errors come back inside that body. See
     *Reading a response* below.

### Change the key

Edit the connection and enter the new key. You can also create a new connection and point each
card at it.

### Security notes

- Okta stores the API key as a connection secret.
- Botnet records can contain **passwords in clear text**. Decide on purpose whether your flows
  write records to Workflows Tables. See *Performance and volume* below.

### What this connector can reach

Both cards call the SOCRadar platform API and nothing else. Each card uses one fixed SOCRadar path.

| Card | Method | Changes data? |
| --- | --- | --- |
| Get Botnet Data | GET | No |
| Close Alarm | POST | Yes. It changes the status of an alarm in SOCRadar |

**Neither card writes to Okta.** Every change in your Okta org is made by Okta's own Okta
connector, under the Okta connection that you authorize separately.

The flow template makes exactly four kinds of call to your Okta org:

| What the template does | Okta API call | OAuth scope required |
| --- | --- | --- |
| Find a user by login or email | `GET /api/v1/users?search=` | `okta.users.read` |
| Read one user's profile and status | `GET /api/v1/users/{idOrLogin}` | `okta.users.read` |
| Suspend a user | `POST /api/v1/users/{id}/lifecycle/suspend` | `okta.users.manage` |
| Clear a user's sessions and tokens | `DELETE /api/v1/users/{id}/sessions` | `okta.users.manage` |

The template never reads your System Log. It never changes group membership, application
assignments or policies. Two points matter when you scope a service app for it:

- `okta.users.manage` covers clearing sessions. There is no separate session scope.
- `okta.users.lifecycle.manage` and `okta.users.credentials.manage` look like OAuth scopes, but
  they are custom role permissions. Granting them to an OAuth application has no effect.

On the SOCRadar side the template calls Get Botnet Data. It calls Close Alarm only when you switch
alarm closing on. These are the only two cards, so the template uses every card the connector
offers.

---

## Card reference

### Reading a response

Every card returns five outputs. The last three appear under **Result**.

| Output | Type | Meaning |
|---|---|---|
| `Status Code` | number | HTTP status of the call to SOCRadar |
| `Body` | object | The parsed response body |
| `is_success` | boolean | `false` when SOCRadar rejected the request |
| `message` | text | SOCRadar's own message, for example the reason for a rejection |
| `response_code` | number | The code SOCRadar puts in the body. It can differ from `Status Code` |

- **HTTP 200 does not mean success.** SOCRadar reports a rejected request inside the body and still
  returns HTTP 200. In that case `is_success` is `false` and `response_code` is `400`. Always check
  `is_success`. A flow that checks only `Status Code` treats a rejected call as an empty result.
- The cards that come with the flow template show only `Status Code` and `Body`. On those
  cards, read `is_success` with an **Object Get** card: `object` is `Body`, `path` is `is_success`.
- Some failures reach the flow without a body: `401` for a wrong key and `404` for an empty
  Company ID. The `400` for an empty required input is raised before any request is sent.
- The Workflows editor will not save a card while a required input is empty. A flow that arrives
  through import can still carry an empty value. The card rejects an empty value when it runs.

### Get Botnet Data

Returns credentials stolen by infostealer malware and collected from botnet logs.

**Inputs**

| Input | Type | Required | Notes |
|---|---|---|---|
| `startDate` | text | yes | Start of the window, included. Format `YYYY-MM-DD HH:mm:ss` |
| `endDate` | text | yes | End of the window, not included. Same format |
| `page` | text | yes | Page number. Pages start at 1. Do not send 0: it returns the same records as page 1 |
| `limit` | text | yes | Records per page. The maximum is 5000 |

**Outputs**

The five outputs in *Reading a response*. The records sit two levels down in `Body`:

| Path in `Body` | Meaning |
|---|---|
| `data.data` | The list of exposure records |
| `data.total_data_count` | Number of records in the whole window. It does not depend on `page` |

To reach the list, chain two **Object Get** cards. The first reads `data` from `Body`. The second
reads `data` from the first result. A single Object Get with path `data` returns an object that
holds the list, not the list itself. A **For Each** over that object sees one item, and the run
still ends green.

**Behaviour to plan for**

- A `limit` above 5000 is rejected: HTTP 200, `is_success` `false`, `response_code` `400`, no
  records, and the message *API Input Error: 'limit' must be between 1 and 5000 inclusive*. Keep
  `limit` at 5000 or below.
- A date that cannot be read is rejected the same way, with the message
  *API Input Error: 'startDate' must be a valid date*.
- A window that ends before it starts is not rejected. It returns zero records.
- `total_data_count` covers the whole window. You are done when the records you fetched reach
  `total_data_count`.
- An empty page on its own does not mean the window is done. Compare with `total_data_count`
  before you move any checkpoint.
- Records come newest first. If you page across several runs, fix `startDate` and `endDate` once
  for the whole crawl. A window that moves between runs shifts the pages, so records can be read
  twice or skipped.

### Close Alarm

Changes the status of a SOCRadar alarm. A flow usually calls it to mark an alarm handled once the
affected account has been remediated.

**Inputs**

| Input | Type | Required | Notes |
|---|---|---|---|
| `alarm_id` | number | yes | The alarm to change |
| `status` | number | yes | The target status code. See the table below |
| `comments` | text | no | A note stored on the alarm |

**Outputs**

The five outputs in *Reading a response*. Check `is_success` and `message`.

**Status codes**

| Code | Status |
|---|---|
| 0 | OPEN |
| 1 | INVESTIGATING |
| 2 | RESOLVED |
| 4 | PENDING_INFO |
| 5 | LEGAL_REVIEW |
| 6 | VENDOR_ASSESSMENT |
| 9 | FALSE_POSITIVE |
| 10 | DUPLICATE |
| 11 | PROCESSED_INTERNALLY |
| 12 | MITIGATED |
| 13 | NOT_APPLICABLE |

SOCRadar returns this list itself when it rejects an invalid status. The flow template sends `12`,
MITIGATED.

**Behaviour to plan for**

- An unknown `alarm_id` is rejected inside the body: HTTP 200, `is_success` `false`,
  `response_code` `400`, and the message *No alarms found matching the given alarm IDs or selected
  filters!* It is not an HTTP error.
- An invalid `status` is rejected the same way, and nothing changes.
- Closing an alarm that already has the target status returns `is_success` `false`. For MITIGATED
  the message is *The selected alarms either do not exist or are already in the
  'Mitigated/Remediated' status.* One alarm can cover several people. A flow that closes the alarm
  once per person gets this answer as a matter of routine.
- Close alarms only after a crawl has finished. A closed alarm leaves the feed and the pages shift.

---

## Performance and volume

These points affect how long a flow runs and which Workflows limits it meets.

- **A wide window returns a lot.** Six months of botnet data for an active company can exceed
  100,000 records. At 5000 records per page that is more than 20 pages. If a helper flow runs once
  per record, one run takes hours. Start with a narrow window.
- **Page across runs, not inside one run.** Store the page number and the fixed window in a
  Workflows table. Fetch one page per scheduled run. Each run stays short, and a failed run resumes
  without reading everything again.
- **Workflows Tables are not a database.** Okta's own guidance is that Tables are not designed as
  scalable storage. Writing every raw record makes two table requests per record and will be
  throttled. Write only the records you acted on.
- **Passwords in clear text.** Botnet records can contain passwords. If you store records, you copy
  those passwords into your Okta org, where admins can read and export them. Store as little as you
  need.
- **Rate limits.** Get Botnet Data and Close Alarm count against your SOCRadar API quota. Okta
  cards that write to Okta, such as suspend and clear sessions, count against Okta's limits.

---

## Safety gates in the reference flow template

The cards only read from and write to SOCRadar. The decision to suspend an Okta user is made by
the flow template. The template makes a wrong decision pass several independent checks before
anything happens. This section maps those checks. You set them in the deployment runbook.

**Note.** These gates live in the flow template, not in the connector cards. A flow you build yourself
from the cards inherits none of them.

| Gate | Controlled by | What it does |
| --- | --- | --- |
| Dry run | `dry_run` | Evaluates and records every decision, but changes nothing in Okta. |
| Suspend switch | `enable_suspend` | Suspension is skipped unless this is on. |
| Alarm close switch | `enable_alarm_close` | Close Alarm is not called unless this is on. |
| Blast radius cap | `max_suspends_per_run` | Stops after a set number of real remediations in one run. |
| Verified domain allowlist | `verified_domains` | Acts on a user only if their email domain is on your list. An empty list blocks every action. |
| Protected account denylist | `never_remediate` | Never acts on the named accounts. An empty list blocks every action, so a forgotten list cannot lock you out. |
| Record retention switch | `save_records` | Controls whether feed records, which can hold passwords in clear text, are written to Workflows Tables. |
| Run lock | `run_active`, `run_started_at` | Stops two scheduled runs from working on the same window at once. It also clears a lock left behind by an interrupted run. |

Four more checks have no switch, because they are part of the structure:

- **Freshness.** If the password was changed after the leak was discovered, the leaked password is
  already dead. The user is skipped.
- **Already disabled.** A user whose Okta status is `SUSPENDED`, `DEPROVISIONED` or `DEACTIVATED`
  is skipped. Every other status can be remediated, `LOCKED_OUT` included. A lockout is temporary
  and is often caused by the attacker trying the leaked password, so a locked account is not a
  safe account. If Okta refuses to suspend a locked account, the run fails instead of passing
  quietly. The person is picked up again on the next run, because the deduplication row is written
  only after a successful suspend. `PASSWORD_EXPIRED`, `STAGED`, `PROVISIONED` and `RECOVERY` are
  handled the same way.
- **Not found.** A feed address that matches no user in your org is skipped.
- **Deduplication.** The unit is the person, meaning an alarm and address pair, not the alarm. One
  alarm that covers many people is processed once for each person, and never twice for the same
  person.

Every decision, every skip included, is written as a row in the **SOCRadar Audit** table. Its
columns are `run_ts`, `actor_flow`, `target_email`, `action`, `result`, `total_data_count`,
`fetched` and `detail`. To confirm a new deployment, run it with `dry_run` on and read that table
before you switch anything else on.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| The run is green but no records were processed | SOCRadar rejected the request inside the body. Check `is_success` |
| Zero records with `limit` above 5000 | The page limit was exceeded. Use 5000 or less |
| The first page is processed twice | `page` was 0. Pages start at 1 |
| A For Each over the records runs once | It reads `data` instead of `data.data`. Chain a second Object Get |
| Records are missing after a crawl across several runs | The window was recomputed each run. Store `startDate` and `endDate` once and reuse them |
| The card fails with `400` and names an input | A required input was empty. Supply a value |
| The card fails with `401` | The API key is wrong or was revoked. Edit the connection |
| The card fails with `404` | The connection has no Company ID |
| Close Alarm returns `is_success` `false` for a real alarm | The alarm already has the target status. Read `message` |
| A card shows "Unable to connect" | The connector has no connection test, so this label does not prove a fault. Run a narrow Get Botnet Data call |

## Related

The [SOCRadar API reference](https://github.com/orcunsami/socradar-api-docs) is the
OpenAPI file of the SOCRadar REST API. Its README lists the two paths this connector calls.

## Support

Support contact: **integration@socradar.io**. This is the address on the connector's support
contact field, and Okta publishes it in the OIN catalog entry.
