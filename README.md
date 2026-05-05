# Hypersonic — fork with proxy-body fix

This is a fork of [Hypersonic](https://metacpan.org/pod/Hypersonic) (CPAN, by LNATION), a perl HTTP server framework that JIT-generates C code for the request loop. The original git repository linked from `META.json` (`github.com/ThisUsedToBeAnEmail/Hypersonic`) returns 404 at the time of this writing, so this fork was published to share a single fix that's hard to work around without patching the framework.

The base source is **Hypersonic 0.12** as released on CPAN.

## The bug

`Hypersonic` reads each HTTP/1.1 request with a single non-blocking `recv()` into a thread-local buffer:

```c
ssize_t len = recv(fd, recv_buf, RECV_BUF_SIZE - 1, 0);
```

It then locates the request body by searching for `\r\n\r\n` and computes:

```c
body     = body_start + 4;
body_len = len - (body - recv_buf);
```

This treats whatever arrived in the first `recv()` as the entire request. It works when the client sends the headers and body in one TCP segment (e.g. `curl -d` over loopback), because the kernel hands them up together.

It **fails** when an HTTP proxy or CDN sits in front of the listener — Cloudflare Tunnel (`cloudflared`), nginx `proxy_pass`, haproxy, etc. Those proxies typically write the request headers in one TCP segment and the body in a subsequent segment. The first `recv()` returns just the headers; `body_len` becomes 0; routes registered with `parse_json` / `parse_form` see an empty body, and the framework returns:

```json
{"error":"invalid_json","message":"json body required"}
```

Confirmed via `tcpdump -A -i lo "tcp port <port>"`:

```
PSH  POST /endpoint HTTP/1.1\r\n…Content-Length: 78\r\n\r\n
PSH  {"...the body..."}      <-- arrives microseconds later, in its own segment
```

GET requests have no body, so they're unaffected. Only POST/PUT/PATCH/DELETE-with-body fire the bug, which is why this can lurk in a production deploy until someone tries to write data.

## The fix

`gen_body_parser` in `lib/Hypersonic/Protocol/HTTP1.pm` now:

1. Locates the body via `\r\n\r\n` (unchanged).
2. Parses the `Content-Length:` header out of the headers buffer (case-insensitive).
3. If `body_len < content_length`, enters a `poll()` + `recv()` loop that drains the remainder into `recv_buf`, capped at `RECV_BUF_SIZE - 1` and the configured `RECV_TIMEOUT`. The TLS path (`HYPERSONIC_TLS`) is honoured.
4. NUL-terminates `recv_buf` at the new `len`.

`<poll.h>` was added to the common includes in `lib/Hypersonic.pm` so the generated C can call `poll()`.

The change is local to the body-parsing step. No public Perl API changed, no new options, no behavioural change for requests that already arrived intact in one `recv()`.

### Diff scope

- `lib/Hypersonic.pm` — one extra `->line('#include <poll.h>')` in the includes block.
- `lib/Hypersonic/Protocol/HTTP1.pm` — extended `gen_body_parser` (`has_body_access` branch only) with the Content-Length parser and drain loop.

See commit `b6eec62` for the exact change.

## How to use this fork

```bash
# clone and install over your existing Hypersonic
git clone git@github.com:imyaman/Hypersonic.git
cd Hypersonic
perl Makefile.PL
make
sudo make install
```

Or with `cpanm`:

```bash
cpanm git@github.com:imyaman/Hypersonic.git
```

Restart your Hypersonic-based service; the C code is regenerated at startup, so the patched body parser is built in.

## Verifying the fix

Before:

```bash
curl -i -H 'Content-Type: application/json' -d '{"foo":"bar"}' https://your-app/endpoint
# HTTP/2 400
# {"error":"invalid_json","message":"json body required"}
```

After:

```bash
curl -i -H 'Content-Type: application/json' -d '{"foo":"bar"}' https://your-app/endpoint
# HTTP/2 200       (or whatever your handler returns)
```

Direct hits to the listener (no proxy in front) behaved correctly both before and after.

## Upstream

If/when the upstream repo reappears, this commit is suitable to submit as a PR — it's a single, contained, behaviour-preserving fix.
