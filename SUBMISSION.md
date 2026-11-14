# Submission

- **Name:**
- **Date submitted:**
- **Repository link:**
- **Total time spent (real time):**

> Replies below are drafts written on the **exercise clock** (Monday 16 November 2026, Asia/Dubai). Keep your real time in the time log at the end. Put screenshots, log excerpts and query output in `evidence/` and link to them.

> **Required:** the Acknowledgement and Priority sections for **all five** items, and the full write-up (investigation log onwards) for **at least three** items of your choice. For an item you did not fully investigate, replace the remaining sections with "Not investigated — what I would check next: …".

- **Items I fully investigated, and why I chose them:**

---

## Incident 1 — `inbox/01-oasis-builders-upload-stuck.md`

### Acknowledgement

- **Sent at (exercise clock):** 09:25 (email received 09:05; P2 first-response target is 1 h, so due by 10:05)
- **Reply:**

> Subject: RE: URGENT - monthly invoice upload stuck on "Processing"
>
> Hello Rashid,
>
> Thank you for the detail. I understand your November file stayed on "Processing", that retrying led to "already exists" messages, and that you need to know which invoices have gone out so you can close the month. I know this is holding up your month-end, and I'm treating it as urgent.
>
> I'm checking exactly what was received for each of your invoice numbers now. **Please don't upload the file again for now**, so we avoid creating duplicates.
>
> I'll send you a list of what has been sent and what hasn't, and the safe next step, by **11:00 today**. If a call would help, tell me a good time.
>
> Regards,
> Nitheesh, Product Support, Invoicely Lite

### Priority and justification

- **Priority:** P2 (raised to P1 if the investigation shows other customers cannot upload either)
- **Why:** Invoices from this seller can still reach the ASP, so it is not P1 on the first test in `docs/sla-matrix.md`. But a whole month-end file is affected, with a close deadline and customers waiting, so I treated it as the top of P2. Any file that takes longer than about 60 s to process hits the same problem, so I will re-check the priority if more customers report it.
- **SLA deadlines I am working to:** first response 10:05 (sent 09:25) / next update 11:00 / resolution target 8 business hours, so 17:05 the same day.

### Investigation log

> Times are mapped onto the exercise clock (Monday 16 Nov 2026). The work itself was done in real time; see the time log at the end for real times.

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 09:30 | Read the email and the attached `oasis-builders-nov-2026.xlsx` (8 invoices, 600 lines) | File layout matches the template, so this is not a bad file |
| 09:40 | Read `importer.go`, `validate.go`, `directory.go` | Import runs inside one HTTP request. Validation makes one Peppol lookup per row, in sequence (`validate.go:165`) |
| 09:55 | Timed the ASP lookup with curl | About 0.2 s each (0.203 s, 0.203 s, 0.224 s) |
| 10:05 | Replayed the Oasis upload on a clean system | HTTP 201 after **120.7 s**, all 8 invoices Accepted, API restarts 0 ([replay output](evidence/01-oasis-replay-output.txt)) |
| 10:20 | Hypothesis: the API crashes on large files (link to the restart alert) | **Ruled out.** `restarts=0`, `oomkilled=false` during the replay |
| 10:30 | Reset the database and uploaded the file three times, 30 s apart, directly to the API | All returned 201 after about 121 s. Upload 1 = 1 Accepted / 7 Rejected, upload 2 = 8 Rejected, upload 3 = 7 Accepted / 1 Rejected. Each invoice number was Accepted exactly once, and the ASP received 8 documents ([per invoice](evidence/01-oasis-retry-per-invoice.txt), [ASP count](evidence/01-oasis-asp-received.txt)) |
| 10:45 | Read `route.ts` in the web app | The portal proxy stops waiting after 60 s (`UPSTREAM_TIMEOUT_MS`, line 7) |
| 10:50 | Uploaded through the portal (port 3000) | **HTTP 504 "Gateway Timeout" after exactly 60 s**; the API kept working in the background ([proxy test](evidence/01-oasis-via-portal-proxy.txt)) |
| 10:55 | Checked how the duplicate rule treats in-progress invoices | `store.go:300` ignores only Rejected invoices, and `importer.go:243` saves invoices as Parsed before validating them. The mid-run screenshot shows "Parsed 7, Validated 1" ([screenshot](evidence/01-oasis-portal-processing.png)) |

### Root cause

- **What happened:** A normal-sized monthly file takes about 2 minutes to process, but the portal stops waiting after 60 s and shows an error page. The import carries on in the API. Rashid retried, and each retry's invoices collided with invoices from the earlier uploads that were still being processed, producing "already exists" messages.
- **Where:**
  - `services/api/internal/validate/validate.go:165-166`: one Peppol lookup per row, no cache
  - `services/api/internal/importer/importer.go:193`: whole import runs inside the upload request
  - `web/app/api/[...path]/route.ts:7`: 60 s proxy timeout
  - `services/api/internal/store/store.go:300-302` together with `importer.go:243`: invoices still in progress count as duplicates
- **Why it happens:** The work grows with the number of rows (600 rows × 0.2 s = 120 s), the timeout in front of it is fixed at 60 s, and the portal gives no sign that the first upload is still running.
- **Evidence:** [replay output](evidence/01-oasis-replay-output.txt), [retry run](evidence/01-oasis-retry-repro.txt), [per-invoice outcomes](evidence/01-oasis-retry-per-invoice.txt), [ASP count](evidence/01-oasis-asp-received.txt), [portal proxy test](evidence/01-oasis-via-portal-proxy.txt), screenshots [processing](evidence/01-oasis-portal-processing.png), [completed](evidence/01-oasis-portal-completed-120s.png) and [Grafana](evidence/01-grafana-overview-oasis-replay.png)
- **Confidence and open questions:** High on the timing, the 504 and the duplicate mechanism, because I reproduced each one. Medium on Rashid's exact outcome, because his click timing was different from mine. I have not tested a file split into smaller parts yet.

### Customer resolution reply OR escalation note

- **Which one and why:** Both. The customer gets an update now, because he asked what was sent. I am also escalating to L3 because this is a product defect, not a data error I can fix myself.

> **Update to Rashid (sent 11:00):**
>
> Hello Rashid,
>
> Here is what I found.
>
> **What happened:** Your November file has 600 lines, and the portal needs about two minutes to check it. After one minute the portal stops waiting and shows an error, even though your file keeps being processed behind the scenes. Each time you uploaded again, the new copy ran into the earlier copy, which is why you saw "already exists". You did nothing wrong.
>
> **1. Which invoices have been sent?** [Before sending, check the ASP records and the portal for each invoice number and paste the list here: invoice number, status, ASP reference. "Accepted" means our delivery partner has received it and it is on its way to your customer. Only say that nothing was sent twice if the ASP records show it.]
>
> **2. What to do now:** Please don't upload the whole file again. The "already exists" messages on invoices that also show Accepted need no action. If any invoice has no Accepted copy, tell me and I'll confirm the safe way to send just those.
>
> I've passed the cause to our engineering team with all the details. I'll update you again by **14:00**, whatever the news. If a call would help, I'm available.
>
> Regards,
> Nitheesh

> **Escalation note (L3):**
>
> ### Escalation: Month-end uploads over about 60 s return 504 and cause duplicate-number rejections
>
> **Priority:** P2 — a seller's month-end file cannot be uploaded reliably; close deadline today.
> **Customer / seller:** Oasis Builders Supply LLC, seller id 2
> **Reported by / at:** Rashid Al Mansoori, email, 09:05 Mon 16 Nov
> **Affected:** every upload that takes longer than 60 s. Oasis file `oasis-builders-nov-2026.xlsx`: 8 invoices, 600 rows.
>
> **Symptoms (customer view):** upload sits on "Processing", then an error page; retries produce AE-BR-09 "already exists" with status Parsed or Validated.
>
> **Evidence:**
> - Web log: `POST /imports status 504 durationMs 59999 TimeoutError`
> - Import duration 120.7 s for 600 rows; participant lookup 0.2 s each (`validate.go:165`)
> - Three overlapping uploads left 8 invoices Accepted once each and 16 Rejected with AE-BR-09 (`evidence/01-oasis-retry-per-invoice.txt`)
> - ASP received 8 documents for sender 0235:1003182746
>
> **Steps to reproduce:**
> 1. Reset the system and upload `samples/oasis-builders-nov-2026.xlsx` for Oasis through http://localhost:3000.
> 2. After 60 s the portal returns 504. The import completes at about 120 s.
> 3. Upload again before it completes: AE-BR-09 against Parsed invoices.
>
> **What I have already tried / ruled out:** API crash or out-of-memory (restarts 0, no OOM kill).
>
> **Suspected cause (with confidence):** High. `route.ts:7` timeout against `validate.go:165` per-row lookup, plus `store.go:300` counting in-progress invoices as duplicates.
>
> **Workaround in place:** yes. The customer is told not to re-upload, and large files can be split into smaller parts (to be verified).
>
> **Timeline:**
> - 09:05 — email received
> - 09:25 — acknowledgement sent
> - 10:50 — 504 at 60 s reproduced through the portal
> - 11:00 — update sent to customer
>
> **Ask:** (1) cache or batch participant lookups per file, (2) make the import asynchronous or raise the timeout, (3) make the duplicate check ignore invoices from imports still being processed, or show a clear message. Please reply by 13:00.
> **Next customer update due:** 14:00

### Workaround offered

- Do not re-upload the whole file. Wait until the batch list shows no batch in "Processing".
- Send only invoices that have no Accepted copy. If an invoice is Validated or Failed, use its **Retry submission**. A Rejected invoice does not use up its number, so a corrected re-upload of just those invoices is allowed once nothing else is in flight.
- For future months, split the file into smaller uploads so each finishes inside 60 s. At 0.2 s per row that is roughly 250 rows or fewer. This is an estimate and still needs testing.

### Prevention ideas

- Cache the Peppol lookup per buyer within one import. That would take this file from 120 s to a few seconds.
- Process imports asynchronously and show real progress.
- Make the proxy timeout longer than the processing budget, or return immediately with an import id.
- Make the duplicate message say that another upload of this file is still running.
- Add a dashboard panel and an alert for import duration.

---

## Incident 2 — `inbox/02-alert-api-restarted-marina-bay.md`

### Acknowledgement

- **Sent at (exercise clock):** 09:52 (alert fired 09:42, Priya's message 09:50; P1 first-response target is 15–30 min, so due by 10:12 at the latest)
- **Sent to:** Priya Nair (reply in #cs-escalations), copy to support-oncall; engineering on-call paged at the same time
- **Reply:**

> Hi Priya, thanks for forwarding this. I'm owning it from support and I'll keep you posted.
>
> It's very likely the same problem. The API restarted 3 times in 15 minutes, and a restart can cause both the "502 Bad Gateway" errors and invoices waiting in "Validated". I'm confirming that now. I've set it as P1 and paged engineering on-call.
>
> For your customers right now: please ask them **not to upload the same file again** if they got a 502. Their invoices may already be in the portal, and a second upload can be rejected as a duplicate. I'll tell you exactly what's safe once I've checked.
>
> On Marina Bay for Wednesday: I don't have an honest answer yet, but I'll give you one by **13:00**, before your call. Next update from me by **10:30**.

### Priority and justification

- **Priority:** P1
- **Why:** The first question in `docs/sla-matrix.md` is "can customers get invoices to the ASP at all?". While the API keeps restarting, uploads fail with 502 and invoices stop moving for several customers at once, so the answer is no. The alert is marked critical/production, several customers are affected, and a go-live on Wednesday depends on it.
- **SLA deadlines I am working to:** first response by 10:12 (sent 09:52) / updates every 30–60 min (10:30, 11:30, 13:00 with the Marina Bay answer) / resolution or agreed workaround by 13:42 (4 h).

### Investigation log

> Times are mapped onto the exercise clock (Monday 16 Nov 2026). The work itself was done in real time; see the time log at the end for real times.

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 09:52 | Acknowledged Priya, paged engineering on-call (P1: page once impact is confirmed, don't wait for root cause) | — |
| 09:55 | Read `docker-compose.yml` | The api container is limited to 128 MB with no extra swap (`docker-compose.yml:60-61`) and restarts automatically (`:62`) |
| 10:00 | Hypothesis from Incident 1: Oasis's large file crashes the API | **Ruled out.** Oasis uploads alone (single and overlapping) caused 0 restarts |
| 10:05 | Replayed this morning's traffic (`loadgen load`: 4 customer files, then 240 peak uploads, 8 at a time) and sampled API memory every 5 s | **3 restarts in about 1 minute** during the peak, the same count as the alert. 24 uploads failed with `EOF` / `connection refused`, the source of the 502s ([load output](evidence/02-load-output.txt)) |
| 10:15 | Looked at the memory samples | Sawtooth: memory climbs from about 7 MiB to 100–121 MiB of 128 MiB, then drops to about 7 MiB as the restart count goes up ([memory samples](evidence/02-api-memory-restarts-during-load.txt)) |
| 10:20 | `docker inspect` showed `OOMKilled=false` | Not conclusive: the flag is reset when the container restarts. Checked cAdvisor instead |
| 10:22 | Prometheus: `container_oom_events_total{container_label_com_docker_compose_service="api"}` | **3**: three out-of-memory kills ([OOM metric](evidence/02-prometheus-oom-events.txt)) |
| 10:30 | Update sent to Priya | — |
| 10:35 | Read `importer/sources.go` and `importer.go` | Every upload is stored in memory (raw file, parsed workbook, invoices) and never removed (`sources.go:31-36`, `importer.go:251`) |
| 10:45 | Is it a leak or only a spike? Uploaded the small Ghaf file 40 times, one at a time | Memory rose steadily, about 1.2 MiB per 92 KB upload (14 → 63 MiB), and did not drop after 20 s idle. **A leak.** About 80–90 uploads fill 128 MB, which matches 240 peak uploads causing 3 restarts ([leak test](evidence/02-sequential-leak-test.txt)) |
| 10:55 | Checked what the restarts left behind in the database | 20 imports stuck in Processing, 62 invoices stuck in Parsed, 46 stuck in Validated ([DB state](evidence/02-db-state-after-restarts.txt)) |
| 11:00 | Listed affected customers | Desert Rose, **Marina Bay Hospitality**, Palm Coast Electronics, Zayed Port Logistics: 5 stuck imports each ([affected customers](evidence/02-affected-customers.txt)) |
| 11:05 | Read `submit.go` and `main.go` | The submission queue is an in-memory channel (`submit.go:65`). Invoices are only queued when an import finishes (`importer.go:281`), and nothing re-queues Validated invoices at start-up (`main.go:56-57`), so a restart strands them |
| 11:10 | Tested the documented workaround on one stuck import (`POST /v1/imports/{id}/retry`) | 5 Validated invoices re-queued and reached **Accepted**. The 1 Parsed invoice stayed stuck. The batch page's source preview is gone (`"available": false`) ([retry test](evidence/02-retry-workaround-test.txt)) |

### Root cause

- **What happened:** The API keeps a copy of every uploaded file in memory and never releases it. During the morning peak, memory reached the container's 128 MB limit and the API was killed and restarted three times. Uploads in progress at each restart failed (the 502s). Invoices that had been checked but not yet sent were held in a queue that also lives only in memory, so they were lost from it and stayed in "Validated".
- **Where:**
  - `services/api/internal/importer/sources.go:31-36`: `SourceCache.Put` adds an entry per upload; nothing ever removes one (`StoredAt` is set but never used)
  - `services/api/internal/importer/importer.go:251`: every import is put in the cache
  - `docker-compose.yml:60-62`: 128 MB limit, no swap, automatic restart
  - `services/api/internal/submit/submit.go:65` and `importer.go:281`: in-memory submission queue; `main.go:56-57`: nothing re-queues stranded invoices at start-up
- **Why it happens:** Memory grows with every upload, for every customer, until the process is killed. The more uploads, the sooner it happens, so the problem gets worse as volume grows.
- **Evidence:** [load output](evidence/02-load-output.txt), [memory samples](evidence/02-api-memory-restarts-during-load.txt), [OOM metric](evidence/02-prometheus-oom-events.txt), [leak test](evidence/02-sequential-leak-test.txt), [DB state](evidence/02-db-state-after-restarts.txt), [affected customers](evidence/02-affected-customers.txt), [retry test](evidence/02-retry-workaround-test.txt)
- **Screenshots:** [alert firing](evidence/02-grafana-alert-firing.png), [OOM kills in Prometheus](evidence/02-prometheus-oom-events.png), [API restarts in Loki](evidence/02-loki-api-restarts.png), [container memory, restarts and Go heap](evidence/02-grafana-container-memory-restarts.png), [stuck imports in the portal](evidence/02-portal-stuck-processing.png), [Marina Bay batch stuck](evidence/02-portal-marina-bay-batch-stuck.png), [leak-test uploads](evidence/02-portal-leak-test-uploads.png)
  - *Clock note:* Grafana screenshots show the browser's time zone (UTC+5:30), the portal shows Dubai time (UTC+4), and logs are UTC. The three restarts are at 07:27:09, 07:27:38 and 07:28:09 in Grafana, which is 01:57:09, 01:57:38 and 01:58:09 UTC. All three sources agree.
- **Confidence and open questions:** High. Reproduced the restart count, measured the growth per upload, and confirmed the OOM kills from cAdvisor. Open: I did not profile the heap, so the 1.2 MiB per upload is measured from container memory, not attributed byte by byte to the cache. The same load also left 737 invoices in "Submitted"; that is a separate problem (HTTP 429 handling), covered in Incident 4. It also affects Marina Bay (batch `mbh-morning-234.xlsx`: Submitted 6, no ASP reference, [screenshot](evidence/02-portal-stuck-processing.png)), so it matters for the go-live too.

### Customer resolution reply OR escalation note

- **Which one and why:** Escalation to L3 (P1, engineering on-call paged at 09:52), because the fix is a code and infrastructure change and the stuck Parsed invoices and Processing imports need a data repair that support cannot do through the product. I keep owning the updates to Priya and the affected customers.

> **Update to Priya (sent 10:30):**
>
> Hi Priya, here's where we are.
>
> **What we know:** It is the same problem. The API ran out of memory and restarted 3 times this morning. Each restart caused the 502s, and some invoices that were waiting to be sent were left in "Validated". Four customers are affected that I can see: Desert Rose, Marina Bay Hospitality, Palm Coast Electronics and Zayed Port Logistics.
>
> **What we're doing:** Engineering on-call has the full details and is working on the cause. I'm checking each affected customer's invoices.
>
> **What it means for customers:**
> - Invoices stuck in "Validated" can be sent now: on the batch page, click **Retry stuck invoices**. I've tested this and they go through. I can also do it for them.
> - A few invoices are stuck at an earlier step and can't be retried from the portal. Engineering will fix these; customers don't need to do anything.
> - Please ask customers not to re-upload files that failed with a 502. The invoice numbers may already be in use, and a re-upload can be rejected as a duplicate.
>
> Next update by 11:30.

> **Update to Priya — Marina Bay go-live (sent 13:00):**
>
> Hi Priya, here's my honest view for your call.
>
> **Marina Bay was affected this morning:** 5 of their uploads were interrupted by the restarts. Separately, some of their other invoices show "Submitted" but have no ASP reference, which means I can't yet confirm they reached the network. I'm investigating that second issue now; it is also affecting another customer. None of their invoices are lost: they are all in the portal. I'll confirm the status of each one, and that nothing went out twice, by 15:00.
>
> **Wednesday:** There are two problems to fix before I'd call go-live safe. First, the API holds every uploaded file in memory, and at this morning's volume it fills up after roughly 80–90 uploads. Second, the "Submitted with no ASP reference" issue above. Neither is specific to Marina Bay, but a go-live adds volume, so the risk is higher on Wednesday unless both fixes are in place. Engineering is working on them, but I don't have confirmed fix dates yet, so I can't promise Wednesday is safe today.
>
> **What I suggest you tell their GM:** we found the cause of this morning's interruption, none of their invoices were lost, and we will confirm go-live readiness by **Tuesday 17 Nov, 12:00** once the fix is deployed and tested. I'll give you a go / no-go by then, and I'm happy to join your call if that helps.

> **Escalation note (L3, paged 09:52, note sent 11:15):**
>
> ### Escalation: API out-of-memory restarts during peak uploads (3 OOM kills), invoices stranded
>
> **Priority:** P1 — uploads fail with 502 and invoices stop moving for several customers; Marina Bay Hospitality goes live Wed 18 Nov.
> **Customer / seller:** Desert Rose (1), Marina Bay Hospitality (3), Palm Coast Electronics (7), Zayed Port Logistics (8); potentially all customers
> **Reported by / at:** Grafana alert `InvoicelyApiRestarted` 09:42; Priya Nair (CSM) 09:50
> **Affected:** 20 imports stuck in Processing, 62 invoices stuck in Parsed, 46 in Validated (`evidence/02-db-state-after-restarts.txt`)
>
> **Symptoms (customer view):** intermittent "502 Bad Gateway" on upload; invoices sitting in "Validated"; batch page loses its source-row preview.
>
> **Evidence:**
> - Prometheus: `container_oom_events_total{container_label_com_docker_compose_service="api"}` = 3
> - Memory sawtooth from ~7 MiB to 100–121 MiB, then a restart (5 s samples, `evidence/02-api-memory-restarts-during-load.txt`)
> - Sequential test: +1.2 MiB per 92 KB upload, not released when idle (`evidence/02-sequential-leak-test.txt`)
> - loadgen: `20 x Post "http://api:8080/v1/imports": EOF`, `4 x … connect: connection refused`
> - SQL: `SELECT status, count(*) FROM imports GROUP BY status;` → Processing 20
>
> **Steps to reproduce:**
> 1. `docker compose --profile tools run --rm --no-deps tools loadgen load`
> 2. Watch `docker stats invoicely-lite-api-1` and the restart count. Restarts start after about 80 uploads.
> 3. Faster: upload any small file repeatedly and watch memory rise without falling.
>
> **What I have already tried / ruled out:** a single large file (Oasis, 600 rows) does not cause restarts. Short-term spike alone is ruled out by the sequential test.
>
> **Suspected cause (with confidence):** High. `services/api/internal/importer/sources.go:31-36`: unbounded `SourceCache` with no eviction, filled at `importer.go:251`, against the 128 MB limit at `docker-compose.yml:60-61`. Stranded invoices because the queue is in memory only (`submit.go:65`), with no re-queue at start-up (`main.go:56-57`).
>
> **Workaround in place:** partly. Validated invoices: "Retry stuck invoices" works (tested). Parsed invoices and Processing imports: no workaround from the product.
>
> **Timeline:**
> - 09:42 — alert fired (3 restarts in 15 min)
> - 09:50 — Priya reports 502s and invoices stuck in Validated
> - 09:52 — Priya acknowledged; engineering on-call paged
> - 10:22 — OOM kills confirmed from cAdvisor
> - 10:30 — update to Priya
> - 10:45 — leak confirmed with the sequential upload test
>
> **Ask:**
> 1. **Now:** a mitigation to stop the restarts today (for example an eviction limit or TTL on `SourceCache`, or a temporary higher memory limit while the fix is made). Please confirm the plan by 11:45.
> 2. **Data repair:** finish or reset the 20 Processing imports and the 62 Parsed invoices. Parsed invoices also block their invoice numbers (AE-BR-09), so customers cannot re-upload them.
> 3. **Fix:** bound the cache or move sources out of memory; re-queue Validated invoices at start-up.
> 4. **Go / no-go for Marina Bay (Wed 18 Nov):** I need engineering's view by Tue 17 Nov 10:00.
>
> **Next customer update due:** 11:30 (Priya); 13:00 Marina Bay answer.

### Workaround offered

- **Validated invoices:** "Retry stuck invoices" on the batch page (or `POST /v1/imports/{id}/retry`). Tested: stuck invoices reached Accepted.
- **Parsed invoices / imports stuck in Processing:** no customer workaround; engineering data repair requested. Customers should not re-upload, because the stuck invoice numbers would be rejected as duplicates.
- **To reduce restarts until the fix:** none that support can apply. A memory increase or a scheduled restart only delays it, and a restart strands more invoices, so this is engineering's call.

### Prevention ideas

- Bound the source cache (size limit or TTL) or store uploaded files in the database or on disk.
- Make the submission queue durable: workers pick up Validated invoices from the database, so a restart loses nothing.
- On start-up, mark imports left in Processing as Failed with a clear message, so customers know to act.
- Alerts: API memory above 80% of its limit; `container_oom_events_total` increasing; invoices in Validated or Parsed for longer than 15 minutes.
- Load-test with a realistic morning peak before each go-live.

---

## Incident 3 — `inbox/03-al-noor-fuel-br-co-15.md`

### Acknowledgement

- **Sent at (exercise clock):**
- **Reply:**

> …

### Priority and justification

- **Priority:**
- **Why:**
- **SLA deadlines I am working to:**

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| | | |

### Root cause

- **What happened:**
- **Where** (component, `file:line`):
- **Why it happens:**
- **Evidence:**
- **Confidence and open questions:**

### Customer resolution reply OR escalation note

- **Which one and why:**

> …

### Workaround offered

### Prevention ideas

---

## Incident 4 — `inbox/04-bluepeak-sahara-not-received.md`

### Acknowledgement

- **Sent at (exercise clock):**
- **Reply:**

> …

### Priority and justification

- **Priority:**
- **Why:**
- **SLA deadlines I am working to:**

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| | | |

### Root cause

- **What happened:**
- **Where** (component, `file:line`):
- **Why it happens:**
- **Evidence:**
- **Confidence and open questions:**

### Customer resolution reply OR escalation note

- **Which one and why:**

> …

### Workaround offered

### Prevention ideas

---

## Incident 5 — `inbox/05-ghaf-cafe-date-error.md`

### Acknowledgement

- **Sent at (exercise clock):**
- **Reply:**

> …

### Priority and justification

- **Priority:**
- **Why:**
- **SLA deadlines I am working to:**

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| | | |

### Root cause

- **What happened:**
- **Where** (component, `file:line`):
- **Why it happens:**
- **Evidence:**
- **Confidence and open questions:**

### Customer resolution reply OR escalation note

- **Which one and why:**

> …

### Workaround offered

### Prevention ideas

---

## Overall

### Prioritisation

The order in which you worked the five items, and why. Did you change your mind as you learned more?

### What I would do differently

### What I learned / what I would need to learn

Be specific: concepts, tools or parts of the system that were new to you, how you got up to speed, and what you would study next.

### Time log (real time)

| Date | From–to | What I worked on |
|---|---|---|
| | | |

### AI usage disclosure

Which tools you used (or "none"), what for, and how you checked their output against the system.
