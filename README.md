# SOCRadar connector for Okta Workflows

**Connector version: 0.7.0**

SOCRadar collects credentials that infostealer malware has stolen and that later appear in
botnet logs. This connector brings that data into Okta Workflows. A flow can read the exposed
credentials, match them against your Okta users, act on the accounts that match, and then close
the SOCRadar alarm.

| Card | Description |
|---|---|
| Get Botnet Data | Retrieves credentials exposed through infostealer malware and botnet logs from SOCRadar for a date window. |
| Close Alarm | Changes the status of a SOCRadar alarm and optionally records a comment. |

Both cards call the SOCRadar platform API and nothing else. Neither card writes to Okta.

## Authorization

The connector authenticates with a SOCRadar API key, sent in the `API-Key` request header. The
connection also stores your SOCRadar Company ID. Every card builds its request path from it, so a
card reads the data of the company that the connection belongs to.

### Prerequisites

You need two values from the SOCRadar platform. Both are on the page
**Settings > API & Integrations > API Options > Company API Key**.

- **API key.** Use an account that has API access. Treat the key as a password.
- **Company ID.** The number that identifies your SOCRadar company. It also appears in the
  platform URL after `/company/`.

### Create a connection

1. In Okta Workflows, add a SOCRadar card to a flow.
2. On the card, click the connection field and choose **New Connection**.
3. Enter a name for the connection, then the **API key** and the **Company ID**.
4. Save the connection. Okta does not call SOCRadar at this step, so a wrong key is not reported
   yet.
5. Test the connection with one Get Botnet Data call over a narrow window. A wrong API key fails
   the card with HTTP `401`. An empty Company ID fails it with HTTP `404`.

To change the key later, edit the connection and enter the new key.

### Scopes

The connector uses an API key, so there are no scopes to select.

## Action cards

Both cards return the same outputs, in two groups.

- **Response** holds `Status Code` and `Body`.
- **Result** holds `is_success`, `message` and `response_code`, read from the body.

**HTTP 200 does not mean success.** SOCRadar reports a rejected request inside the body and still
returns HTTP 200. In that case `is_success` is `false` and `response_code` is `400`. Always check
`is_success`. A flow that checks only `Status Code` treats a rejected call as an empty result.

### Get Botnet Data

Retrieves credentials exposed through infostealer malware and botnet logs from SOCRadar for a
date window.

**Options**

This card has no options.

**Inputs**

Group: **Query**

| Label | Definition | Type | Required |
|---|---|---|---|
| `startDate` | Start of the window, included. Format `YYYY-MM-DD HH:mm:ss` | Text | Yes |
| `endDate` | End of the window, not included. Same format as `startDate` | Text | Yes |
| `page` | Page number. Pages start at 1 | Text | Yes |
| `limit` | Records per page, from 1 to 5000 | Text | Yes |

**Outputs**

Group: **Response**

| Label | Definition | Type |
|---|---|---|
| `Status Code` | HTTP status of the call to SOCRadar | Number |
| `Body` | The parsed response body | Object |

Group: **Result**

| Label | Definition | Type |
|---|---|---|
| `is_success` | `false` when SOCRadar rejected the request | True/False |
| `message` | SOCRadar's own message, for example the reason for a rejection | Text |
| `response_code` | The code SOCRadar puts in the body. It can differ from `Status Code` | Number |

The records sit two levels down in `Body`:

| Path in `Body` | Meaning |
|---|---|
| `data.data` | The list of exposure records |
| `data.total_data_count` | Number of records in the whole window. It does not depend on `page` |

To reach the list, chain two **Object Get** cards. The first reads `data` from `Body`. The second
reads `data` from the first result.

**Limitations and known issues**

- A `limit` above 5000 is rejected with HTTP 200, `is_success` `false` and no records. The message
  is *API Input Error: 'limit' must be between 1 and 5000 inclusive*.
- A date that cannot be read is rejected the same way. The message is
  *API Input Error: 'startDate' must be a valid date*.
- A window that ends before it starts is not rejected. It returns zero records.
- A `page` of 0 returns the same records as page 1, so the first page is read twice.
- A single Object Get with path `data` returns an object that holds the list, not the list itself.
  A **For Each** over that object sees one item, and the run still ends green.
- An empty page does not by itself mean the window is done. The window is done when the records
  you fetched reach `total_data_count`.
- Records come newest first. If you page across several runs, fix `startDate` and `endDate` once
  for the whole crawl. A window that moves between runs shifts the pages, so records can be read
  twice or skipped.
- An empty required input fails the card with HTTP `400` before any request is sent. The message
  names the input, for example *No startDate input provided. Enter a valid startDate.*
- Botnet records can contain passwords in clear text. See *Performance* below before you store
  them.

### Close Alarm

Changes the status of a SOCRadar alarm and optionally records a comment. A flow usually calls it
to mark an alarm handled once the affected account has been remediated.

**Options**

This card has no options.

**Inputs**

Group: **Body**

| Label | Definition | Type | Required |
|---|---|---|---|
| `alarm_id` | The alarm to change | Number | Yes |
| `status` | The target status code. See the table below | Number | Yes |
| `comments` | A note stored on the alarm | Text | No |

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

**Outputs**

The same **Response** and **Result** groups as Get Botnet Data. Check `is_success` and `message`.

**Limitations and known issues**

- An unknown `alarm_id` is rejected inside the body with HTTP 200 and `is_success` `false`. The
  message is *No alarms found matching the given alarm IDs or selected filters!*
- An invalid `status` is rejected the same way, and nothing changes. SOCRadar lists the valid codes
  in its message.
- Closing an alarm that already has the target status returns `is_success` `false`. For MITIGATED
  the message is *The selected alarms either do not exist or are already in the
  'Mitigated/Remediated' status.* One alarm can cover several people, so a flow that closes the
  alarm once per person gets this answer as a matter of routine.
- A closed alarm leaves the feed, and the pages of a crawl shift. Close alarms only after a crawl
  has finished.

## Performance

- **A wide window returns a lot.** Six months of botnet data for an active company can exceed
  100,000 records, which is more than 20 pages at 5000 records per page. If a helper flow runs once
  per record, one run takes hours. Start with a narrow window.
- **Page across runs, not inside one run.** Store the page number and the fixed window in a
  Workflows table and fetch one page per scheduled run. Each run stays short, and a failed run
  resumes without reading everything again.
- **Workflows Tables are not a database.** Writing every raw record makes two table requests per
  record and will be throttled. Write only the records you acted on.
- **Passwords in clear text.** If you store botnet records, you copy those passwords into your Okta
  org, where admins can read and export them. Store as little as you need.
- **Rate limits.** Both cards count against your SOCRadar API quota. Okta cards that write to Okta,
  such as suspend and clear sessions, count against Okta's limits.

## Flow template

SOCRadar also provides a flow template that uses these cards. The template, not the connector,
decides whether to suspend an Okta user. A flow you build yourself from the cards has none of the
checks below.

The template makes four kinds of call to your Okta org through Okta's own connector: find a user,
read a user, suspend a user and clear a user's sessions. The Okta connection needs the
`okta.users.read` and `okta.users.manage` scopes. The template never reads your System Log and
never changes groups, application assignments or policies.

| Setting | What it does |
| --- | --- |
| `dry_run` | Evaluates and records every decision, but changes nothing in Okta. |
| `enable_suspend` | Suspension is skipped unless this is on. |
| `enable_alarm_close` | Close Alarm is not called unless this is on. |
| `max_suspends_per_run` | Stops after a set number of real remediations in one run. |
| `verified_domains` | Acts on a user only if their email domain is on this list. An empty list blocks every action. |
| `never_remediate` | Never acts on the named accounts. An empty list blocks every action. |
| `save_records` | Controls whether feed records are written to Workflows Tables. |
| `run_active`, `run_started_at` | Stop two scheduled runs from working on the same window at once. |

The template also skips a user when the password was changed after the leak was found, when the
account is already `SUSPENDED`, `DEPROVISIONED` or `DEACTIVATED`, or when the address matches no
user. It handles each person once per alarm. A `LOCKED_OUT` account is not skipped, because a
lockout is often caused by the attacker trying the leaked password.

Every decision is written as a row in the **SOCRadar Audit** table. To confirm a new deployment,
run it with `dry_run` on and read that table before you switch anything else on. The cards that
come with the template show only `Status Code` and `Body`. On those cards, read `is_success` with
an **Object Get** card: `object` is `Body`, `path` is `is_success`.

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

The [SOCRadar API reference](https://github.com/orcunsami/socradar-api-docs) is the OpenAPI file
of the SOCRadar REST API. Its README lists the two paths this connector calls.

## Support

integration@socradar.io
