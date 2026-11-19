# Scrub check

This guide was written from private build notes. This page lists what was taken out and how that was checked. **Run the check again after any edit, before every push.**

Checked: 2026-11-18, every file in this repo.

## What was removed or replaced

| Private detail | In this guide |
|---|---|
| Real home subnet and every real address | `<placeholders>`, `192.168.1.0/24` examples, `100.x.y.z` for Tailscale |
| MAC addresses | none. The guide says "find your MAC" and shows `aa:bb:cc:dd:ee:ff` only as a format example |
| Drive serials, service tags, exact model numbers of the server's own parts | removed ("a 2012-era quad-core Intel desktop chip") |
| The server's hostname, Tailscale machine names, tailnet name | removed. "The box"; `<container-name>.<your-tailnet>.ts.net` |
| Emails, real names | removed. The only name is the GitHub handle `DooDleman77`. Git commits use GitHub's noreply address |
| Internet provider, family, home and work details | removed ("the provider's app", "the house") |
| Private notes system (roles, file names, task IDs) | removed entirely |
| The AI co-pilot's real name | "Murderbot", the nickname used in the book |
| Passwords, API keys, passphrases, login links | never in the source. Configs use `<placeholders>`; `.gitignore` blocks `.env`, keys, `.restic-pass` and a real ImmichFrame `Settings.yml` |
| Cluster-era material (Part 2) | not in this repo |

**Kept on purpose:** software names and versions, the Zigbee stick's model (a common retail product), the USB vendor:product ID `10c4:ea60` (the same on every stick with that chip), Jellyfin's internal placeholder item ID `00000000-0000-0000-0000-000000000001`, and the example UIDs `100000`/`100107`/`100033`/`100999` (standard unprivileged-container math).

## The check

Two parts:

- **A private check** for the exact real values (the real subnet, tailnet name, hostnames, names, provider). It runs **outside** this repo on purpose: putting real values in a public search command would publish them.
- **The generic searches below,** which anyone can run from this folder:

```bash
grep -rnIE "192\.168\.|\b10\.0\.0\.|100\.[0-9]+\." . --exclude-dir=.git          # addresses
grep -rnIiE "([0-9a-f]{2}[:-]){5}[0-9a-f]{2}" . --exclude-dir=.git              # MAC addresses
grep -rnIiE "[0-9a-f]{32,}" . --exclude-dir=.git                                 # long hex (keys, hashes)
grep -rnIi "ts\.net" . --exclude-dir=.git                                        # Tailscale names
grep -rnIiE "[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}" . --exclude-dir=.git         # emails
grep -rnIiE "tskey|token|password|serial|\btags?\b" . --exclude-dir=.git         # secret-ish words
grep -rnIiE -e 'tskey-[a-z0-9]{6,}' -e 'gh[pousr]_[a-z0-9]{20,}' -e 'sk-[a-z0-9]{32,}' \
  -e 'AKIA[0-9A-Z]{16}' -e '-----BEGIN [A-Z ]*PRIVATE KEY' \
  -e "(api[_ -]?key|token|secret|password|passphrase)[\"'\`]?[[:space:]]*[:=][[:space:]]*[\"'\`]?[a-z0-9+/_=-]{10,}" . --exclude-dir=.git
```

## Results (2026-11-18)

| Search | Hits | Verdict |
|---|---|---|
| Real subnet, tailnet name, hostnames, personal / provider / notes-system words (private check) | **0** | clean |
| `192.168.` | 2 | both the made-up example `192.168.1.0/24` (README, chapter 5) |
| `100.N.` / `10.0.0.` | 0 | clean (Tailscale addresses are written `100.x.y.z`) |
| MAC addresses | 1 | the format example `aa:bb:cc:dd:ee:ff` (chapter 2) |
| Long hex | 0 | clean |
| `ts.net` | 4 | all `<your-tailnet>` placeholders or "the `…ts.net` address" |
| Emails / `@` | 0 emails; 2 `@` | `ssh root@100.x.y.z` (placeholder) and the `@` character in a sentence about passwords |
| `tskey` | 0 | clean |
| `token` | 1 | prose ("gives the app its own token") |
| `password` | prose only | instructions ("set a password", "password manager", `smbpasswd`, `--password-file`); no values |
| `serial` | 2 | "WebSerial" and "serial port" (Zigbee chapter) |
| `tag` | 2 | Prowlarr's "Tags" feature (chapter 6) |
| Secret-looking lines | **0** | clean |
