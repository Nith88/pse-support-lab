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
- **Confidence and open questions:** High on the timing, the 504 and the duplicate mechanism, because I reproduced each one. Medium on Rashid's exact outcome, because his click timing was different from mine. The invoice list in the 11:00 update comes from my reproduction, where the portal and the ASP records agree one to one on all 8 message ids ([sent list](evidence/01-oasis-sent-list.txt)); on a live shift I would run the same two checks on his data right before sending. I have not tested a file split into smaller parts yet.

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
> **1. Which invoices have been sent?** All 8 of your November invoices have been sent, each exactly once. I checked both our portal and the records of our delivery partner (the ASP, the service that passes invoices to your customers). "Accepted" means the ASP has received the invoice and it is on its way to your customer; the ASP reference is its confirmation number.
>
> | Invoice | Status | ASP reference (first 8 characters) |
> |---|---|---|
> | OBS-2611-001 | Accepted | 03b4746d |
> | OBS-2611-002 | Accepted | e52ec6af |
> | OBS-2611-003 | Accepted | 52fb2a88 |
> | OBS-2611-004 | Accepted | dd139e6d |
> | OBS-2611-005 | Accepted | 9ce96746 |
> | OBS-2611-006 | Accepted | b714ed71 |
> | OBS-2611-007 | Accepted | 127440ff |
> | OBS-2611-008 | Accepted | 3e07176d |
>
> The other entries you see marked "already exists" are the extra copies from your repeat uploads. They were stopped by the portal and never sent, so your customers will not receive duplicates.
>
> **2. What to do now:** Nothing more is needed for November: all 8 invoices are out, so you can tell your controller the period can close. Please don't upload the file again; the "already exists" messages need no action. For next month, I'll send you advice on splitting the file before you upload, so this doesn't happen again.
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
>
> Nitheesh, Product Support

### Priority and justification

- **Priority:** P1
- **Why:** The first question in `docs/sla-matrix.md` is "can customers get invoices to the ASP at all?". While the API keeps restarting, uploads fail with 502 and invoices stop moving for several customers at once, so the answer is no. The alert is marked critical/production, several customers are affected, and a go-live on Wednesday depends on it.
- **SLA deadlines I am working to:** first response by 10:12 (sent 09:52) / updates no more than 60 min apart: 10:30, 11:15, 12:15, 13:00 (Marina Bay answer), 13:45 / resolution or agreed workaround by 13:42 (4 h).
- **Status at the 13:42 resolution target:** a workaround is in place and tested for invoices stuck in Validated. Invoices stuck in Parsed and imports stuck in Processing still need an engineering data repair, and the restarts will recur until the memory fix is deployed. So it **stays P1**. I will lower it to P2 once engineering's mitigation stops the restarts, and tell Priya when I do.

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
| 10:30 | **Update 1** to Priya | Confirmed cause of the 502s; advice unchanged; next update 11:15 |
| 10:35 | Read `importer/sources.go` and `importer.go` | Every upload is stored in memory (raw file, parsed workbook, invoices) and never removed (`sources.go:31-36`, `importer.go:251`) |
| 10:45 | Is it a leak or only a spike? Uploaded the small Ghaf file 40 times, one at a time | Memory rose steadily, about 1.2 MiB per 92 KB upload (14 → 63 MiB), and did not drop after 20 s idle. **A leak.** About 80–90 uploads fill 128 MB, which matches 240 peak uploads causing 3 restarts ([leak test](evidence/02-sequential-leak-test.txt)) |
| 10:55 | Checked what the restarts left behind in the database | 20 imports stuck in Processing, 62 invoices stuck in Parsed, 46 stuck in Validated ([DB state](evidence/02-db-state-after-restarts.txt)) |
| 11:00 | Listed affected customers | Desert Rose, **Marina Bay Hospitality**, Palm Coast Electronics, Zayed Port Logistics: 5 stuck imports each ([affected customers](evidence/02-affected-customers.txt)) |
| 11:05 | Read `submit.go` and `main.go` | The submission queue is an in-memory channel (`submit.go:65`). Invoices are only queued when an import finishes (`importer.go:281`), and nothing re-queues Validated invoices at start-up (`main.go:56-57`), so a restart strands them |
| 11:10 | Tested the documented workaround on one stuck import (`POST /v1/imports/{id}/retry`) | 5 Validated invoices re-queued and reached **Accepted**. The 1 Parsed invoice stayed stuck. The batch page's source preview is gone (`"available": false`) ([retry test](evidence/02-retry-workaround-test.txt)) |
| 11:15 | **Update 2** to Priya with the affected customers and the tested workaround; full escalation note sent to engineering | — |
| 11:30 | Message to the four affected customers (sent with Priya, see below) | — |
| 12:00 | Checked Marina Bay's batches ahead of the 13:00 answer | Batch `mbh-morning-234.xlsx`: **Submitted 6, no ASP reference**. The same pattern as Sahara's report (Incident 4, received 11:30), so a second problem affects Marina Bay ([screenshot](evidence/02-portal-stuck-processing.png)) |
| 12:15 | **Update 3** to Priya | Second issue flagged; Marina Bay answer still due 13:00 |
| 13:00 | **Update 4**: Marina Bay go-live answer to Priya | — |
| 13:42 | Resolution target reached | Workaround in place for Validated invoices only; stays P1 (see Priority) |
| 13:45 | **Update 5** to Priya | Status at the resolution target |

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

- **Which one and why:** Escalation to L3 (P1, engineering on-call paged at 09:52, full note at 11:15), because the fix is a code and infrastructure change and the stuck Parsed invoices and Processing imports need a data repair that support cannot do through the product. I keep owning the updates to Priya and the affected customers.

> **Update 1 to Priya (sent 10:30):**
>
> Hi Priya, here's where we are.
>
> **What we know:** It is the same problem. The API ran out of memory and was restarted automatically 3 times this morning, and each restart interrupted the uploads in progress, which is where the 502s came from.
>
> **What we're doing:** Engineering on-call has been paged and has the details. I'm now working out why memory runs out and which customers' invoices were left waiting.
>
> **What it means for customers:** the advice from my first message still stands: please don't re-upload files that failed with a 502.
>
> Next update by **11:15**.
>
> Nitheesh, Product Support

> **Update 2 to Priya (sent 11:15):**
>
> Hi Priya, an update with something customers can do.
>
> **What we know:** The API holds every uploaded file in memory and doesn't release it, so it runs out after roughly 80–90 uploads and restarts. Four customers have invoices that were left waiting: Desert Rose, Marina Bay Hospitality, Palm Coast Electronics and Zayed Port Logistics.
>
> **What we're doing:** Engineering has the full write-up and I've asked for a plan to stop the restarts by 11:45.
>
> **What it means for customers:**
> - Invoices stuck in "Validated" can be sent now: on the batch page, click **Retry stuck invoices**. I've tested this and they go through. I can also do it for them.
> - A few invoices are stuck at an earlier step and can't be retried from the portal. Engineering will fix these; customers don't need to do anything.
> - Please still ask customers not to re-upload files that failed with a 502. The invoice numbers may already be in use, and a re-upload can be rejected as a duplicate.
>
> I'll send the four customers a short note at 11:30 (text below) unless you'd rather send it yourself. Next update by **12:15**.
>
> Nitheesh, Product Support

> **Message to the affected customers (sent 11:30 to Desert Rose, Marina Bay Hospitality, Palm Coast Electronics and Zayed Port Logistics, with Priya in copy):**
>
> Subject: Interrupted uploads this morning — what to do
>
> Hello,
>
> Some of your uploads this morning were interrupted, and you may have seen a "502 Bad Gateway" error. This was a problem on our side, and our engineers are working on it.
>
> Your invoices are not lost. Here's what to do:
> - If a batch shows invoices in **"Validated"**, open the batch and click **Retry stuck invoices**. They will be sent. If you prefer, reply and I'll do it for you.
> - If some invoices show **"Parsed"**, or a batch still says **"Processing"**, please leave them. We'll fix these on our side; you don't need to do anything.
> - Please **don't upload the same file again**. The invoice numbers are already in the portal, and a second upload would be rejected as a duplicate.
>
> I'll update you by **14:00** today. If anything looks wrong, reply to this email.
>
> Nitheesh, Product Support, Invoicely Lite

> **Update 3 to Priya (sent 12:15):**
>
> Hi Priya, a short update.
>
> **What we know:** While checking Marina Bay's batches I found a second problem: some of their invoices show "Submitted" but have no ASP reference (the confirmation number we get when the network receives an invoice). It looks the same as an issue Sahara Medical reported at 11:30, which I'm also investigating.
>
> **What we're doing:** Engineering is working on the memory problem, and I've added this second issue to what they need to look at.
>
> **What it means:** I'm still on track to give you the Marina Bay answer by **13:00**.
>
> Nitheesh, Product Support

> **Update 4 to Priya — Marina Bay go-live (sent 13:00):**
>
> Hi Priya, here's my honest view for your call.
>
> **Marina Bay was affected this morning:** 5 of their uploads were interrupted by the restarts. Separately, some of their other invoices show "Submitted" but have no ASP reference, which means I can't yet confirm they reached the network. None of their invoices are lost: they are all in the portal. I'll confirm the status of each one, and that nothing went out twice, by 15:00.
>
> **Wednesday:** There are two problems to fix before I'd call go-live safe. First, the API holds every uploaded file in memory, and at this morning's volume it fills up after roughly 80–90 uploads. Second, the "Submitted with no ASP reference" issue above. Neither is specific to Marina Bay, but a go-live adds volume, so the risk is higher on Wednesday unless both fixes are in place. Engineering is working on them, but I don't have confirmed fix dates yet, so I can't promise Wednesday is safe today.
>
> **What I suggest you tell their GM:** we found the cause of this morning's interruption, none of their invoices were lost, and we will give you a clear **go or no-go for Wednesday by Tuesday 17 Nov, 12:00**. I'm happy to join your call if that helps.
>
> Next update by **13:45**.
>
> Nitheesh, Product Support

> **Update 5 to Priya (sent 13:45):**
>
> Hi Priya, status at the four-hour mark.
>
> **What we know:** Customers can send invoices stuck in "Validated" with **Retry stuck invoices**. Invoices stuck at the earlier step, and the restarts themselves, still need engineering's fix, so I'm keeping this at the highest priority.
>
> **What we're doing:** I'm chasing engineering on-call for the mitigation plan I asked for by 11:45, and for the repair of the stuck invoices, which I've asked for by 15:00. I'll pass on their plan and timing as soon as I have it; I won't guess a time before they confirm it.
>
> **What it means:** No change for customers: retry Validated invoices, don't re-upload.
>
> Next update by **14:45**, and the Marina Bay invoice check by 15:00 as promised.
>
> Nitheesh, Product Support

> **Escalation note (L3, on-call paged 09:52; full note sent 11:15):**
>
> ### Escalation: API out-of-memory restarts during peak uploads (3 OOM kills), invoices stranded
>
> **Priority:** P1 — uploads fail with 502 and invoices stop moving for several customers; Marina Bay Hospitality goes live Wed 18 Nov.
> **Customer / seller:** Desert Rose Trading (seller id 1), Marina Bay Hospitality (3), Palm Coast Electronics (7), Zayed Port Logistics (8); potentially all customers
> **Reported by / at:** Grafana alert `InvoicelyApiRestarted` to support-oncall, 09:42; Priya Nair (CSM) in #cs-escalations, 09:48, forwarded 09:50
> **Affected:** since the first restart, at or after 09:27 (the alert counts 3 restarts in the 15 min before 09:42). 20 imports stuck in Processing, 62 invoices stuck in Parsed, 46 in Validated (`evidence/02-db-state-after-restarts.txt`). Example: Marina Bay import `ecaace7f-c1ce-4254-bc4d-d14642599b2d` (`mbh-morning-238.xlsx`): invoices `MBH-1116-GYN6-238-01` to `-05` in Validated, `-06` in Parsed.
>
> **Symptoms (customer view):** intermittent "502 Bad Gateway" on upload; invoices sitting in "Validated"; batch page loses its source-row preview.
>
> **Evidence:**
> - Payload excerpt: n/a (not invoice-specific)
> - ASP response / error code: n/a for the restarts (the HTTP 429 issue is escalated separately under Incident 4)
> - Logs (Loki): `{service="api"} |= "api starting"` → three start-ups at 01:57:09, 01:57:38 and 01:58:09 UTC in the reproduction; loadgen: `20 x Post "http://api:8080/v1/imports": EOF`, `4 x … connect: connection refused`
> - Metrics: Prometheus `container_oom_events_total{container_label_com_docker_compose_service="api"}` = 3; dashboard "Invoicely Lite – Overview", Containers row (Container memory, Container restarts, API Go heap in use), reproduction window 01:50–02:05 UTC
> - Memory: sawtooth from ~7 MiB to 100–121 MiB, then a restart (5 s samples, `evidence/02-api-memory-restarts-during-load.txt`); sequential test +1.2 MiB per 92 KB upload, not released when idle (`evidence/02-sequential-leak-test.txt`)
> - Database: `SELECT status, count(*) FROM imports GROUP BY status;` → Processing 20; invoices by status → Parsed 62, Validated 46
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
> - 09:48 — Priya reports 502s and invoices stuck in Validated (forwarded 09:50)
> - 09:52 — Priya acknowledged; engineering on-call paged
> - 10:22 — OOM kills confirmed from cAdvisor
> - 10:30 — update 1 to Priya
> - 10:45 — leak confirmed with the sequential upload test
> - 11:10 — retry workaround tested on Validated invoices
> - 11:15 — update 2 to Priya; this note sent
>
> **Ask:**
> 1. **Acknowledge the page** by 10:07 (15 min). I will chase on-call directly if there is no acknowledgement.
> 2. **Mitigation today** to stop the restarts (for example an eviction limit or TTL on `SourceCache`, or a temporary higher memory limit while the fix is made). Please confirm the plan by **11:45**.
> 3. **Data repair:** finish or reset the 20 Processing imports and the 62 Parsed invoices, by **15:00 today** so I can confirm to customers. Parsed invoices also block their invoice numbers (AE-BR-09), so customers cannot re-upload them.
> 4. **Fix:** bound the cache or move sources out of memory; re-queue Validated invoices at start-up. Please give a deploy estimate by **17:00 today**.
> 5. **Go / no-go view for Marina Bay** (go-live Wed 18 Nov) by **Tue 17 Nov 10:00**, so I can answer Priya by 12:00.
>
> **Next customer update due:** 12:15 (Priya); 13:00 Marina Bay answer; 14:00 affected customers.

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

- **Sent at (exercise clock):** 10:40 (email received 10:15; P2 first-response target is 1 h, so due by 11:15. Sent between P1 actions on Incident 2, after the 10:30 update to Priya)
- **Reply:**

> Subject: RE: Invoices rejected with BR-CO-15 – November fuel invoices
>
> Dear Fatima,
>
> Thank you for the detailed message and for attaching the file. I understand that several of your November fuel invoices were rejected with BR-CO-15, while others prepared in exactly the same way were accepted, and that your external auditor has asked for an explanation.
>
> BR-CO-15 is a check made by our delivery partner: the invoice total including VAT must equal the total before VAT plus the VAT. One point that may help already: those totals are **not taken from your file**. Our portal calculates them from your invoice lines. So I'm starting by checking how the portal calculated the totals on your rejected invoices, compared with the accepted ones.
>
> In the meantime:
> - The rejected invoices were **not sent** to your customers, and their invoice numbers are **not used up**, so they can still be issued under the same numbers once we know the cause.
> - Please **don't change your quantities, prices or line amounts** to make the totals fit. Your dispatch figures should stay as they are until I've confirmed where the difference comes from.
>
> I'll update you by **13:00 today**. Once I know the cause, I'll also give you a written explanation you can share with your auditor.
>
> Kind regards,
> Nitheesh, Product Support, Invoicely Lite

### Priority and justification

- **Priority:** P2
- **Why:** A subset of one customer's invoices (3 of 6 in the replay of this morning's traffic, [load output](evidence/02-load-output.txt)) is rejected, while the rest of the portal works. That is "subset of invoices rejected" in `docs/sla-matrix.md`. The auditor review raises the business impact, but there is no same-day legal or payment deadline in the email, and the rejected invoices can be issued later under the same numbers. If the cause turns out to be in how the portal calculates totals, it would affect every customer with 3-decimal prices, and I would re-check the priority then.
- **SLA deadlines I am working to:** first response by 11:15 (sent 10:40) / updates every 2–4 h: 13:00, 16:00, then Tuesday 09:00 / resolution target 8 business hours, so Tuesday 17 Nov 08:15 (7 h 45 min on Monday until 18:00, then the clock pauses until 08:00 Tuesday).

### Not investigated — what I would check next

**Why I did not fully investigate it on this shift:** Incidents 2 and 4 were P1 (invoices not reaching the ASP for several customers, a go-live and a payment run), and Incident 1 was a month-end close. This one is P2, the rejected invoices are held safely and can be issued later under the same numbers, and the customer has been told not to change anything. So I gave the deep investigation time to the P1s and kept Al Noor on its SLA updates.

**What I already know (not yet investigated):**
- In the replay of this morning's traffic, Al Noor's file gave **3 Accepted and 3 Rejected** ([load output](evidence/02-load-output.txt), import `dbf4f404-57aa-4d10-8808-7ca3d87f918f`).
- Per `docs/template-field-guide.md`, invoice totals are calculated by the portal from the lines, not read from the file, and `Item net price*` may have 3 decimals ("Fuel and other quantity-priced goods often use 3 decimals").
- My working hypothesis, **unconfirmed**: a rounding difference in how the portal adds up line amounts, VAT per category and the invoice totals when prices have 3 decimals. The total including VAT is then off by a fils or so on some invoices but not others. Which invoices fail would depend on the numbers on each one, which would explain "prepared in exactly the same way" but different results.

**What I would check next, in order:**
1. **The rejected invoices themselves.** For import `dbf4f404-…`: `SELECT invoice_number, status, total_excl_vat, total_vat, total_incl_vat, errors FROM invoices WHERE import_id = '…' ORDER BY invoice_number;` Then the full ASP response on one rejected invoice (`GET /v1/invoices/{id}`, field `aspResponse`) to see the exact amounts the ASP compared.
2. **The payload the portal sent.** On the invoice page, open "PINT AE payload (JSON)" and compare the totals `taxExclusiveAmount`, `taxAmount` and `taxInclusiveAmount` (defined in `pint.go:53-56`) with the line amounts. Then recalculate by hand: sum of line net amounts, VAT at 5%, rounded to 2 decimals at each step.
3. **How the portal calculates totals.** Read `services/api/internal/pint/pint.go` (the mapping from rows to the PINT AE invoice): where it rounds (per line, per VAT category, or only at the end), and whether it uses the line net amount from the file or recalculates quantity × price with 3 decimals.
4. **How the ASP checks BR-CO-15.** Read the BR-CO-15 check in `services/asp-mock/main.go`: whether it compares exact values or allows a rounding tolerance, and to how many decimals.
5. **Compare with what worked before.** Al Noor's October file (seeded import, 4 invoices, all Accepted): were the prices 3-decimal too, or does the difference only appear with certain values?
6. **Decide the next step from the result:**
   - If the portal's calculation is inconsistent (for example rounding the sum differently from the parts), it is a product defect. I would escalate to L3 with the hand calculation as evidence, and tell Fatima clearly that the error was ours, not in her figures, with a written explanation for the auditor.
   - If the file has a real inconsistency (for example a line amount that doesn't match quantity × price beyond what AE-BR-08 allows), I would explain exactly which line and how to correct it.
   - Either way: confirm how the rejected invoices can be issued under the same numbers, and test that before telling her.

**Customer updates while it is open:** 13:00 (what I've checked and what's next), 16:00, then Tuesday 09:00 at the latest, each with the next update time, per the P2 cadence.

---

## Incident 4 — `inbox/04-bluepeak-sahara-not-received.md`

### Acknowledgement

- **Sent at (exercise clock):** 11:45 (email received 11:30; first set as P2, first-response target 1 h, so due by 12:30. Re-prioritised to P1 at 12:05, see below)
- **Reply:**

> Subject: RE: Sahara Medical - invoices "Submitted" but buyer not receiving them
>
> Hello Daniel,
>
> Thank you for the clear report, and for holding off on the resend. I understand that your connector sent Sahara Medical's 40 invoices this morning and got a success response, the portal shows them as "Submitted", but Emirates Palm Hospital has only received a few, and many invoices have no ASP reference. I know the hospital's payment run is this week, so I'm treating this as urgent.
>
> I'm checking now, invoice by invoice, what the network actually received. **Please keep holding the resend** until I confirm what's safe, so the hospital doesn't receive anything twice.
>
> I'll update you by **12:45 today** with what I've found. If a call would help after that, I'm happy to arrange one.
>
> Regards,
> Nitheesh, Product Support, Invoicely Lite

### Priority and justification

- **Priority:** P2 at first (11:45), **raised to P1 at 12:05**
- **Why:**
  - *P2 at 11:45:* from the email, one customer had a subset of invoices not delivered. The rest of the portal was working, so it matched "subset of invoices affected" in `docs/sla-matrix.md`.
  - *P1 from 12:05:* the logs showed the cause is not specific to Sahara. Any customer that sends more than 8 invoices at once has the rest silently left in "Submitted", never delivered. The database shows 769 such invoices across 5 customers, including Marina Bay Hospitality before its go-live. That is "invoices not reaching the ASP for one or more customers", so P1, as the matrix requires when new evidence shows wider impact. It is also silent: the portal says "Submitted", and nothing alerts. Business impact: the hospital's payment run this week.
  - I told Daniel about the priority change in the 12:45 update, as the matrix asks.
- **SLA deadlines I am working to:** first response by 12:30 (sent 11:45) / as P1 from 12:05, updates no more than 60 min apart: 12:45, 13:30, 14:30 / resolution or agreed workaround by 16:05 (4 h from the P1 decision).
- **Status at the resolution target:** no customer workaround exists (see Workaround). The 32 Sahara invoices need an engineering repair, so it stays P1 until they are delivered and the fix is deployed.

### Investigation log

> Times are mapped onto the exercise clock (Monday 16 Nov 2026). This work ran alongside Incident 2 (P1, updates at 12:15, 13:00, 13:45). The work itself was done in real time; see the time log at the end. The Sahara invoice numbers in the evidence carry the replay tag (for example `SMS-1116-001-GYN6`) in the load run, and the plain numbers (`SMS-1116-001`) in my direct reproduction.

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 11:45 | Acknowledged Daniel | — |
| 11:50 | Checked Sahara's batch in the database | 40 invoices: **8 Accepted** with an ASP reference, **32 Submitted with an empty ASP reference** ([DB statuses](evidence/04-sahara-db-statuses.txt)) |
| 11:55 | ASP records: `GET /v1/documents?sender=0235:1006453719` | The ASP received **only the 8** Accepted invoices. The other 32 never reached the network, although the portal says "Submitted" ([ASP records](evidence/04-sahara-asp-received.txt)) |
| 12:00 | API and ASP logs for those invoices | API: **8 × HTTP 200 and 32 × HTTP 429**, all within the same second. ASP: 32 × `rate limit exceeded`, `retry_after_s: 1`. The API logs each 429 at **INFO** level, not as an error ([logs](evidence/04-sahara-logs-429.txt)) |
| 12:00 | (Checked Marina Bay's batches for Incident 2) | Batch `mbh-morning-234.xlsx`: Submitted 6, no ASP reference: the same pattern |
| 12:05 | Read the ASP's fair-use policy and `submit.go` | ASP allows a burst of **8 documents per sender, then 2 per second** (`asp-mock/main.go:35-37`). The API marks an invoice "Submitted" *before* sending (`submit.go:114`), and its response handling covers 200, 422 and 5xx but **not 429** (`submit.go:133-155`). So a throttled invoice is left in "Submitted" for good, and the `Retry-After` header is ignored |
| 12:05 | Scope across all customers; **raised to P1** | **769** invoices in "Submitted" with no ASP reference across 5 customers (Zayed Port 178, Marina Bay 177, Desert Rose 177, Palm Coast 173, Sahara 64). The ASP's own counter shows `throttled = 769`, the same number ([scope and metrics](evidence/04-scope-and-metrics.txt)) |
| 12:10 | Escalated to L3. Engineering on-call was already engaged on Incident 2, so I linked the two | — |
| 12:20 | Reproduced with `samples/sahara-medical-batch-1116.xlsx` on a quiet system | The same result: 8 Accepted, 32 Submitted with no ASP reference. So it is not caused by the morning load; a single 40-invoice batch is enough ([reproduction](evidence/04-sahara-repro-and-resend.txt)) |
| 12:25 | Status history of a stuck invoice; tried **Retry submission** | History ends at "Submitted – Sent to ASP", with no ASP response stored. Retry is refused: **HTTP 409** "only invoices in status Validated or Failed can be retried". Batch retry re-queues 0 |
| 12:30 | Simulated Bluepeak's resend: uploaded the same file again | All 40 rejected with AE-BR-09 ("already exists … status 'Accepted'" or "'Submitted'"). The resend added **nothing** at the ASP. So a resend is safe from duplicates, but it does not deliver the 32 either: they are **trapped** |
| 12:35 | Checked the monitoring | `invoicely_submissions_total` only has the outcomes accepted / rejected / failed (`metrics.go:66-70`), and a 429 increments none of them. So "Submissions by outcome" hides the problem. The only alert rules are for API restarts and 5xx (`rules.yml:10, 50`). Only the ASP-side panel shows throttling. Also, the time-series panel "ASP responses by HTTP status" shows **no 429s at all**: the `code="429"` series is only created when the first 429 arrives, so its first sample is already 32 and `increase()` counts that as 0 (only the codes for submission outcomes are pre-created, `metrics.go:66-70`). The first burst of throttling after any API restart is invisible on that chart ([Prometheus check](evidence/04-prometheus-429-series-start.txt), [screenshot](evidence/04-grafana-submissions-row.png)) |
| 12:40 | Checked who the 40 invoices are for (the portal showed `SMS-1116-032` going to Coral Reef Resorts LLC) | The batch has **two buyers**: 35 invoices for Emirates Palm Hospital and 5 for **Coral Reef Resorts LLC**. In this morning's batch, the hospital is missing 28 and **Coral Reef Resorts is missing 4** (016, 024, 032, 040). They have not reported it, so they probably have not noticed. Also, which 8 get through depends on timing: 001–008 in the morning run, but 001–007 and 009 in my reproduction (008 throttled). So every invoice has to be checked individually ([buyers](evidence/04-sahara-buyers.txt)) |
| 12:45 | Update to Daniel (priority change, answers to both questions) | — |

### Root cause

- **What happened:** The ASP limits how fast one sender can submit: 8 documents at once, then 2 per second. When a batch is bigger than that, the ASP answers the extra invoices with "429 Too Many Requests – try again in 1 second". Invoicely Lite does not handle that answer. It had already marked each invoice "Submitted" before sending, so the 32 refused invoices stayed "Submitted" with no ASP reference, were never sent again, and look delivered in the portal. Only the 8 that got through were delivered: 7 to Emirates Palm Hospital and 1 to a second buyer in the same batch, Coral Reef Resorts.
- **Where:**
  - `services/api/internal/submit/submit.go:114`: status set to Submitted before the ASP has answered
  - `services/api/internal/submit/submit.go:133-155`: response switch has no case for HTTP 429 (or any other unexpected code), so the invoice is left as it is and `Retry-After` is ignored
  - `services/api/main.go:56`: 4 workers send as fast as they can, so any batch over 8 invoices per seller exceeds the burst
  - `services/asp-mock/main.go:35-37` and `:274-283`: the ASP's documented fair-use limit and its 429 response (`docs/architecture.md`, "ASP integration": "the document has not been received and should be sent again later")
  - `services/api/internal/metrics/metrics.go:66-70`: no submission outcome for throttling, so the dashboard and alerts do not see it
- **Why it happens:** The ASP's rule is documented, but the client only handles success, business rejection and server errors. A throttled invoice falls through the switch, with nothing logged at warning or error level.
- **Evidence:** [DB statuses](evidence/04-sahara-db-statuses.txt), [ASP records](evidence/04-sahara-asp-received.txt), [API and ASP logs](evidence/04-sahara-logs-429.txt), [reproduction, retry and resend test](evidence/04-sahara-repro-and-resend.txt), [scope across customers and metrics](evidence/04-scope-and-metrics.txt), [buyers in the batch](evidence/04-sahara-buyers.txt)
- **Screenshots:** [Sahara batch in the portal](evidence/04-portal-sahara-batch.png), [invoice status history](evidence/04-portal-sahara-invoice-history.png), [resend rejected as duplicates](evidence/04-portal-sahara-resend-rejected.png), [Grafana "Submissions to ASP" row](evidence/04-grafana-submissions-row.png) (*Submissions by outcome* shows only 8 accepted; *ASP: documents by outcome (ASP side)* shows 8 accepted and 32 throttled; *ASP responses by HTTP status (since API start)* shows HTTP 200 = 13 and HTTP 429 = 32, but the time-series panel above it shows no 429s, see the investigation log at 12:35)
- **Confidence and open questions:** High. The counts match exactly in three places (8 accepted / 32 throttled in the logs, the database and the ASP; 769 stuck across customers against 769 throttled at the ASP), and I reproduced it on a quiet system. Which invoices get through depends on the order the workers send them, so the list differs between runs; the invoice numbers in the update to Daniel are from this morning's batch (shown without the replay tag `-GYN6` that the load tool adds). Open: I have not confirmed with engineering the safest way to release the 32 (for example, set them to Failed and use Retry). Before any resend, each invoice must be checked against the ASP records so nothing is delivered twice.

### Customer resolution reply OR escalation note

- **Which one and why:** Both. This is a product defect, and the 32 stuck invoices need a data change that neither support nor the customer can make through the product (Retry is refused for "Submitted"), so I escalated to L3. I keep owning the conversation with Daniel and Sahara.

> **Update to Daniel (sent 12:45):**
>
> Hello Daniel,
>
> Here is what I found. I've also raised this to our **highest priority**, because the same problem affects other customers too.
>
> **1. Is Invoicely losing invoices?** The problem is on our side, and I'm sorry for the trouble it's causing. Of Sahara's 40 invoices, **8 were delivered** to the network (those are the ones with an ASP reference, the confirmation number the network gives us). **The other 32 were not delivered.** The network asked us to slow down because so many invoices arrived at once, and our portal did not send them again, but it still showed them as "Submitted". They are **not lost**: all 32 are stored in the portal and will be sent once the fix is in. Your connector did nothing wrong; the HTTP 201 you received means the file was accepted correctly.
>
> I checked each invoice in our portal against the network's records:
> - **Delivered (8):** SMS-1116-001 to SMS-1116-008.
> - **Not delivered (32):** SMS-1116-009 to SMS-1116-040.
>
> **Please note: this batch is for two customers, not one.** Most of the invoices are for Emirates Palm Hospital, but 5 are for **Coral Reef Resorts LLC**, and 4 of those were not delivered either (SMS-1116-016, -024, -032 and -040). Coral Reef Resorts may not have noticed yet, so Sahara may want to let them know.
>
> **2. Should you resend the batch?** **No, please don't.** I tested it: a resend is rejected as a duplicate, so it won't deliver the 32 (and won't duplicate the 8). We'll send the 32 from our side. Before sending each one, we'll check it against the network's records so the hospital doesn't receive anything twice.
>
> **What happens next:** our engineering team has all the details and is working on both the fix and releasing the 32 invoices. I don't have a time from them yet, so I won't guess one. Given the hospital's payment run, I've asked for the 32 to be handled first.
>
> I'll update you by **13:30**, and I'll confirm by email when each invoice is delivered.
>
> Regards,
> Nitheesh, Product Support, Invoicely Lite

> **Escalation note (L3, sent 12:10, updated 12:35):**
>
> ### Escalation: HTTP 429 from the ASP not handled — throttled invoices stuck in "Submitted", never delivered
>
> **Priority:** P1 — invoices silently not reaching the ASP for 5 customers (769 invoices); Sahara Medical's buyer has a payment run this week; Marina Bay Hospitality goes live Wed 18 Nov.
> **Customer / seller:** Sahara Medical Supplies FZE (seller id 5), via Bluepeak ERP Integrations. Also Zayed Port Logistics (8), Marina Bay Hospitality (3), Desert Rose Trading (1), Palm Coast Electronics (7)
> **Reported by / at:** Daniel Okafor (Bluepeak, Sahara's integration partner), email to support, 11:30
> **Affected:** since this morning's uploads; any batch with more than 8 invoices for one seller. Sahara import `77098132-f99b-4f86-9cf2-e635f97f2529` (load run): 32 invoices from `SMS-1116-009-GYN6` onwards; reproduction import `56652e1a-faca-4d5c-ba9d-443d4318f316`. 769 invoices in total (`evidence/04-scope-and-metrics.txt`)
>
> **Symptoms (customer view):** invoices show "Submitted" with an empty ASP reference; the buyer never receives them; no error anywhere; Retry is refused; a resend is rejected as a duplicate.
>
> **Evidence:**
> - Payload excerpt: n/a (payloads are valid; the 8 sent first from the same batch were accepted)
> - ASP response: `HTTP 429`, `Retry-After: 1`, body `{"error":"rate_limited","message":"Too many documents from sender 0235:1006453719. Retry after 1s."}`
> - Logs (`docker compose logs api | grep '"asp response"' | grep SMS-1116`): 8 × `"status_code":200`, 32 × `"status_code":429`, all at 01:54:55 UTC in the reproduction, logged at INFO. asp-mock: 32 × `rate limit exceeded`, `retry_after_s: 1`
> - Metrics: `sum by (code) (invoicely_asp_responses_total)` shows 429s; `sum by (outcome) (invoicely_submissions_total)` has no outcome for them; ASP side `sum by (outcome) (asp_documents_total)` → throttled 769. Dashboard "Invoicely Lite – Overview", row "Submissions to ASP"
> - Database: `SELECT s.name, count(*) FROM invoices i JOIN sellers s ON s.id = i.seller_id WHERE i.status = 'Submitted' AND i.asp_message_id = '' GROUP BY 1;` → 769 across 5 sellers
>
> **Steps to reproduce:**
> 1. On a quiet system, upload `samples/sahara-medical-batch-1116.xlsx` for seller 5 (`curl -F file=@… -F seller_id=5 http://localhost:8080/v1/imports`).
> 2. After a few seconds: 8 Accepted, 32 Submitted with empty `asp_message_id`.
> 3. `POST /v1/invoices/{id}/retry` on a Submitted invoice → 409.
>
> **What I have already tried / ruled out:** load-related only (ruled out: reproduced on a quiet system); payload problem (ruled out: same batch, first 8 accepted); invoice-level retry and batch retry (refused / re-queues 0); resend of the file (rejected AE-BR-09, nothing delivered).
>
> **Suspected cause (with confidence):** High. `services/api/internal/submit/submit.go:133-155` has no case for 429, and `submit.go:114` sets Submitted before the response. Throttled invoices are left in Submitted and never retried.
>
> **Workaround in place:** No. Customers and support cannot release Submitted invoices through the product.
>
> **Timeline:**
> - 11:30 — Daniel's email received
> - 11:45 — acknowledged (P2)
> - 12:00 — 429s found in the logs
> - 12:05 — scope confirmed (769 across 5 customers); raised to P1
> - 12:10 — this escalation; linked to Incident 2 (same on-call)
> - 12:30 — confirmed that retry and resend do not help
>
> **Ask:**
> 1. **Release the 32 Sahara invoices first** (payment run this week): for each, confirm it is absent from the ASP records, then resubmit (for example set to Failed and retry). Please agree the approach by **13:15** and complete it by **15:00** today.
> 2. Then the remaining 737 invoices for the other 4 customers, Marina Bay first (go-live Wed). Plan by **15:00**.
> 3. **Fix:** handle 429 by waiting for `Retry-After` and retrying (keep the invoice Validated or set it to Failed, never leave it Submitted); pace submissions per seller to the ASP's limit; treat any unexpected status as Failed. Deploy estimate by **17:00** today.
> 4. **Monitoring:** add a "throttled" submission outcome and an alert on invoices in Submitted for more than 15 minutes.
>
> **Next customer update due:** 13:30 (Daniel); confirmation to Daniel as each invoice is delivered.

### Workaround offered

- **For the 32 stuck invoices:** none that the customer or support can apply. Retry is refused for "Submitted" (409) and a resend is rejected (AE-BR-09). Engineering repair requested, with a check against the ASP records first so nothing is sent twice.
- **Told Daniel:** do not resend; the 8 delivered invoices are safe; we will send the 32 from our side.
- **Possible interim for future batches (to be tested with engineering first, not offered to the customer yet):** the ASP's bucket refills at 2 documents per second, so batches of at most 8 invoices sent at least 5 seconds apart should not be throttled. I worked this out from the code (`asp-mock/main.go:35-37`); I have not tested it.

### Prevention ideas

- Handle HTTP 429 properly: honour `Retry-After`, retry automatically, and never leave an invoice in Submitted without an ASP answer.
- Only mark an invoice Submitted once the ASP has actually received it, or add a separate "Waiting to retry" status that customers can see.
- Pace submissions per seller to the ASP's fair-use limit instead of sending as fast as the workers can.
- Treat any unexpected ASP response as Failed (retryable), and log it at warning level or above.
- Add a "throttled" outcome to `invoicely_submissions_total`, and alert on invoices in Submitted for more than 15 minutes, and on 429 rate.
- Pre-create the `invoicely_asp_responses_total` series for the expected codes (200, 422, 429, 5xx) at start-up, so the first 429s after a restart show on the dashboard.
- Let Retry work on invoices stuck in Submitted with no ASP reference, after a check against the ASP records.

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
