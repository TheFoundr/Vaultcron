# Vaultcron
VaultCron monitors your cron jobs and alerts you the moment one stops running. When it fails, AI tells you exactly why — not just that it did. Add one curl command to any script. Works with Python, Node, Ruby, Go, GitHub Actions, and Kubernetes. Free trial, no card needed.

# VaultCron — Cron Job Monitoring with AI Diagnosis

> The first cron monitor that tells you **why** your job failed, not just that it did.

[![VaultCron Status](https://img.shields.io/badge/status-live-brightgreen)](https://vaultcron.com)
[![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

---

## What is VaultCron?

Most deployed applications have scheduled jobs running silently in the background — database backups, email digests, billing retries, data syncs. When these jobs fail, there is no error message, no notification, no sign anything went wrong until the damage is already done.

**VaultCron monitors your cron jobs and alerts you the moment one stops running.**

When a job fails, our AI analyses your ping history, duration trends, and metadata payloads to tell you exactly what went wrong and how to fix it.

---

## Quick start (2 minutes)

### 1. Create a monitor

Sign up at [vaultcron.com](https://vaultcron.com) and create a monitor. Set your schedule and grace period. You get a unique ping URL:
```
https://vaultcron.com/api/ping/YOUR-MONITOR-ID
```

### 2. Add one line to your cron job
```bash
# Before VaultCron
0 2 * * * /home/user/backup.sh

# After VaultCron — pings only if script succeeds
0 2 * * * /home/user/backup.sh && curl -s https://vaultcron.com/api/ping/YOUR-ID
```

### 3. That's it

If VaultCron doesn't receive a ping within your expected window, you get an alert immediately.

---

## Features

### ⚡ AI-powered failure diagnosis
When a monitor goes down, click Diagnose. Our AI analyses your ping history, duration trends, log output, and metadata to give you a specific, actionable explanation — not a generic "job failed" message.

### 📦 Context-aware payload rules
Send metadata with your pings and set rules to alert when values are unexpected:
```bash
curl -s -X POST https://vaultcron.com/api/ping/YOUR-ID \
  -H "Content-Type: application/json" \
  -d '{"rows_processed": 0, "file_size_kb": 1024, "duration_ms": 3200}'
```

Set a rule: alert if `rows_processed < 1`. Catches silent successes — jobs that ran but did nothing.

### 🏥 Health scores
Every monitor gets a health score (A–F) based on uptime, timing consistency, duration trends, and recent history. See at a glance which jobs are degrading before they fail.

### 👥 Multi-worker monitoring
Track distributed jobs running across multiple servers simultaneously. VaultCron waits until all expected workers have pinged before marking the job complete.

### 📊 Sites monitoring
Real user performance monitoring for your websites. Track load times, JS errors, backend vs frontend vs network time, and get AI insights on what's slowing your pages down.

### 🌙 Overnight digest
Wake up to a summary of everything that happened overnight. Every monitor, every ping, every anomaly — one email at 7am.

### 🔔 Alert channels
- Email (all plans)
- Slack webhooks (Indie+)
- Custom webhooks (Studio+)
- SMS (add-on)

---

## Integration examples

### Bash / cron
```bash
# Simple heartbeat
0 2 * * * /scripts/backup.sh && curl -s https://vaultcron.com/api/ping/YOUR-ID

# With start and fail events
0 2 * * * curl -s https://vaultcron.com/api/ping/YOUR-ID/start && \
          /scripts/backup.sh && \
          curl -s https://vaultcron.com/api/ping/YOUR-ID || \
          curl -s -X POST https://vaultcron.com/api/ping/YOUR-ID/fail \
            -d '{"log": "backup failed"}'
```

### Python
```python
import requests

PING_URL = "https://vaultcron.com/api/ping/YOUR-MONITOR-ID"

try:
    requests.get(f"{PING_URL}/start", timeout=5)
    
    result = run_your_job()
    
    requests.post(PING_URL, json={
        "rows_processed": result.rows,
        "duration_ms":    result.duration,
    }, timeout=5)

except Exception as e:
    requests.post(f"{PING_URL}/fail", json={"log": str(e)}, timeout=5)
    raise
```

### Node.js
```javascript
const PING_URL = "https://vaultcron.com/api/ping/YOUR-MONITOR-ID"

await fetch(`${PING_URL}/start`).catch(() => {})

try {
  const result = await runYourJob()
  await fetch(PING_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ rows_processed: result.rows })
  }).catch(() => {})
} catch (err) {
  await fetch(`${PING_URL}/fail`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ log: err.message })
  }).catch(() => {})
  throw err
}
```

### Python decorator
```python
import requests
import functools
import time

def monitor(ping_url):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            requests.get(f"{ping_url}/start", timeout=5)
            start = time.time()
            try:
                result = func(*args, **kwargs)
                requests.post(ping_url, json={
                    "duration_ms": int((time.time() - start) * 1000)
                }, timeout=5)
                return result
            except Exception as e:
                requests.post(f"{ping_url}/fail", json={"log": str(e)}, timeout=5)
                raise
        return wrapper
    return decorator

@monitor("https://vaultcron.com/api/ping/YOUR-ID")
def nightly_backup():
    # your job here
    pass
```

### GitHub Actions
```yaml
name: Nightly backup
on:
  schedule:
    - cron: '0 2 * * *'

jobs:
  backup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Ping start
        run: curl -s https://vaultcron.com/api/ping/${{ secrets.VAULTCRON_PING_URL }}/start

      - name: Run backup
        run: ./scripts/backup.sh

      - name: Ping success
        if: success()
        run: curl -s https://vaultcron.com/api/ping/${{ secrets.VAULTCRON_PING_URL }}

      - name: Ping failure
        if: failure()
        run: |
          curl -s -X POST https://vaultcron.com/api/ping/${{ secrets.VAULTCRON_PING_URL }}/fail \
            -H "Content-Type: application/json" \
            -d '{"log": "workflow failed"}'
```

### Kubernetes CronJob
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: your-backup-image
            command: ["/bin/sh", "-c"]
            args:
              - |
                /scripts/backup.sh && \
                curl -s https://vaultcron.com/api/ping/YOUR-ID || \
                curl -s -X POST https://vaultcron.com/api/ping/YOUR-ID/fail
          restartPolicy: OnFailure
```

### Ruby
```ruby
require 'net/http'
require 'json'
require 'uri'

PING_URL = "https://vaultcron.com/api/ping/YOUR-ID"

def ping(path = "", body = nil)
  uri = URI("#{PING_URL}#{path}")
  http = Net::HTTP.new(uri.host, uri.port)
  http.use_ssl = true
  req = body ? Net::HTTP::Post.new(uri) : Net::HTTP::Get.new(uri)
  req['Content-Type'] = 'application/json'
  req.body = body.to_json if body
  http.request(req)
rescue => e
  # never let monitoring block your job
end

ping("/start")
begin
  run_job()
  ping("", { rows_processed: 1500 })
rescue => e
  ping("/fail", { log: e.message })
  raise
end
```

### Go
```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
)

const pingURL = "https://vaultcron.com/api/ping/YOUR-ID"

func ping(path string, body map[string]interface{}) {
    var req *http.Request
    if body != nil {
        b, _ := json.Marshal(body)
        req, _ = http.NewRequest("POST", pingURL+path, bytes.NewBuffer(b))
        req.Header.Set("Content-Type", "application/json")
    } else {
        req, _ = http.NewRequest("GET", pingURL+path, nil)
    }
    http.DefaultClient.Do(req) // fire and forget
}

func main() {
    ping("/start", nil)
    
    err := runJob()
    if err != nil {
        ping("/fail", map[string]interface{}{"log": err.Error()})
        fmt.Println("job failed:", err)
        return
    }
    
    ping("", map[string]interface{}{"rows_processed": 1500})
}
```

---

## API reference

### Ping endpoint
```
GET  https://vaultcron.com/api/ping/:id
POST https://vaultcron.com/api/ping/:id
```

Record a successful run. Accepts an optional JSON body with metadata.

### Start event
```
POST https://vaultcron.com/api/ping/:id/start
```

Signal that a job has started. Enables duration tracking and overlap detection.

### Fail event
```
POST https://vaultcron.com/api/ping/:id/fail
```

Explicitly report a failure. Include log output for AI diagnosis.
```bash
curl -s -X POST https://vaultcron.com/api/ping/YOUR-ID/fail \
  -H "Content-Type: application/json" \
  -d '{"log": "Error: database connection refused at line 42"}'
```

### Request body fields

| Field | Type | Description |
|-------|------|-------------|
| `log` | string | Log output. Shown in alerts and AI diagnosis. Max 10,000 chars. |
| `worker` | string | Worker ID for multi-worker jobs. |
| `[any]` | number | Any numeric field stored as metadata. Usable in payload rules. |

### Status badge
```markdown
![VaultCron](https://vaultcron.com/api/badge/YOUR-MONITOR-ID)
```

---

## Monitor statuses

| Status | Description |
|--------|-------------|
| `Waiting` | Created but no pings received yet |
| `Healthy` | Receiving pings within expected window |
| `Late` | Missed expected time, within grace period, no alert yet |
| `Down` | Grace period passed, alert fired |
| `Paused` | Manually paused, no alerts |

---

## Payload rules

Send any numeric data with your ping and set rules to alert when values are unexpected. Catches silent failures — jobs that ran but processed nothing.
```bash
curl -s -X POST https://vaultcron.com/api/ping/YOUR-ID \
  -H "Content-Type: application/json" \
  -d '{
    "rows_processed": 1542,
    "emails_sent": 847,
    "file_size_kb": 2048,
    "duration_ms": 3200
  }'
```

Then set rules:
- Alert if `rows_processed < 1`
- Alert if `emails_sent < 1`
- Alert if `file_size_kb < 100`

---

## Health scores

Every monitor is scored A–F based on:

| Factor | Weight | Description |
|--------|--------|-------------|
| Uptime | 40% | Percentage of expected pings received on time |
| Consistency | 30% | How consistent ping timing is |
| Duration | 20% | Whether job duration is stable or degrading |
| Trend | 10% | Whether the score is improving or worsening |

---

## Pricing

| | Free | Indie | Studio | Max |
|--|------|-------|--------|-----|
| Price | €0 | €19/mo | €34/mo | €89/mo |
| Monitors | 3 | 20 | 100 | Unlimited |
| AI diagnosis | ✗ | ✓ | ✓ | ✓ |
| Slack alerts | ✗ | ✓ | ✓ | ✓ |
| Sites monitoring | ✗ | ✗ | ✓ | ✓ |
| Team members | ✗ | ✗ | 3 | 5 |

[View full pricing →](https://vaultcron.com/pricing)

---

## Free tools

We built 100+ free tools for developers and SaaS founders — no signup required:

- [Cron Expression Explainer](https://vaultcron.com/tools/cron-expression-explainer)
- [Cron Expression Generator](https://vaultcron.com/tools/cron-expression-generator)
- [Unix Timestamp Converter](https://vaultcron.com/tools/unix-timestamp-converter)
- [JWT Decoder](https://vaultcron.com/tools/jwt-decoder)
- [Uptime Calculator](https://vaultcron.com/tools/uptime-calculator)
- [SLA Downtime Calculator](https://vaultcron.com/tools/sla-downtime-calculator)
- [Error Budget Calculator](https://vaultcron.com/tools/error-budget-calculator)
- [MRR Calculator](https://vaultcron.com/tools/mrr-calculator)
- [Runway Calculator](https://vaultcron.com/tools/runway-calculator)
- [Browse all 100+ tools →](https://vaultcron.com/tools)

---

## Documentation

Full documentation at [vaultcron.com/docs](https://vaultcron.com/docs)

- [Quickstart](https://vaultcron.com/docs/quickstart)
- [API reference](https://vaultcron.com/docs/api/ping)
- [Python integration](https://vaultcron.com/docs/integrations/python)
- [Node.js integration](https://vaultcron.com/docs/integrations/node)
- [GitHub Actions](https://vaultcron.com/docs/integrations/github-actions)
- [Kubernetes](https://vaultcron.com/docs/integrations/kubernetes)
- [Payload rules](https://vaultcron.com/docs/concepts/payload-rules)
- [AI diagnosis](https://vaultcron.com/docs/concepts/ai-diagnosis)
- [Health scores](https://vaultcron.com/docs/concepts/health-scores)

---

## Alternatives comparison

| | VaultCron | Cronitor | Healthchecks.io | Dead Man's Snitch |
|--|-----------|---------|-----------------|-------------------|
| AI diagnosis | ✓ | ✗ | ✗ | ✗ |
| Payload rules | ✓ | Partial | ✗ | ✗ |
| Sites monitoring | ✓ | ✗ | ✗ | ✗ |
| Health scores | ✓ | ✗ | ✗ | ✗ |
| Free tier | ✓ | ✓ | ✓ | ✗ |
| Starting price | €19/mo | $29/mo | $20/mo | $19/mo |

---

## Built with

- [Next.js 16](https://nextjs.org) — App Router
- [Supabase](https://supabase.com) — Database, Auth, RLS
- [Stripe](https://stripe.com) — Payments
- [OpenAI](https://openai.com) — AI diagnosis (GPT-4o)
- [Resend](https://resend.com) — Email alerts
- [Vercel](https://vercel.com) — Hosting and cron jobs
- [Tailwind CSS](https://tailwindcss.com) — Styling

---

## Support

- Documentation: [vaultcron.com/docs](https://vaultcron.com/docs)
- Email: hello@vaultcron.com
- Issues: [GitHub Issues](https://github.com/mercys1/vaultcron/issues)

---

## License

MIT — see [LICENSE](LICENSE) for details.

---

<p align="center">
  Built by a solo founder in Dublin 🇮🇪
  <br />
  <a href="https://vaultcron.com">vaultcron.com</a>
</p>
