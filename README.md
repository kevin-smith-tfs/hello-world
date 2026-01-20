Cool — we’ll build this as a **repeatable RCA notebook** with:

1. **Traffic normalized to bps** (so spikes are “real bandwidth”)
2. **Top VIP talker** (ranked during the spike window)
3. A clean **RCA flow** you can re-run for the next incident

Below is a notebook template you can copy section-by-section.

---

# RCA Notebook Layout (recommended)

## Section 0 — Set the incident window (do this first)

In the notebook UI, set the timeframe to:

* **Last 30 days** (to find the spike), then
* Change to a **tight window** around the spike (ex: 2–6 hours)

You’ll use the tight window for “top talker”.

---

# Section 1 — Interface bandwidth (bps) for ports 1.1 and 1.4

### 1A) Bytes IN → **bps**

(keep this line above, then paste the new one right under it)

```dql
fetch dt.entity.f5_interface
| filter entity.name == "1.1" or entity.name == "1.4"
| makeTimeseries value = rate(avg(com.dynatrace.extension.f5.bigip.sys.interface.stat.bytes.in.count)) * 8,
    resolution: 1m
```

### 1B) Bytes OUT → **bps**

```dql
fetch dt.entity.f5_interface
| filter entity.name == "1.1" or entity.name == "1.4"
| makeTimeseries value = rate(avg(com.dynatrace.extension.f5.bigip.sys.interface.stat.bytes.out.count)) * 8,
    resolution: 1m
```

📌 Why this is correct:

* Your metrics are counters → `rate()` converts to “per second”
* bytes/sec → `* 8` makes it **bits/sec**
* 1-minute buckets via `resolution: 1m`

---

# Section 2 — Are we dropping/errored during the spike?

### 2A) Errors in/out (rate)

```dql
fetch dt.entity.f5_interface
| filter entity.name == "1.1" or entity.name == "1.4"
| makeTimeseries value = rate(avg(com.dynatrace.extension.f5.bigip.sys.interface.stat.errors.in.count)),
    resolution: 1m
```

### 2B) Drops (rate)

```dql
fetch dt.entity.f5_interface
| filter entity.name == "1.1" or entity.name == "1.4"
| makeTimeseries value = rate(avg(com.dynatrace.extension.f5.bigip.sys.interface.stat.drops.in.count)),
    resolution: 1m
```

If these stay ~0 while bps spikes → it’s almost always **legit load**, not interface failure.

---

# Section 3 — System correlation (connections + droppedPacketRate)

### 3A) Client/server connections

```dql
timeseries
  clientConns = avg(com.dynatrace.extension.f5.bigip.sys.clientCurConns),
  serverConns = avg(com.dynatrace.extension.f5.bigip.sys.serverCurConns),
  resolution: 1m
```

### 3B) Dropped packet rate (system)

```dql
timeseries
  droppedPacketRate = avg(com.dynatrace.extension.f5.bigip.sys.droppedPacketRate),
  resolution: 1m
```

---

# Section 4 — Identify the TOP VIP talker (during the spike window)

## 4A) Rank VIPs by **peak requests/min** during the selected timeframe

Set the notebook timeframe to the spike window first.

```dql
timeseries vipReqs = avg(com.dynatrace.extension.f5.bigip.virtualserver.stat.tot.requests.count),
  by: { dt.entity.f5_virtualserver },
  resolution: 1m
| fieldsAdd vipName = entityName(dt.entity.f5_virtualserver)
| fieldsAdd peakReqs = arrayMax(vipReqs)
| sort peakReqs desc
| limit 10
| fields vipName, peakReqs
```

This gives you the **top 10 VIPs** by peak request rate in that exact incident window.

> If `entityName()` isn’t recognized in your tenant, replace those two lines with:

```dql
| fieldsAdd vipName = dt.entity.f5_virtualserver
```

You’ll still get the VIP entity IDs (workable for correlation).

---

## 4B) Plot the top 1–3 VIPs over time (visual confirmation)

Take the top VIP name/ID from 4A, then filter to it:

```dql
timeseries vipReqs = avg(com.dynatrace.extension.f5.bigip.virtualserver.stat.tot.requests.count),
  by: { dt.entity.f5_virtualserver },
  resolution: 1m
| filter entityName(dt.entity.f5_virtualserver) == "<PASTE_TOP_VIP_NAME_HERE>"
```

If the VIP request spike lines up with interface bps spike → that’s your **top talker RCA**.

---

# Section 5 — “Top talker summary” block (ticket-ready)

Once you have:

* spike timestamp
* interface bps peak
* top VIP + peakReqs
* drops/errors correlation

Use this structure:

* **What happened:** Interface 1.1/1.4 traffic spiked to X bps at TIME
* **Impact evidence:** drops/errors stayed near 0 (or spiked if they did)
* **Primary driver:** VIP `<name>` peaked at Y req/min at TIME (aligned)
* **Supporting evidence:** client/server conns increased to Z at TIME
* **Conclusion:** spike driven by VIP demand (or by saturation if drops/errors rose)

---

## Two quick checks to avoid false RCA

* Make sure the notebook timeframe is tight when running 4A (otherwise it ranks “top talkers” over 30 days, not during the incident).
* If VIP requests don’t correlate but interface bps does, the “top talker” might be **pool member traffic** or **mgmt/replication** traffic — then we pivot to pool/member metrics next.

---

If you paste the output of **Section 4A** (top 10 VIPs + peakReqs) and tell me the **spike time**, I’ll give you the exact “Top VIP over time” filter query and a clean RCA paragraph you can drop into a ticket.
