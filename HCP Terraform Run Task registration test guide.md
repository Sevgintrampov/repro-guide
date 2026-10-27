# HCP Terraform Run Task registration: test guide

How to test, without any third-party integration, what HTTP response an HCP Terraform Run Task endpoint must return for registration to succeed.

## Purpose and background

Registering a Run Task makes HCP Terraform send a synthetic verification request (`stage: "test"`, `access_token: "test-token"`, placeholder callback URL) to the task URL. A customer's Run Task integration (Blink) could not be registered, and the callback to the placeholder URL returned 401 and 404.

This guide sets up a tiny endpoint you control, registers it as a Run Task, and changes only the HTTP status code it returns. That isolates the cause.

**Results from our run (2026-10-23 and 2026-10-26):**

| Endpoint response | Registration result |
| --- | --- |
| 200 immediately, no callback | 201 Created (registered) |
| 200 after a 30 s delay | 201 Created (registered) |
| 200, then PATCH to the callback (got 401) | 201 Created (registered) |
| 202 immediately | 400 Bad Request, empty `detail` |
| 201 immediately | 400 Bad Request, empty `detail` |

Conclusions:

- Registration does not need a callback. The 401 and 404 on the placeholder callback are not the cause.
- Registration accepted a plain 200 and rejected 201 and 202. We did not test other 2xx codes.
- The public docs say the endpoint should return `200 OK`. They do not document the `test` stage or the registration handshake.

## What you need

- A Mac or Linux machine (on Windows, use WSL) with **Python 3** (`python3 --version`). On a Mac without it, run `xcode-select --install` or `brew install python`.
- **cloudflared** to expose your local endpoint over public HTTPS (free, no account needed for quick tunnels).
- Your HCP Terraform **organization name**.
- An HCP Terraform **API token** that can manage run tasks (a user token from an organization owner: Settings, then Tokens).
- Three terminal windows or tabs: A (tunnel), B (receiver), C (commands).

## Step 1: Install cloudflared

macOS with Homebrew:

```bash
brew install cloudflared
cloudflared --version
```

macOS without Homebrew (use `uname -m`: `arm64` is Apple Silicon, `x86_64` is Intel):

```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-darwin-arm64.tgz -o cloudflared.tgz
tar -xzf cloudflared.tgz
chmod +x cloudflared
./cloudflared --version
```

For Intel, replace `arm64` with `amd64` in the URL, and run it later as `./cloudflared`. If macOS blocks it, run `xattr -d com.apple.quarantine ./cloudflared`.

Linux (Debian or Ubuntu):

```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o cloudflared.deb
sudo dpkg -i cloudflared.deb
```

**Check:** `cloudflared --version` prints a version number.

## Step 2: Create the receiver script

Create a folder and save the script below as `receiver.py` in it:

```bash
mkdir ~/rt-probe
cd ~/rt-probe
nano receiver.py
```

Paste the code into nano, save with Ctrl+O then Enter, and exit with Ctrl+X. Saving through an editor is more reliable than pasting a long heredoc into the terminal, which can garble the text.

The script returns the HTTP status you set with `CODE` (default 200). `MODE` adds optional behavior: `slow` waits 30 seconds before answering, and `ack_callback` also PATCHes the supplied callback URL after responding.

```python
import json
import os
import time
import urllib.request
import urllib.error
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

MODE = os.environ.get("MODE", "ack")
STATUS = int(os.environ.get("CODE", "200"))


def log(*a):
    print(*a, flush=True)


def do_callback(body):
    try:
        p = json.loads(body)
        payload = {
            "data": {
                "type": "task-results",
                "attributes": {"status": "passed", "message": "probe"},
            }
        }
        req = urllib.request.Request(
            p["task_result_callback_url"],
            method="PATCH",
            data=json.dumps(payload).encode(),
            headers={
                "Authorization": "Bearer " + p["access_token"],
                "Content-Type": "application/vnd.api+json",
            },
        )
        r = urllib.request.urlopen(req)
        log("=== CALLBACK status", r.status)
    except urllib.error.HTTPError as e:
        log("=== CALLBACK status", e.code, e.read().decode())
    except Exception as e:
        log("=== CALLBACK error", repr(e))


class H(BaseHTTPRequestHandler):
    def do_POST(self):
        t0 = time.time()
        body = self.rfile.read(int(self.headers.get("Content-Length", 0)))
        log("\n=== REQUEST", time.strftime("%Y-%m-%d %H:%M:%S"), "MODE=" + MODE, "CODE=" + str(STATUS))
        for k, v in self.headers.items():
            log(k + ": " + v)
        log(body.decode())
        if MODE == "slow":
            time.sleep(30)
        self.send_response(STATUS)
        self.send_header("Content-Length", "0")
        self.end_headers()
        log("=== RESPONDED", STATUS, "after", round(time.time() - t0, 2), "s")
        if MODE == "ack_callback":
            do_callback(body)

    def log_message(self, *a):
        pass


log("listening on :8080, MODE =", MODE, "CODE =", STATUS)
ThreadingHTTPServer(("0.0.0.0", 8080), H).serve_forever()
```

**Check:** `python3 -m py_compile receiver.py && echo OK` prints `OK`.

## Step 3: Start the tunnel (terminal A)

```bash
cloudflared tunnel --url http://localhost:8080
```

It prints a URL such as `https://random-words.trycloudflare.com`. Copy it and keep this terminal open. The URL changes every time the tunnel restarts.

## Step 4: Start the receiver (terminal B)

```bash
cd ~/rt-probe
python3 receiver.py
```

**Check:** it prints `listening on :8080, MODE = ack CODE = 200`.

## Step 5: Verify the tunnel works before involving HCP (terminal C)

```bash
curl -i -X POST https://YOUR-TUNNEL-URL -d '{"hello":"world"}'
```

**Check:** the response starts with `HTTP/2 200` and the request appears in terminal B.

## Step 6: Register the receiver as a Run Task (terminal C)

Set your variables:

```bash
export ORG="your-org-name"
export TFC_TOKEN="your-token"
export TUNNEL="https://YOUR-TUNNEL-URL"
```

Then create the task. The command is on a single line to avoid terminal paste problems. Each task needs a unique `name`:

```bash
curl -s -i -X POST "https://app.terraform.io/api/v2/organizations/$ORG/tasks" -H "Authorization: Bearer $TFC_TOKEN" -H "Content-Type: application/vnd.api+json" -d "{\"data\":{\"type\":\"tasks\",\"attributes\":{\"name\":\"probe-200\",\"url\":\"$TUNNEL\",\"category\":\"task\",\"enabled\":true}}}"
```

Note the HTTP status and body of the response, and check terminal B to see the request HCP sent.

- **201** with a task `id` means registration succeeded.
- **400** with `{"errors":[{"status":"400","title":"bad request","detail":""}]}` means registration failed.

## Step 7: Run the test matrix

For each row, stop the receiver in terminal B with Ctrl+C, restart it with the command shown, and run the Step 6 command again with a new task name. The tunnel can stay up.

| Run | Receiver command | Task name | Expected |
| --- | --- | --- | --- |
| 1 | `python3 receiver.py` | `probe-200` | 201 Created |
| 2 | `CODE=202 python3 receiver.py` | `probe-202` | 400 |
| 3 | `CODE=201 python3 receiver.py` | `probe-201` | 400 |
| 4 | `MODE=slow python3 receiver.py` | `probe-slow` | 201 Created (receiver answers after about 30 s) |
| 5 | `MODE=ack_callback python3 receiver.py` | `probe-callback` | 201 Created, and terminal B shows `=== CALLBACK status 401` |

## Step 8: Read the results

| What you see | What it means |
| --- | --- |
| Only a plain 200 registers | HCP requires exactly 200. An integration that returns any other status, even another 2xx, cannot be registered. |
| 200 registers but the integration does not | Check the integration's real response (see the next section). |
| The receiver log shows no request at all | A network or URL problem (tunnel closed, wrong `TUNNEL` value), not the protocol. |
| The task is created, but the callback shows 401 | Expected. The `test-token` and callback URL in the synthetic event are placeholders. |

In the captured request, `User-Agent` is `TFC/1.0`, `Content-Type` is `application/json`, and `X-Tfc-Task-Signature` is empty because no HMAC key was set. Each registration attempt arrived once, with no retries.

## Step 9: Test a real integration (optional)

If you can reach the integration's webhook URL, send it the same synthetic event and look at the status, headers and timing it returns:

```bash
curl -i -X POST "WEBHOOK_URL" -H "Content-Type: application/json" -w "\n\nHTTP status: %{http_code}\nTotal time: %{time_total}s\n" -d '{"payload_version":1,"access_token":"test-token","stage":"test","task_result_id":"taskrs-xxxxxxxxxxxxxxxx","task_result_enforcement_level":"test","task_result_callback_url":"https://app.terraform.io/api/v2/task-results/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/callback","organization_name":"test-org","workspace_name":"test-workspace","capabilities":{"outcomes":true}}'
```

This triggers the integration's workflow once per run. In our case the Blink webhook returned a 201, which matches the failures above. If you cannot run the command yourself (the integration runs in a customer's environment), ask the customer to run it and send you the full output.

## Workaround when the integration cannot return 200

Put a thin proxy in front of the integration. HCP posts to the proxy, the proxy immediately returns 200, and it forwards the unchanged request body to the real webhook. Real runs should still work because the payload carries the real `access_token` and callback URL, so the integration calls back to HCP directly. A Cloudflare Worker or an AWS Lambda would do. This workaround has not been tested against real runs.

## Step 10: Clean up

Stop the tunnel and the receiver with Ctrl+C. Then list and delete the probe tasks you created:

```bash
curl -s "https://app.terraform.io/api/v2/organizations/$ORG/tasks" -H "Authorization: Bearer $TFC_TOKEN" | grep -o '"id":"task-[A-Za-z0-9]*"'

curl -s -X DELETE "https://app.terraform.io/api/v2/tasks/TASK_ID" -H "Authorization: Bearer $TFC_TOKEN"
```

Replace `TASK_ID` with each id from the list, in the form `task-XXXXXXXXXXXXXXXX`. Consider revoking the API token if its value appeared anywhere you shared.

## Troubleshooting

| Problem | Fix |
| --- | --- |
| `dpkg: command not found` on a Mac | `dpkg` is for Linux. Use the macOS install in Step 1. |
| `SyntaxError` when running `receiver.py` | The text was garbled by pasting. Recreate the file in an editor (Step 2) and run the compile check. |
| `python3: command not found` | Install Python 3 (`xcode-select --install` or `brew install python`). |
| 401 on the task creation call | The token is missing, expired or lacks permission. Check `TFC_TOKEN` and that the user is an organization owner. |
| 422 on the task creation call | The task `name` already exists or is invalid. Use a new unique name. |
| Tunnel URL stopped working | `cloudflared` was restarted and the URL changed. Update `TUNNEL`. |

## Limits of these results

- Only 200, 201 and 202 were tested, and each was tried only a few times.
- The real timeout for a slow response is unknown. A 30 s delay was tolerated.
- Behavior with an HMAC key configured was not tested.
- This is consistent with the public docs asking for `200 OK`, but the docs do not describe the registration handshake itself.
