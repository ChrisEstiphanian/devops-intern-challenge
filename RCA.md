# Root Cause Analysis

Fill this in as you go. Bullet points are fine — be specific, not long.

---

## Defect 1: Proxy image fails to build

### Symptom / error output

`docker compose build` blew up on the `proxy` service. The last line looked like a normal apt error:

```
failed to solve: process "/bin/sh -c apt-get update && apt-get install -y nginx curl htop vim git" did not complete successfully: exit code: 100
```

but a few lines above that is the actual reason:

```
Setting up nginx-common (1.24.0-2ubuntu7.18) ...
Configuration file '/etc/nginx/nginx.conf'
 ==> File on system created by you or by a script.
 ==> File also in package provided by package maintainer.
*** nginx.conf (Y/I/N/O/D/Z) [default=N] ? dpkg: error processing package nginx-common (--configure):
 end of file on stdin at conffile prompt
```

### How I diagnosed it

Ran `docker compose build` by itself first instead of the whole test script, just to narrow down where it was actually dying. My first guess was a missing package or something dumb like a typo, so I just read the last line and assumed apt-get was broken somehow. That was wrong — scrolled up and saw dpkg was literally asking a yes/no question (`Y/I/N/O/D/Z`) and then timing out because nothing was there to answer it.

### Root cause

The Dockerfile copies our `nginx.conf` into `/etc/nginx/nginx.conf` *before* nginx is even installed. So when apt installs `nginx-common` right after, it tries to drop its own default config in that same spot, notices a file is already sitting there that it didn't put there, and stops to ask "keep yours or use mine?" There's no one around to type an answer during a Docker build, so it just fails.

### Fix

Swapped the order in the Dockerfile — install nginx first, copy the config second. That way apt puts its default config down with nothing in the way, and our custom one just overwrites it right after. No prompt, no hang.

---

## Defect 2: Proxy cannot reach the backend

### Symptom / error output

Once the proxy image actually built, hitting it threw `502 Bad Gateway` — nginx couldn't reach the backend at all.

### How I diagnosed it

Went straight to `proxy/nginx.conf` since that's where the proxying config lives, and found this:

```
upstream app_backend {
    server 127.0.0.1:8080;
}
```

### Root cause

Classic mistake — `127.0.0.1` means "myself" inside a container, not "the other container." The proxy and the backend are two separate containers, each with their own loopback. Docker Compose actually makes this easy: every service gets a DNS name matching its name in `docker-compose.yml`, so the backend is reachable at just `backend` from anywhere else on the same compose network.

### Fix

Changed `server 127.0.0.1:8080;` to `server backend:8080;`. Used the service name instead of localhost.

---

## Defect 3: Tests still fail after fixing defect 2

### Symptom / error output

```
proxy-1       | ... connect() failed (111: Connection refused) while connecting to upstream, ... upstream: "http://172.18.0.2:8080/api/info" ...
test-runner-1 | TEST RUNNER CRASHED: HTTP Error 502: Bad Gateway
```

### How I diagnosed it

After the defect 2 fix, the proxy logs showed it was now correctly resolving `backend` to its real container IP — so DNS/routing was fine. But it was getting "connection refused," which means nothing was even listening on that port yet at the moment the request came in.

Went and looked at `backend/app.py` (didn't touch it, just read it, since it's off-limits) and found `load_config()` sleeps a couple seconds per config key before the server even starts — about 8 seconds total before it's ready to accept connections.

Then checked `docker-compose.yml` — `depends_on` there only waits for the backend *container* to start, not for the app inside it to actually be listening. So `proxy` and `test-runner` were starting way before the backend was actually able to answer anything.

### Fix, and why I chose it over the alternatives

Added a `healthcheck` on the `backend` service that pings `/health` (using Python since there's no `curl` in that image), and changed `proxy`'s `depends_on` to wait on `condition: service_healthy` instead of just "started."

I did briefly consider just bumping the test runner's 3-second timeout instead — would've been a one-line change. But that's exactly the kind of "weaken the test to make it pass" move the README says not to do. It also wouldn't really fix anything, just make the race condition harder to lose — a slower machine or a longer config-load time and it breaks again. The healthcheck actually removes the race entirely: nothing even tries to talk to the backend until it's confirmed ready.

---

## Defect 4: CI pipeline fails

### Symptom / error output

Pushed defects 1-3's fixes and CI still went red, with nothing to do with the containers this time:

```
/home/runner/work/_temp/....sh: line 1: ./scripts/run-tests.sh: Permission denied
##[error]Process completed with exit code 126.
```

### How I diagnosed it

Checked `git ls-files -s scripts/run-tests.sh` and saw mode `100644` — not executable. Checked the original template repo the same way and it has the exact same mode on the exact same blob, so this wasn't something I broke by cloning on Windows, it was baked into the repo from the start.

### Root cause

`scripts/run-tests.sh` was never marked executable in git. On Windows, Git Bash doesn't strictly enforce the Unix execute bit, so `./scripts/run-tests.sh` ran fine for me locally without me ever noticing it wasn't flagged as executable. GitHub Actions runs on real Ubuntu, which does enforce it, so `ci.yml`'s `./scripts/run-tests.sh` step got a flat "permission denied."

### Fix

Ran `chmod +x scripts/run-tests.sh`, but `git status` didn't pick up the change — turned out this repo has `core.filemode=false` set, so git ignores permission changes entirely by default. Used `git update-index --chmod=+x scripts/run-tests.sh` instead, which explicitly stages the execute bit regardless of that setting. Committed (mode changed `100644` → `100755`) and pushed — CI went green on the next run.

---

## Bonus questions

1. **Reduce the proxy image size**

   Swapped `proxy/Dockerfile`'s base from `ubuntu:24.04` + `apt-get install nginx curl htop vim git` to the official `nginx:1.27-alpine` image, which already has nginx built in. No `apt-get` at all anymore, and the debugging tools (`curl`, `htop`, `vim`, `git`) are gone too — none of them were ever used at runtime, they were just left over from whoever built the original image.

   - Before: `300MB`
   - After: `48.2MB`
   - (`docker images devops-intern-challenge-proxy`)

2. **Run the proxy as a non-root user**

   - Changed nginx's `listen` directive from `80` to `8080` in `nginx.conf`. Port 80 is a privileged port that normally requires root (or a Linux capability) to bind; moving to an unprivileged port (>1024) means nothing in the container needs root at all.
   - Added `USER nginx` in the Dockerfile — `nginx:alpine` already ships a low-privilege `nginx` user/group (uid/gid 101), so this applies to the whole process, master included, not just worker processes.
   - Had to `mkdir` + `chown` nginx's cache/temp directories (`/var/cache/nginx/*`) to the `nginx` user before switching to it — they're root-owned by default in the base image, and nginx failed to start without this (`mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)`).
   - Also had to move the pid file to `/tmp/nginx.pid` (`pid /tmp/nginx.pid;`) since the default `/var/run/nginx.pid` location isn't writable by a non-root user.
   - Updated `docker-compose.yml`: proxy's container-side port changed from `80` to `8080` (host-published port 8081 unchanged), and `test-runner`'s `PROXY_URL` updated to match.
   - Verified: `docker compose exec proxy whoami` → `nginx`, `id` → `uid=101(nginx) gid=101(nginx)`.

3. **Other security/performance improvements**

   - Dropping `curl`/`htop`/`vim`/`git` from the image isn't just about size — it's also attack surface. None of those tools should exist in a production container; if someone ever got shell access to it, there's a lot less sitting around to abuse (no compiler/editor/VCS, no extra network client).
   - Switching off a general-purpose Ubuntu base to a purpose-built, actively maintained `nginx` image also means fewer unrelated packages to ever need patching for CVEs unrelated to nginx itself.

---

## Anything you would do differently with more time

- Pin the base images by digest (`nginx:1.27-alpine@sha256:...`, same for `python:3.11-alpine`) instead of just a tag, so a build can't silently change out from under CI later.
- Add a healthcheck to `proxy` too, and have `test-runner` depend on `condition: service_healthy` for it as well, instead of just `service_started` — right now it's safe because nginx starts almost instantly, but it's one more thing that could quietly become a race condition later.
- Double-check `core.filemode=false` isn't masking any other permission-sensitive file in the repo the same way it did for `run-tests.sh`.
