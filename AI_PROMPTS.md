# Generative AI Disclosure

Using AI tools for this challenge is allowed. Be honest here — an accurate
"I used AI heavily" is worth more to us than an inaccurate "I did not".

**Did you use AI tools?** (yes / no): yes

**Which ones?** (e.g. ChatGPT, Claude, Copilot, Cursor): Claude (Claude Code)

---

## 1. Key prompts

1. Walk me through this repo and what each of the four defects actually is, in plain language, before changing any code — I want to understand the architecture first.
2. Build just the proxy service and show me the full log, not just the last line — I want the real error, not a guess.
3. Explain why the Dockerfile fails to build and what the minimal, correct fix is — I'll make the edit myself.
4. The proxy builds now but requests fail — explain how Compose's service-name DNS works and what's wrong in nginx.conf.
5. Tests still fail after fixing the routing — diagnose why, don't just patch the test's timeout to make it pass.
6. CI fails even though it's green locally — pull the actual Actions log and find the real cause instead of assuming it's the same bug.

For each one it explained the root cause first, I made the actual edit myself, then we re-ran the tests to verify before moving on.

---

## 2. Where the AI got it wrong

Before we'd even run a build, I asked why the proxy would fail and it guessed `apt-get install nginx` was hanging on a `tzdata` timezone prompt — a real and pretty common Ubuntu thing, so it sounded believable. Once we actually ran the build and read the log, that wasn't it at all — the real issue was the Dockerfile copying `nginx.conf` into place before nginx was even installed, so apt found a conflicting config file already there and asked a totally different prompt. Caught it by making it actually run the thing instead of trusting the first guess.

---

## 3. Anything in your submission you could not fully explain?

Nothing — went through each fix with it before touching anything, and checked every single one by rerunning the tests (and the real GitHub Actions run at the end).
