# SOCRadar connector for Okta Workflows

**Connector version: 0.13.0**

SOCRadar finds credentials that infostealer malware has stolen and published in botnet logs.
This connector brings that data into Okta Workflows. A flow can match the exposed credentials to
Okta users, act on them, and then close the SOCRadar alarm.

| Card | Description |
|---|---|
| Get Botnet Data | Retrieves credentials exposed through infostealer malware and botnet logs from SOCRadar for a date window. |
| Close Alarm | Changes the status of a SOCRadar alarm and optionally records a comment. |

Both cards call only the SOCRadar platform API. Neither card writes to Okta.

## Authorization

The connector sends a SOCRadar API key in the `API-Key` header. The connection also stores your
SOCRadar Company ID, and every request reads the data of that company.

### Prerequisites

In SOCRadar, open **Settings > API & Integrations > API Options > Company API Key** and copy two
values:

- **API key.** Treat it as a password.
- **Company ID.** It also appears in the platform URL after `/company/`.

### Create a connection

1. Add a SOCRadar card to a flow.
2. Click the connection field and choose **New Connection**.
3. Enter a name, the **API key** and the **Company ID**. Save.
4. The connector checks the connection with one call to the SOCRadar botnet endpoint over an
   empty date window. On the card, a working connection shows **Connection successful** and a
   wrong API key shows **Unable to connect**.

5. Assign the connection to the folder that holds your flows (**Connections**, open the
   connection, **Assign Folders**). Then open every SOCRadar and Okta connector card in each
   flow, choose the connection on the card, and save the flow. A card with no connection selected makes the first
   flow activation fail with HTTP 422 and no error text.

### Scopes

None. The connector uses an API key.

## Action cards

Both cards return two output groups: **Response** (`Status Code`, `Body`) and **Result**
(`is_success`, `message`, `response_code`).

**Check `is_success`, not `Status Code`.** SOCRadar returns HTTP 200 even when it rejects a
request. A rejection shows as `is_success` `false`, with the reason in `message`.

### Get Botnet Data

Retrieves credentials exposed through infostealer malware and botnet logs from SOCRadar for a
date window.

**Options**

None.

**Inputs**

Group: **Query**

| Label | Definition | Type | Required |
|---|---|---|---|
| Start Date | Start of the window, included. Format `YYYY-MM-DD HH:mm:ss` | Text | Yes |
| End Date | End of the window, not included. A time of `00:00:00` is the one exception. SOCRadar then reads it as the end of that day, so the day is included. Same format | Text | Yes |
| Page | Page number, starting at 1 | Text | Yes |
| Limit | Records per page, from 1 to 5000 | Text | Yes |

Dates use the format `YYYY-MM-DD HH:mm:ss` in UTC. The connector's date check also lets an ISO value with `T` and `Z`, such as `2026-11-17T10:00:00Z`, through to SOCRadar. The check reads only the digits of a value, so a time zone offset is not applied. A value the connector cannot read is not rejected by the flow. It is sent to SOCRadar as it is, and SOCRadar returns its own error response.

**Outputs**

Group: **Response**

| Label | Definition | Type |
|---|---|---|
| `Status Code` | HTTP status of the call | Number |
| `Body` | The parsed response body | Object |

Group: **Result**

| Label | Definition | Type |
|---|---|---|
| `is_success` | `false` when SOCRadar rejected the request | True/False |
| `message` | SOCRadar's message, such as the reason for a rejection | Text |
| `response_code` | The code in the body. It can differ from `Status Code` | Number |

The records are at `data.data` in `Body`. The total for the whole window is at
`data.total_data_count`. To reach the list, chain two **Object Get** cards that each read `data`.

**Limitations and known issues**

- A Limit above 5000, or a date that cannot be read, is rejected with HTTP 200 and `is_success`
  `false`.
- An empty required input fails the card with HTTP `400` before any request is sent.
- A Page below 1 fails the card with HTTP `400` and the message `page must be 1 or greater`.
- An End Date earlier than the Start Date fails the card with HTTP `400` and the message
  `endDate must be later than startDate`. An End Date with the time `00:00:00` counts as the end
  of that day, so it is accepted when it falls on the Start Date's day.
- A date the connector can read has 14 digits or more once every character that is not a digit is
  removed, or exactly 8 digits (a date only). Other values are passed to SOCRadar, which returns its
  own validation error.
- One Object Get on `data` returns an object, not the list. A For Each over it runs once, and the
  run still ends green.
- An empty page does not mean the window is done. It is done when the records you fetched reach
  `total_data_count`.
- Do not rely on the order records arrive in. When you page across runs, fix Start Date and
  End Date once for the whole crawl. Otherwise records are skipped or read twice.
- Records can contain passwords in clear text.

### Close Alarm

Changes the status of a SOCRadar alarm and optionally records a comment. A flow usually calls it
after the affected account has been remediated.

**Options**

None.

**Inputs**

Group: **Body**

| Label | Definition | Type | Required |
|---|---|---|---|
| Alarm ID | The alarm to change | Number | Yes |
| Status | The target status code, listed below | Number | Yes |
| Comments | A note stored on the alarm | Text | No |

Status codes: 0 OPEN, 1 INVESTIGATING, 2 RESOLVED, 4 PENDING_INFO, 5 LEGAL_REVIEW,
6 VENDOR_ASSESSMENT, 9 FALSE_POSITIVE, 10 DUPLICATE, 11 PROCESSED_INTERNALLY, 12 MITIGATED,
13 NOT_APPLICABLE.

**Outputs**

The same **Response** and **Result** groups as Get Botnet Data.

**Limitations and known issues**

- An unknown Alarm ID or an invalid Status is rejected with HTTP 200 and `is_success`
  `false`. Nothing changes.
- Closing an alarm that already has the target status also returns `is_success` `false`. One
  alarm can cover several people, so a flow that closes it once per person sees this routinely.
- A closed alarm can leave the feed and shift the pages. Close alarms after a crawl has
  finished.

## Performance

- Six months of data for an active company can exceed 100,000 records. Start with a narrow
  window.
- Fetch one page per scheduled run and keep the page number and the window in a Workflows table.
  Each run stays short, and a failed run resumes where it stopped.
- Workflows Tables are rate limited. Store only the records you acted on. Stored records can hold
  passwords in clear text.
- Both cards count against your SOCRadar API quota.

## Flow template

SOCRadar also provides a flow template built on these cards. It adds the safety checks: dry run,
a verified domain list, protected accounts and a cap on suspensions per run. It uses Okta's own
connector with the `okta.users.read` and `okta.users.manage` scopes. It skips a user whose
password changed after the leak, or who is already suspended, deprovisioned or deactivated. A
`LOCKED_OUT` user is not skipped, because a lockout is often the attacker trying the leaked
password. A flow
you build yourself from the cards has none of these checks.

Run the template with dry run on first and read the **SOCRadar Audit** table. The cards in the
template show only `Status Code` and `Body`. Read `is_success` there with an **Object Get** card.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| The run is green but no records were processed | SOCRadar rejected the request. Check `is_success` |
| The card fails with `401` | The API key is wrong or was revoked. Edit the connection |
| The card fails with `404` | The connection has no Company ID |
| A card shows "Unable to connect" | The connection check failed. Check the API key and the Company ID |

## Related

The [SOCRadar API reference](https://github.com/radargoger/socradar-api-docs) is the OpenAPI file
of the SOCRadar REST API. Its README lists the two paths this connector calls.

## Support

integration@socradar.io
