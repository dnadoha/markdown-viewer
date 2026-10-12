# Internal message resend (PFL) — AuthenticateProcessData2

Focused deep-dive on the pending internal-message resend branch in `BusinessHelper.AuthenticateProcessData` (`Hit.BAGViewer.Business\Process\GeneralAuthentication.cs:411–427`) and the database state it works with. Scope: only the internal (BKB) message send/resend flow.

<details>
<summary><h2 style="display: inline-block">Purpose & problem context</h2></summary>

We can send internal messages in bulk, but some receivers cannot take unbounded internal messages, and we cannot send partial messages. This document maps exactly how internal messages are generated, sent, and retried when they fail, and which database tables hold the state — so the solution for bounded delivery can be designed against the real mechanics.

The starting point is this branch (quoted verbatim):

```csharp
if (listNotificationBAGObj.Count > 0)
{
    Task.Run(() =>
     {
         NotificationStuffManager stuffManager = new NotificationStuffManager(connString, System.Configuration.ConfigurationManager.AppSettings["BagMsgHandler"]);
         List<process_message> result_per_msg = stuffManager.ExportBAGNotificationXml(listNotificationBAGObj, Path.Combine(this.exportDirectory, "InternalMessages"), this.ProcessCode, process_lastupdate);
         //try to resend all pending msg
         if (result_per_msg.Where(r => r.msg_status == "PFL" && !r.is_sent).Count() > 0)
         {
             Util.WriteLog(project_code + ":: Resending pending message from running process on " + DateTime.Now.ToString("MM/dd/yyyy HH:mm:ss"));
             stuffManager.RegenerateAllInternalMsg(Path.Combine(this.exportDirectory, "InternalMessages"), "", "", "");
         }
         UpdateRelatedBagObjects(listNotificationBAGObj, project_code, ProcessCode, UpdateRelBagObjMode.UpdateNummer);
         UpdateRelatedBagObjects(listNotificationBAGObj, project_code, ProcessCode, UpdateRelBagObjMode.UpdateTypeAddressableobj);
     });
}
```

Where it sits: at the end of `AuthenticateProcessData(...)`, after the objects were sent to the LV. It runs on a background `Task` — the user's request returns before the internal messages are sent.

</details>

<details>
<summary><h2 style="display: inline-block">The condition, line by line</h2></summary>

- `result_per_msg` — the result of `ExportBAGNotificationXml(...)` for **this run**: one `process_message` per generated XML (one per subscriber), each carrying `msg_status`, `err_log` and `is_sent`.
- `r.msg_status == "PFL"` — the message was not accepted (receiver rejected it, an ordering/validation guard blocked it, or the send failed).
- `!r.is_sent` — excludes messages that were **already sent before**. `is_sent` is an in-memory flag only (set to `true` when the receiver accepts, or when the duplicate guard finds an earlier `SSC` row); it is **not a database column**. In the database, "sent" means `msg_status = 'SSC'`.
- `.Count() > 0` — a single failed message of this run triggers a full pending sweep.
- `Util.WriteLog(...)` — writes to the viewer log (`LogPath` setting, `BAGViewer_<yyyyMM>.log`).
- `RegenerateAllInternalMsg(<Out>\InternalMessages, "", "", "")` — empty `FromLastupdate` means "find it yourself": the engine computes the oldest pending moment via `GetPendingMsgTime()` (see below).

</details>

<details>
<summary><h2 style="display: inline-block">How internal messages are generated and sent (where the bulk comes from)</h2></summary>

`ExportBAGNotificationXml(listBagObjInfo, exportDirectory, processCode, lastupdate)` (`NotificationStuffManager.cs:37`):

1. Loads the data of this process run — `GetAllData(...)` reads the run's objects per type (panden, verblijfsobjecten, nummers, ligplaatsen, standplaatsen, openbare ruimte, woonplaats, ...) and the run's subscribers (`select * from msg_subscriber where msg_category = 'BKB' and project = '<project>'`).
2. Builds one XML **per subscriber** via `PrepareDataForXXX(...)` (one method per process code; e.g. `PrepareDataForBRAHOR`, `PrepareDataForBAGMUT`). Each XML contains **all objects of the run** — there is no size limit.
3. Inserts one `bag.process_message` row per XML with status `CRT` (`Insert_Proc_Message`).
4. Sends each message individually over SOAP (`SenderManager.Send` → `subscriber.endpoint`): receiver accepts → `SSC`; receiver rejects → `PFL` + the receiver's reason; exception → `PFL` + exception message.

Two ways bulk arises:

- **Objects per message** — one message carries the whole run for a receiver (unbounded).
- **Messages per sweep** — all pending runs are sent back-to-back, without pacing or throttling.

Ordering guard: messages per receiver must go out in process order. `IsPrevMsgSent` checks the previous authenticated run's message for the same sender/receiver; if it is not `SSC`, the new message becomes `PFL` with *"Process fails as previous message(s) is not sent yet."* — a receiver that cannot keep up blocks its own later messages.

Precedent for bounding: the LV (Kadaster) messages **are** chunked — `GenerateXMLFiles` caps each XML at `maxOccurs` (app setting, currently 100) objects (`GeneralAuthentication.cs:218`). Internal BKB messages have no equivalent.

</details>

<details>
<summary><h2 style="display: inline-block">What RegenerateAllInternalMsg(folder, "", "", "") does</h2></summary>

This is the same function used by the manual resend actions. With an empty timestamp it:

1. Calls `GetPendingMsgTime()` — returns the **oldest** `bag.process_header` timestamp for which at least one receiver has no `SSC` message; only `status = 'Authenticated'` runs count; codes `BAG_AIO` and `BAG_OA` are excluded.
2. If nothing is pending, returns `Success` with *"Geen bericht gevonden om te verzenden"* and closes the user notification.
3. Otherwise loads every authenticated run from that timestamp onwards (`GetMessageInfoFromLastupdate`), in timestamp order.
4. For each run: calls `ExportBAGNotificationXml(...)` again — this **regenerates the XMLs, inserts fresh `process_message` rows (CRT) and re-sends**. The duplicate guard `IsMsgSentBefore` prevents re-sending a message that already has an `SSC` row for the same timestamp/sender/receiver (those get `PFL` + *"already sent"* instead).
5. Logs the result of every message (sender, receiver, process code, timestamp, status, reason).
6. `break` — if a whole run produced **no** `SSC` at all, the rest of the pending runs are skipped.
7. If the sweep produced at least as many `SSC` messages as there were runs, the user notification is closed (the check is `SSC count >= runs walked`, not per-receiver).

Recap:

```
branch (any PFL && !is_sent)
  └─ RegenerateAllInternalMsg(folder, "", "", "")
       ├─ GetPendingMsgTime()  → oldest timestamp with a receiver missing SSC
       └─ for each authenticated run from there (timestamp order):
            ExportBAGNotificationXml(...)   → regenerate + send per subscriber
            no SSC in this run?  → break
       └─ enough SSC?  → close user_notification
```

Note: the sweep always restarts from the **oldest** pending run. If one receiver keeps failing, that timestamp never advances and every subsequent run re-attempts the whole pending range.

</details>

<details>
<summary><h2 style="display: inline-block">Status model and failure reasons</h2></summary>

| Status | Meaning | Set when |
|---|---|---|
| `CRT` | Created — row inserted, send not attempted yet | `Insert_Proc_Message` (`DBCommunicator.cs:118`) |
| `SSC` | Success — receiver accepted the message | `SenderManager.Send` on `Accept` (`SenderManager.cs:71`) |
| `PFL` | Failed/pending — not accepted; candidate for resend | guards, validation, or send failure |

`err_log` values used by this flow:

| err_log | Cause | Set in | Resent by the branch? |
|---|---|---|---|
| `Process fails as this message is already sent.` | An `SSC` message for the same timestamp + sender + receiver already exists (duplicate guard) | `NotificationStuffManager.cs:252` | **No** — `is_sent = true`, excluded by `!r.is_sent` |
| `Process fails as previous message(s) is not sent yet.` | Previous authenticated run's message to this receiver is not `SSC` (ordering guard) | `:267` | Yes |
| `Process fails as the message data is not authenticated yet.` | `process_header` for this timestamp is not `Authenticated` | `:260` | Yes |
| `Process fails as message is not valid.` | Generated XML failed XSD validation | `:281` | Yes |
| *(receiver's reject description)* | Receiver answered with a reject | `SenderManager.cs:76` | Yes |
| *(exception message)* | SOAP / network / certificate / timeout | `SenderManager.cs:82` | Yes |

Retry behaviour worth knowing:

- Every retry inserts a **new** `process_message` row and updates that row — old `PFL` rows stay in the table.
- Pending detection asks "does this receiver have **any** `SSC` for this timestamp" — stale `PFL` rows do not block, but they accumulate.
- `WriteMessageInfo` and the user notification are written for `SSC` and for `PFL` that is not the "already sent" case (`:271`).
- User notification (`bag.user_notification`): created `Open` with `source = INTERNAL_MSG_ENGINE`, `type = IN_APP`, `level = error`, `message = err_log` — only if no `Open` row for that source exists yet. Set to `Close` by a resend sweep that succeeds (or finds nothing pending).

</details>

<details>
<summary><h2 style="display: inline-block">Database — BAG project schema (bag.*)</h2></summary>

### bag.process_message — one row per generated message

```sql
CREATE TABLE bag.process_message
(
    id integer NOT NULL GENERATED ALWAYS AS IDENTITY (...),
    proc_code character varying NOT NULL,
    proc_timestamp character varying NOT NULL,
    msg_type character varying,
    sender character varying,
    receiver character varying,
    msg_status character varying,
    err_log text,
    msg_xml_path character varying,
    receiver_response text,
    xsd_validated boolean DEFAULT false,
    CONSTRAINT process_message_pkey PRIMARY KEY (id)
)
```

| Column | Holds | Written by |
|---|---|---|
| `proc_code` | Process code of the run (e.g. `BAG_MUT`) | insert |
| `proc_timestamp` | The run's process timestamp — joins to `process_header."timestamp"` | insert |
| `msg_type` | BKB message type (`Mapper.Process2BKBTypeMapDic[proc_code]`) | insert |
| `sender` / `receiver` | Subscriber app codes (`sender_app` / `receiver_app`) | insert |
| `msg_status` | `CRT` → `SSC` / `PFL` | update (`Update_ProcMsg`) |
| `err_log` | Failure reason (see table above) or validation summary | update / validation |
| `msg_xml_path` | Full path of the generated XML (used as the update key) | insert |
| `receiver_response` | Raw SOAP response from the receiver | update |
| `xsd_validated` | Result of XML validation | validation |

Important: `msg_id` (message-handler id) and `is_sent` exist only in the runtime model (`process_message` class, `msg_subscriber.cs:24`) — they are **not columns**; the insert explicitly excludes `id`, `msg_id`, `is_sent`.

### bag.process_header — one row per process run

```sql
CREATE TABLE bag.process_header
(
    id integer NOT NULL GENERATED ALWAYS AS IDENTITY (...),
    code character varying NOT NULL,
    "timestamp" character varying NOT NULL,
    date date NOT NULL,
    status character varying,
    user_name character varying,
    pcode_version character varying,
    xml_string text,
    remark character varying,
    doc_date date, send_to_lv_date date, auth_date date,
    error_date date, checkout_date date, checkin_date date,
    err_log text,
    CONSTRAINT process_header_pkey PRIMARY KEY (id)
)
```

Used by this flow for: `status = 'Authenticated'` (send gate, `CanMsgSend`), `"timestamp"` (ordering + joins), `code` (excluded codes `BAG_AIO`, `BAG_OA`), pending detection (`GetPendingMsgTime`).

### bag.user_notification — the open/close notification

```sql
CREATE TABLE bag.user_notification
(
    id integer NOT NULL GENERATED ALWAYS AS IDENTITY (...),
    type character varying,
    level character varying,
    source character varying NOT NULL,
    message character varying,
    status character varying,
    created_date character varying,
    updated_date character varying,
    CONSTRAINT pk_user_notification PRIMARY KEY (id)
)
```

Lifecycle: `Open` on send/failure (see above); `Close` when a resend sweep succeeds or finds nothing pending. `GetNotificationOfInternalMsg` reads the `Open` row to warn the viewer before the next authentication.

Repo scripts: `SQLs\AOXX\BAG\BAG_tables.sql` (`process_header` :3042, `process_message` :3075, `user_notification` :5998). No indexes on `proc_timestamp` / `sender` / `receiver` are defined in these scripts — the pending/ordering queries join and scan on them, which matters as the `PFL` queue grows.

</details>

<details>
<summary><h2 style="display: inline-block">Database — message-handler side (GBCMBSYS)</h2></summary>

The message-handler database (connection string from the viewer setting `BagMsgHandler`) keeps the receiver-side registry.

### messages — meta + status per internal message

Base table as in the repo script (`MessageHandlers\BGTLink Message Handler\Database\Scripts\PostgreSQL\MH-Tables.sql:146`):

```sql
CREATE TABLE messages
(
  msg_id             serial NOT NULL,
  proj_code          varchar(5) NOT NULL,
  gov_code           varchar(5) NOT NULL,
  msg_dir            varchar(3) NOT NULL,
  msg_type           varchar(3) NOT NULL,
  msg_ref            varchar(40) NOT NULL DEFAULT '',
  msg_crossref       varchar(40) NOT NULL DEFAULT '',
  msg_timestamp      varchar(20) NOT NULL DEFAULT '',
  request_id         varchar(40) NOT NULL DEFAULT '',
  request_ref        varchar(40) NOT NULL DEFAULT '',
  request_timestamp  varchar(20) NOT NULL DEFAULT '',
  request_descr      varchar(500) NOT NULL DEFAULT '',
  msg_xml            text NOT NULL DEFAULT '',
  status             varchar(3) NOT NULL DEFAULT '',
  response_ref       varchar(40) NOT NULL DEFAULT '',
  response_crossref  varchar(40) NOT NULL DEFAULT '',
  response_xml       text NOT NULL DEFAULT '',
  attachment         varchar(255) NOT NULL DEFAULT '',
  extra1             varchar(500) NOT NULL DEFAULT '',
  extra2             varchar(500) NOT NULL DEFAULT '',
  record_log         text NOT NULL DEFAULT '',
  CONSTRAINT pk_messages PRIMARY KEY (msg_id)
)
```

What the engine writes (`WriteMessageInfo`, `NotificationStuffManager.cs:342` → `MessageManager.SendMessageMetaInfo` → `MessageArchive.WriteMessage`): `msg_id`, `proj_code`, `gov_code`, `process_code`, `msg_dir='OUT'`, `msg_type='BKB'`, `msg_ref` (referentienummer), `status` (= `msg_status`: `CRT`/`SSC`/`PFL`), `record_log` (= `err_log`), `process_timestamp`, `response_xml` (receiver response), `created_at`, `request_descr`, `msg_timestamp`, `request_timestamp`, `attachment` (XML file name), `sender`, `receiver`. Insert when `msg_id` is empty, update otherwise; the database returns `msg_id` (kept in memory only).

Caution: the repo script is an older snapshot — the code writes columns not present in it (`process_code`, `process_timestamp`, `created_at`, `sender`, `receiver`, `request_descr`). Check the live schema when designing.

### msg_subscriber — who receives what, and where

Queried as `select * from msg_subscriber where msg_category = 'BKB' and project = '<project>'`. Fields (model `msg_subscriber.cs:9`): `msg_category`, `project`, `sender_app/admin/org/user`, `receiver_app/admin/org/user`, `endpoint` (SOAP URL). One row = one receiver for the project; this is the natural place to configure per-receiver limits.

</details>

<details>
<summary><h2 style="display: inline-block">Key queries (verbatim, SqlProvider.cs)</h2></summary>

Oldest pending run — a timestamp counts as pending if any receiver has no `SSC` row for it:

```sql
select ph.timestamp
from bag.process_header ph
left join
(   select pm.receiver, pm.proc_timestamp
    from bag.process_message pm
    group by pm.receiver, pm.proc_timestamp
) t1 on ph.timestamp = t1.proc_timestamp
left join bag.process_message t2 on t2.receiver = t1.receiver and t1.proc_timestamp = t2.proc_timestamp and t2.msg_status='SSC'
where t2.id is null and ph.status = 'Authenticated' and ph.code not in ('BAG_AIO','BAG_OA')
order by ph.timestamp;
```

Runs to resend (authenticated runs from a timestamp onwards):

```sql
select code proc_code, timestamp proc_timestamp, '{1}' sender, '{2}' receiver
from bag.process_header
where timestamp >= '{0}' and status = 'Authenticated'
order by timestamp;
```

Ordering guard — was the previous authenticated run's message to this receiver sent?

```sql
select pm.id, pm.msg_status
from bag.process_header ph
left join bag.process_message pm on pm.proc_timestamp = ph.timestamp and pm.sender = '{1}' and pm.receiver = '{2}'
where ph.timestamp < '{0}' and ph.status = 'Authenticated'
order by ph.timestamp desc, pm.msg_status desc limit 1;
```

Send gate — is this run authenticated?

```sql
select ph.id from bag.process_header ph
where ph.timestamp = '{0}' and ph.status = 'Authenticated';
```

Duplicate guard — was this exact message already sent?

```sql
select proc_timestamp, msg_status from bag.process_message
where proc_timestamp = '{0}' and sender = '{1}' and receiver = '{2}' and msg_status = 'SSC'
order by proc_timestamp;
```

Write-back after a send attempt:

```sql
update bag.process_message
set msg_status = '{0}', receiver_response = '{1}', err_log = '{2}'
where msg_xml_path = '{3}';
```

Insert of new messages (generated per row): `insert into bag.process_message (proc_code, proc_timestamp, msg_type, sender, receiver, msg_status, err_log, msg_xml_path, receiver_response, xsd_validated) values (...)` — empty values become `null`; `id`, `msg_id`, `is_sent` are excluded.

</details>

<details>
<summary><h2 style="display: inline-block">Entry points & related endpoints</h2></summary>

- Automatic — the branch in this document, on every `AuthenticateProcessData` run that produced internal messages.
- Manual (hub `ProcessHub` + REST `BAGProcessController`, same operations):
  - `ResendPendingInternalMsg(report_code)` — "resend now": the identical `RegenerateAllInternalMsg(folder, "", "", "")` call.
  - `RegenerateAllInternalMsg(report_code, FromLastupdate, Sender, Receiver)` — sweep starting from a given timestamp / filtered to sender + receiver.
  - `GetNotificationOfInternalMsg(report_code)` — pre-check: returns the open notification.
  - `SendBKBMessage(procCode, timeStamp, reportCode)` — sends the pending messages of one run directly (`SenderManager.Send(processCode, timestamp)`).
- Viewer pre-check: before authenticating, the viewer calls `GetNotificationOfInternalMsg`; while a send-failure notification is open it warns: *"Door een eerder opgetreden fout is het momenteel niet mogelijk om interne berichten te versturen…"*
- BusinessHelper wrappers: `RegenerateAllInternalMsg` :2181, `GetNotificationOfInternalMsg` :2227, `ResendPendingInternalMsg` :2242.

Settings involved:

| Setting | Used for |
|---|---|
| `BagMsgHandler` | Connection string of the message-handler DB (subscribers + `messages`) |
| `LogPath` | Viewer log file (`BAGViewer_<yyyyMM>.log`) |
| `maxOccurs` | LV message chunk size (currently 100) — the existing bounding precedent |

</details>

<details>
<summary><h2 style="display: inline-block">Observations for the bounded-delivery solution</h2></summary>

Observations from the mechanics above (analysis notes, not design decisions):

- **Atomicity today**: one message = the complete object set of a run for one receiver. There is no chunking for BKB messages; the LV path shows the codebase already knows how to bound messages (`maxOccurs`).
- **Ordering**: strict per receiver. A receiver that rejects or stalls accumulates `PFL` and blocks its own later messages; other receivers continue as long as at least one message of a run succeeds.
- **Sweep behaviour**: every failure triggers a sweep that restarts from the **oldest** pending run; a permanently failing receiver means the whole pending range is re-attempted on every run, and `break`-on-no-`SSC` can stop a sweep before reaching later runs.
- **No pacing**: sends go out back-to-back with no delay or per-receiver limit.
- **Queue growth**: each retry inserts new `process_message` rows; old `PFL` rows remain (no cleanup in this flow), and the repo scripts define no indexes for the join keys.
- **Notification signal**: the `Open` notification is also created on successful sends (uniqueness is only `source` + `status`, and a successful message has no `err_log`), so it is not a pure failure signal — it is only closed by a later successful resend sweep (or a no-pending sweep).
- **Configuration point**: `msg_subscriber` already holds one row per receiver + endpoint per project — the obvious home for per-receiver limits.
- **Feedback already in place**: user notification (`Open`/`Close`) and the viewer pre-check warning.

Questions to settle while designing: what exactly do the limited receivers choke on (number of objects in one message vs. number of messages in a burst)? What is the acceptable maximum per message? Would receivers accept multiple *complete* messages (chunks, each a full BKB message with its own referentienummer) even though a single message cannot be partial? Where should pacing happen — generation, sending, or scheduling?

</details>

<details>
<summary><h2 style="display: inline-block">Source map</h2></summary>

| File | Relevant parts |
|---|---|
| `Hit.BAGViewer.Business\Process\GeneralAuthentication.cs` | `AuthenticateProcessData` :374; resend branch :412–427; LV chunking `maxOccurs` :218 |
| `HIT.BAG.NotificationEngine\NotificationStuffManager\NotificationStuffManager.cs` | `ExportBAGNotificationXml` :37; `WriteMessageInfo` :342; `RegenerateAllInternalMsg` :409; `WriteUserNotificatioin` :460; `GetNotificationOfInternalMsg` :481 |
| `HIT.BAG.NotificationEngine\Sender\SenderManager.cs` | SOAP send, `SSC`/`PFL` writes :31 |
| `HIT.BAG.NotificationEngine\DBAccess\SqlProvider.cs` | all SQL :266–387 |
| `HIT.BAG.NotificationEngine\DBAccess\DBCommunicator.cs` | DB calls :109–219 |
| `HIT.BAG.NotificationEngine\NotificationStuffManager\msg_subscriber.cs` | models: `msg_subscriber` :9, `process_message` :24 |
| `HIT.BAG.NotificationEngine\Processes\Common.cs` | `GetAllData` :37, subscriber load :142 |
| `HIT.BAG.NotificationEngine\Processes\OprProcesses.cs` / `TerugmeldingProcesses.cs` | message builders (`PrepareDataForBRAHOR` :41 / `PrepareDataForBAGMUT` :498) |
| `Hit.BAGViewer.Business\BusinessHelper.cs` | wrappers :2181, :2227, :2242 |
| `HitBAGApi\HitBAGApi\Hub\ProcessHub.cs` / `Controllers\BAGProcessController.cs` | endpoints :141/:391/:419, :368/:400/:408/:413 |
| `HIT.UserNotificationMsgLib\NotificationManager.cs` + `DataModels.cs` | user notification CRUD + model |
| `Common\HIT.MessageArchiver\HIT.MessageArchiver\MessageArchive.cs` / `MessageManager.cs` | message-handler DB writes |
| `SQLs\AOXX\BAG\BAG_tables.sql` | DDL :3042, :3075, :5998 |
| `MessageHandlers\BGTLink Message Handler\Database\Scripts\PostgreSQL\MH-Tables.sql` | `messages` DDL :146 |

</details>
