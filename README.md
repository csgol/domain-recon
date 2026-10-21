```
    ____                        _          ____
   / __ \____  ____ ___  ____ _(_)___     / __ \___  _________  ____
  / / / / __ \/ __ `__ \/ __ `/ / __ \   / /_/ / _ \/ ___/ __ \/ __ \
 / /_/ / /_/ / / / / / / /_/ / / / / /  / _, _/  __/ /__/ /_/ / / / /
/_____/\____/_/ /_/ /_/\__,_/_/_/ /_/  /_/ |_|\___/\___/\____/_/ /_/

```

A small RDAP-based bulk domain availability checker for vetting candidate
domains during **authorized** security engagements — e.g. picking phishing-
simulation / red-team infrastructure domains before you register them.

It queries registry **RDAP** endpoints over HTTPS (no port-43 WHOIS, no
scraping, no rate-limit-hostile hammering). A `404` means the name is
unregistered/available; `200` means it's taken.

---

## Why RDAP instead of WHOIS

- **Structured & predictable** — clean HTTP status codes (`404` = available,
  `200` = registered) instead of parsing free-text WHOIS banners that differ
  per registrar.
- **Works over HTTPS (443)** — no reliance on port 43, which is often blocked
  on locked-down or corporate networks.
- **No guessing the WHOIS server** — `.com`/`.net` go straight to Verisign;
  everything else routes through the `rdap.org` redirector.

---

## Usage

```bash
# make it executable once
chmod +x check-domains.sh

# check the example list in .com and .net
./check-domains.sh names.txt

# check custom TLDs
./check-domains.sh names.txt com net org io

# pipe names in from stdin
printf 'example-one\nexample-two\n' | ./check-domains.sh -
```

Input file = one **base name per line, without the TLD**. Blank lines and
lines starting with `#` are ignored. See [`names.txt`](names.txt) for the
format.

### Reading the output

| Result       | Meaning                                                        |
|--------------|----------------------------------------------------------------|
| `AVAILABLE`  | RDAP returned `404` — unregistered. Reliable signal.           |
| `registered` | RDAP returned `200` — taken.                                   |
| `? 429`      | Rate-limited. Increase `SLEEP` at the top of the script.       |
| `? 000`      | No response (timeout/DNS). Check connectivity.                 |
| `? HTTP nnn` | Something else — inspect manually.                             |

Tunables live at the top of the script: `MAX_TIME` (per-request timeout) and
`SLEEP` (delay between requests).

---

## Availability is only step one

An *available* name can still be a bad pick — it may have been registered and
**burned** before, leaving a spam/malware history that gets it pre-blocked by
secure email gateways and web filters. Before you register a surviving
candidate, also check:

1. **Prior use / reputation** — VirusTotal, urlscan.io, and the Wayback
   Machine. You want *no* history of abuse.
2. **Web-filter categorization** — Palo Alto URL Filtering, Cisco Talos,
   Symantec SiteReview. Aim for a clean/benign category, not "newly seen" or
   "suspicious," which SEGs flag.
3. **Trademark collision** — a quick USPTO TESS search so a theme name doesn't
   accidentally shadow a real mark (an ethics *and* legal problem, even in
   scope).

### Deliverability / infra hygiene (once registered)

- **Age or warm** domains before use; freshly registered names get flagged.
- Keep phishing-sim domains on **separate infrastructure** from C2 and other
  tooling, so a burned domain doesn't roll up the rest of the op.
- Set up SPF/DKIM/DMARC properly so mail actually lands.

A future version of this script can fold the VirusTotal + urlscan checks in
directly (you'd supply your own API keys via a `.env`, which is already
`.gitignore`d).

---

## Requirements

- `bash` and `curl` (both standard on Linux / macOS / WSL).
- Outbound HTTPS to `rdap.verisign.com` and `rdap.org`.

---

## Files

```
rt-domain-recon/
├── check-domains.sh   # the checker
├── names.txt          # example input (replace with your scoped list)
├── .gitignore         # keeps real scope/results/secrets out of git
└── README.md
```

---
