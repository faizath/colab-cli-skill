---
name: colab-cli
description: Run GPU/TPU jobs on Google Colab from the terminal with the `colab` CLI. Use for provisioning Colab VMs, long training or batch runs on a remote GPU, syncing files to/from a session, and recovering orphaned sessions. Covers the failure modes that silently kill multi-hour runs.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

# Google Colab CLI

Drive Colab VMs from the terminal: `colab new` / `exec` / `upload` / `download` / `stop`.
Run `colab skill` and `colab readme` first — they print the bundled, version-current docs.
This file covers what those docs do **not** say, learned by losing a run to each item.

## The one thing that kills long runs

**A Colab VM is reclaimed unless something keeps calling the keep-alive RPC.**
`colab new` spawns a daemon that does this. Anything that removes the local session
record kills that daemon, and the VM dies ~40 minutes later — long after the event
that doomed it, which makes the two look unrelated.

The cascade, which is easy to walk into:

1. The runtime proxy token expires (**1 hour TTL**).
2. Your next `colab exec` gets a 401/404.
3. The CLI "helpfully" **auto-prunes the session** and **kills the keep-alive daemon**.
4. `sessions.json` is now `{}`; `colab sessions` shows the VM as an orphan `[?]`.
5. ~40 min later Colab reclaims the VM. Training, checkpoints, uploads: gone.

**Reading a log does not refresh the idle timer. Only the keep-alive RPC does.**
If you restore access after a prune, you have restored *observability*, not *survival*.
Re-establish keep-alive in the same breath or you are still on the clock.

For any run longer than ~45 minutes, drive keep-alive yourself rather than trusting
the daemon (see the poller below). It is a few lines and it is the difference between
a finished run and a lost one.

**Expect the prune; engineer around it, not against it.** On a 2-hour run the CLI will
prune itself at the 1h mark — that is normal, not a bug you can avoid. Observed twice
in one session:

* Without an independent keep-alive: VM reclaimed ~40 min later, run lost at 38%.
* With one: `colab exec` starts failing (`Session ... appears to be lost (404/401).
  Cleaning up.`), `colab upload` returns a misleading
  `File or directory not found: <remote path>` — and the VM, the job, and the
  checkpoints are all completely fine. Those CLI errors are the *symptom* of the prune,
  not evidence about the job.

So: once a job is launched detached, stop routing anything through `colab -s <name>`.
Drive reads, writes, and keep-alive over the REST API with per-call credentials. The
named-session path is for setup only.

## Long runs: detach, then poll

Never hold a multi-hour job open on `colab exec` — the websocket is fragile and
`--timeout` defaults to **30 seconds** (raise it for anything slower).

Launch detached **on the VM**, so the job outlives the kernel, the websocket, the
token, and the session record:

```python
subprocess.Popen(['python', '-u', '/content/train.py', *args],
                 stdout=open('/content/logs/train.log', 'ab'),
                 stderr=subprocess.STDOUT,
                 start_new_session=True, cwd='/content')
```

`start_new_session=True` is what makes it survive. A detached job survives a token
expiry, a pruned session, and a kernel restart. It does **not** survive VM reclaim.

Then poll from a persistent monitor that refreshes keep-alive every pass and re-mints
credentials each time, so it depends on no local state:

```python
from colab_cli.client import Client, Prod
from colab_cli.auth import get_credentials, AuthProvider
creds = get_credentials(os.path.expanduser('~/.colab-cli-oauth-config.json'),
                        provider=AuthProvider.OAUTH2)
cl = Client(Prod(), creds)
a = next(x for x in cl.list_assignments() if MATCH in x.endpoint)
cl.keep_alive_assignment(a.endpoint)          # <- the line that saves the run
rp = a.runtime_proxy_info                      # fresh url + 1h token, per call
requests.get(f'{rp.url}/api/contents/content/logs/train.log',
             params={'authuser': '0', 'colab-runtime-proxy-token': rp.token,
                     'content': '1'}).json()['content']
```

Run that with the CLI tool env's own interpreter
(`~/.local/share/uv/tools/google-colab-cli/bin/python`) — a system `python3` will fail
on `pydantic_core`. Use `AuthProvider.ADC` only if ADC is actually configured; if
`google.auth.default()` raises, fall back to `OAUTH2`.

Make jobs **resumable** (`--resume` from the newest checkpoint) and checkpoint often.
Resumability converts a reclaim from "lost run" into "lost minutes".

## Recovering an orphaned session `[?]`

The VM is still assigned server-side; only the local name→VM mapping is gone.

**Do not run `colab new` to recover it.** Reading `assign()` suggests it returns an
existing assignment — it does not reliably. It provisioned a **second billable A100**
in a different region while the first still held the running job. Verified the hard way.

Instead: `list_assignments()` mints a fresh 1-hour token for any live assignment, which
gives full access to the Jupyter REST API (`/api/contents/...`, `/api/kernels`) without
any session record. Use that to read logs, list checkpoints, and download results —
and call `keep_alive_assignment()` while you do, or you are watching a VM die.

If a peer/agent shares `~/.config/colab-cli/sessions.json`, a prune in one session
breaks the other. Isolate with the global `--config <path>` flag.

## Monitoring, honestly

Silence is not success, and noise is not failure. Both directions cost real time:

- A watcher that greps only for the success marker stays **silent through a crash** —
  a crashloop looks identical to "still running". Match every terminal state.
- A watcher that greps `Traceback` across all output fires on the **CLI's own**
  transient `RuntimeError: Connection was lost.` and reports a healthy run as dead.
  Scope failure patterns to the job's log, require repeats before declaring failure,
  and treat an unusable poll as "retry", not "failed".
- `until <check>; do :; done` with no `sleep`, backgrounded with no timeout, gets
  SIGTERM'd at the harness cap and prints `Terminated`. That is the *poller* dying,
  not the job. Verify the job before reacting.

## `colab exec` takes a file, not a string

There is **no positional argument**. The only way to send code is `-f/--file`:

```bash
colab exec -s NAME -f setup.py --timeout 120      # correct
colab exec -s NAME 'import os; os.makedirs(...)'  # never runs
```

Inline code is rejected with `Got unexpected extra argument(s)` inside a Rich
error panel that **reprints your code back at you**. Skim the tail of that output
and it reads like an echo of a command that ran. It did not run.

That is the whole trap: the failure is silent at the point of use and loud
somewhere unrelated. A `makedirs` that never executed surfaces one step later as
an opaque `HTTP 500` from `colab upload` — which the table below attributes to a
missing parent directory, correctly, while saying nothing about why the
directory is missing.

So: **write the snippet to a local file, and make it print something you assert
on.** Never infer that an exec succeeded from the absence of an obvious error.

```python
# setup.py -- ends with a claim you can check
import os
for d in ("/content/pkg", "/content/logs"):
    os.makedirs(d, exist_ok=True)
print("dirs ready:", [os.path.isdir(d) for d in ("/content/pkg", "/content/logs")])
```

The same applies to uploads: `colab upload` reports success on the *request*, so
confirm size and md5 from the VM side (see "Verify data crossed intact").

## Environment traps

| Symptom | Cause | Fix |
|:--|:--|:--|
| `AttributeError: module 'jupyter_kernel_client' has no attribute 'KernelClient'` | colab-cli ≤0.6.0 pins the dep unpinned; 1.0 renamed the class | `uv tool install --force google-colab-cli --with "jupyter-kernel-client<1.0"` |
| `ImportError: Found an incompatible version of torchao` on `get_peft_model` | `peft` calls `is_torchao_available()`, which **raises** instead of returning False on Colab's build | `pip uninstall -y torchao` (unless you actually use it) |
| `colab upload` → opaque HTTP 500 | parent directory does not exist — commonly because the `colab exec` meant to create it was passed inline and never ran | `os.makedirs` on the VM first, via `colab exec -f`, and print an assertion |
| `Got unexpected extra argument(s)` echoing your own code back | `colab exec` has no positional arg; code must come from `-f/--file` | write it to a file first; see the section above |
| `TimeoutError: Timeout waiting for output` | `colab exec --timeout` defaults to 30s | raise it, or detach |
| `colab exec` / `execute_code` never returns, but the job it launched *did* start | kernel startup blocks after a session prune; the `Popen` fires before the call hangs | fire-and-forget: run it under a short timeout, ignore the hang, then confirm via a result file the code wrote |

## Cost discipline

- **Always `colab stop -s <name>` when done.** Idle VMs bill until stopped; nothing
  reclaims them except a 24h cap.
- `colab run` (without `--keep`) self-cleans even if the script errors — prefer it for
  one-shot jobs.
- Check `colab sessions` before `colab new`. Two A100s cost twice as much as one, and
  the duplicate is easy to create while chasing a broken session.
- Prove the config on a tiny smoke run before spending hours: it catches OOM at batch
  size, missing packages, and API drift for a couple of minutes of GPU.

## Verify data crossed intact

**A large upload can be accepted and then silently dropped.** Not corrupted — *absent*,
with the call returning success. A 170 MB `adapter_model.safetensors` sent as one
base64 contents-API `PUT` (a 227 MB JSON body) never landed, while 20 MB and 37 MB files
over the same code path were fine. Chunk anything above ~24 MB.

What makes this cost a debugging cycle rather than a retry is that **nothing fails where
the file is missing.** The error surfaces later, inside a library, worded as if you had
passed a bad argument:

```
HFValidationError: Repo id must be in the form 'repo_name' or 'namespace/repo_name':
  '/content/fairleap-v1-clm-qwen3.5-4b-adapter'
```

That is `peft.load_peft_weights` finding no local weights and falling back to the Hub. It
reads as a path bug; it means the weights are not there. Same family as the CLI's
`File or directory not found` after a prune: **the message names the wrong problem.** When
a local-path load starts complaining about repo ids, list the remote directory before you
touch the code.

**Reporting the local byte count is not verification.** `len(blob)` after a `PUT` says only
what you tried to send. Read the size back from the server, per chunk:

```python
def remote_size(proxy, path):                     # contents API, content=0
    return contents(proxy, path, params={"content": "0"}).get("size")

for i, part in enumerate(parts):                  # ~24 MB each
    put_blob(proxy, part, f"{remote}.part{i:03d}")
    assert remote_size(proxy, f"{remote}.part{i:03d}") == len(part)
```

Then concatenate on the VM, confirm the joined size, and **md5 both ends** — size agreement
survives a truncation at a chunk boundary, content agreement does not:

```python
hashlib.md5(open(path,'rb').read()).hexdigest()
```

Prefer pulling over pushing wherever you have the choice. Downloads stream through
Jupyter's raw `/files/` endpoint, which is both faster and far more reliable than the
contents API's base64-inside-JSON — a third larger on the wire and fully buffered at both
ends. 170 MB down took 37 s; the same file up needed seven verified chunks and a join.

```python
requests.get(f'{rp.url}/files/{path}', params={'authuser': '0',
             'colab-runtime-proxy-token': rp.token}, stream=True)
```
