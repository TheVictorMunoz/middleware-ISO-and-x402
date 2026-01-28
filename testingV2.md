## Minimum Stress Test Instructions (ProofRails V2)

These instructions define the **minimum** stress test a dev must run against **ProofRails V2** (`middleware-ISO-and-x402`) and the exact artifacts they must report back.

This test focuses on the two core HTTP endpoints shown in the repo README:

* `POST /v1/iso/record-tip` (receipt creation) ([GitHub][1])
* `POST /v1/iso/verify` (bundle verification) ([GitHub][1])

> Important: avoid x402 “premium endpoints” during stress testing unless explicitly requested, since those are pay-per-use. ([GitHub][1])

---

### 1) Prerequisites

**Tools**

* Python 3 + pip
* Node.js (optional; UI not required for this test)
* `k6` (load testing tool)

**Repo setup**
Clone and run the backend using the Quick Start steps (as baseline): ([GitHub][1])

```bash
git clone https://github.com/TheVictorMunoz/middleware-ISO-and-x402.git
cd middleware-ISO-and-x402

pip install -r requirements.txt
alembic upgrade head
```

Start the API (use **no reload** for stress testing):

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --log-level info --access-log
```

(Quick Start uses `--reload`; do **not** use reload for stress tests.) ([GitHub][1])

---

### 2) Baseline sanity check (must pass)

Run once before load:

```bash
curl -i -X POST http://localhost:8000/v1/iso/record-tip \
  -H "Content-Type: application/json" \
  -d '{
    "tip_tx_hash": "0xabc123...",
    "chain": "flare",
    "amount": "1.00",
    "currency": "FLR",
    "sender_wallet": "0xSender...",
    "receiver_wallet": "0xReceiver...",
    "reference": "baseline-test"
  }'
```

Expected: HTTP success (200/201/202) and a response containing a receipt identifier / status.

Endpoint and payload shape are per README. ([GitHub][1])

---

### 3) Create load scripts (k6)

Create a folder:

```bash
mkdir -p stress
```

#### A) `stress/k6_record_tip.js` (required)

```javascript
import http from "k6/http";
import { check, sleep } from "k6";

export const options = {
  scenarios: {
    warmup: { executor: "constant-vus", vus: 5, duration: "30s" },
    ramp: {
      executor: "ramping-vus",
      startVUs: 5,
      stages: [
        { duration: "2m", target: 25 },
        { duration: "3m", target: 50 },
        { duration: "2m", target: 50 },
        { duration: "30s", target: 0 },
      ],
    },
    spike: {
      executor: "ramping-vus",
      startVUs: 0,
      stages: [
        { duration: "10s", target: 150 },
        { duration: "20s", target: 150 },
        { duration: "10s", target: 0 },
      ],
    },
  },
  thresholds: {
    http_req_failed: ["rate<0.02"],       // <2% failures
    http_req_duration: ["p(95)<1500"],    // p95 under 1.5s (local baseline)
  },
};

function randHex(n) {
  const chars = "abcdef0123456789";
  let s = "";
  for (let i = 0; i < n; i++) s += chars[Math.floor(Math.random() * chars.length)];
  return s;
}

export default function () {
  const url = "http://localhost:8000/v1/iso/record-tip";

  const payload = JSON.stringify({
    tip_tx_hash: "0x" + randHex(64),
    chain: "flare",
    amount: "1.00",
    currency: "FLR",
    sender_wallet: "0x" + randHex(40),
    receiver_wallet: "0x" + randHex(40),
    reference: "k6-" + __VU + "-" + Date.now(),
  });

  const res = http.post(url, payload, {
    headers: { "Content-Type": "application/json" },
  });

  check(res, {
    "status is 200/201/202": (r) => [200, 201, 202].includes(r.status),
  });

  sleep(0.2);
}
```

This script targets the repo’s `record-tip` endpoint as documented. ([GitHub][1])

#### B) `stress/k6_verify.js` (optional)

Only run this if you have **real bundle URLs/hashes**; otherwise it just measures validation failures.

```javascript
import http from "k6/http";
import { check, sleep } from "k6";

export const options = { vus: 20, duration: "2m" };

export default function () {
  const url = "http://localhost:8000/v1/iso/verify";
  const payload = JSON.stringify({ bundle_url: "https://ipfs.io/ipfs/QmExample..." });

  const res = http.post(url, payload, {
    headers: { "Content-Type": "application/json" },
  });

  check(res, { "not 5xx": (r) => r.status < 500 });
  sleep(0.2);
}
```

Verify endpoint is documented in the README. ([GitHub][1])

---

### 4) Run the stress test (required command)

In Terminal A (API logs):

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --log-level info --access-log | tee stress/api.log
```

In Terminal B (run k6):

```bash
k6 run stress/k6_record_tip.js | tee stress/k6_record_tip.out.txt
```

Optional (only if valid bundle URLs/hashes exist):

```bash
k6 run stress/k6_verify.js | tee stress/k6_verify.out.txt
```

---

### 5) Minimum monitoring to capture (required)

During the test, capture **system resource snapshots**:

**If running locally (non-docker):**

* Take at least 3 screenshots or saved outputs during:

  * warmup
  * sustained 50 VUs
  * spike

Suggested commands:

```bash
top
```

**If running via docker-compose:** (repo includes `docker-compose.yml`) ([GitHub][1])

```bash
docker stats | tee stress/docker_stats.txt
```

---

### 6) Pass/Fail criteria (minimum bar)

Mark the run as **PASS** if:

* `http_req_failed < 2%`
* No sustained 5xx burst during the 50 VU plateau
* No obvious memory runaway (RSS climbs continuously with no stabilization)

If it fails, record:

* the first load level where failure begins (e.g., at ~35 VUs, or during spike)
* the dominant failure mode (timeouts, 5xx, DB lock, etc.)

---

## 7) What the dev must report back

Create a short report (markdown is fine) with:

**A) Environment**

* Git SHA / branch
* OS + CPU cores + RAM
* DB type/config (whatever it uses by default)
* Any config changes (anchoring on/off, worker count, env vars)

**B) Results (from k6 output)**

* Peak RPS achieved
* p50 / p95 / p99 latency
* Failure rate (`http_req_failed`)
* Top 3 status codes and their counts

**C) First bottleneck**

* CPU-bound vs DB-bound vs external dependency
* Evidence: log lines + system stats snapshot(s)

**D) Artifacts (attach / link)**

* `stress/api.log`
* `stress/k6_record_tip.out.txt`
* (optional) `stress/k6_verify.out.txt`
* (optional) `stress/docker_stats.txt` or screenshots of `top`

---

