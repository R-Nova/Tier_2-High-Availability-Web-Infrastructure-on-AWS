# Blockers Encountered & Resolutions

Every real obstacle hit during this build, in the order encountered, with root cause and fix — so the same time isn't lost twice on a rebuild.

| # | Blocker / Symptom | Root Cause | Fix Applied |
|---|---|---|---|
| 1 | MobaXterm: "Unable to use key file … (unable to open file)" | The `.pem` private key was stored on a cloud-synced drive letter (`G:\`) — Google Drive/OneDrive wasn't running or signed in, so Windows couldn't read the file. | Moved the key to a local path (`C:\Users\<you>\Desktop\CloudB\...`) that doesn't depend on any sync client being active, then repointed the MobaXterm session and SSH config to the new path. |
| 2 | SSH client (MobaXterm & VS Code) hangs — "connecting and reconnecting" indefinitely | Verbose SSH (`ssh -v`) showed the TCP connection succeeding but hanging right after the client sent its version string — meaning `sshd` on the jump server itself had stalled. Confirmed independently via AWS EC2 Instance Connect, which also failed with a generic connection error, proving the problem was server-side, not local network or client config. | Rebooted the EC2 instance from the console (Instance State → Reboot, not Stop/Start, to preserve the public IP). This restarted `sshd` and every other service; both MobaXterm and VS Code connected normally afterward. |
| 3 | VS Code Remote-SSH: `/var/www/html/` appears empty after connecting | VS Code's file explorer panel simply hadn't refreshed after the folder was opened — the terminal's own `ls -la` on the same path showed the files were actually present. | Manually refreshed the file explorer panel (right-click → Refresh); files appeared immediately. No actual data was missing. |
| 4 | Datadog agent install: `sudo systemctl status datadog-agent` → "could not be found" | Two separate bugs stacked: (1) the install one-liner had `DD_API_KEY` and `DD_SITE` incorrectly merged with an `=` and no space, so `DD_SITE` was never actually set; (2) the install script URL itself (`s3.amazonaws.com/dd-agent-bootstrap/...`) returned an S3 `NoSuchBucket` error — the bucket no longer exists. | Rewrote the command with `DD_API_KEY` and `DD_SITE` as two properly space-separated variables, and switched to the current official endpoint: `install.datadoghq.com/scripts/install_script_agent7.sh`. Verified with `cat` on the downloaded script before trusting it a second time. |
| 5 | SSH hop from jump server to private instance: "Permission denied (publickey)" / "Identity file not accessible" | The private key only ever existed on the local Windows machine, never on the jump server itself — so any `ssh` command run from the jump server had no key to offer. | Copied the `.pem` file onto the jump server via MobaXterm's SFTP panel, then `chmod 400`'d it so SSH would accept its permissions, before hopping into each private IP. |
| 6 | Website unreachable — browser redirected to a GoDaddy parking page | No DNS record yet pointed `cloudbinocular.online` / `www` at the AWS Application Load Balancer; GoDaddy's own default records were still active. | Removed GoDaddy's default parking records; added a CNAME for `www` pointing at the ALB's DNS name, and set root-domain Forwarding to the `www` address. |
| 7 | Custom domain: "This site can't be reached" / connection timed out, even after DNS checker showed the record propagated | The value entered for the CNAME had a placeholder-style / earlier fake ALB name in it rather than the real one copied from the console. | Copied the exact DNS name directly from EC2 → Load Balancers using the console's copy-icon (never hand-typed), and re-verified the ALB responded correctly first via its raw `amazonaws.com` address before touching DNS again. |
| 8 | Browser: "Your connection is not private — certificate is from cloudbinocular.online" when visiting www.cloudbinocular.online | The original ACM certificate attached to the ALB listener only listed `cloudbinocular.online` as a covered name — `www.cloudbinocular.online` was never included as an additional name (SAN). | Requested a new ACM certificate listing both `cloudbinocular.online` and `www.cloudbinocular.online`, validated it via DNS, and attached the new certificate to the ALB's HTTPS:443 listener. |
| 9 | New ACM certificate stuck on "Pending validation" indefinitely | The validation CNAME record for the `www.cloudbinocular.online` name was entered in GoDaddy without its required `.www` subdomain prefix, so the record GoDaddy actually published didn't match what ACM was checking for. | Compared ACM's exact "CNAME name" column value character-by-character against GoDaddy, corrected the Name field to include the missing `.www` segment, and waited a short cycle for validation to complete. |

---

# Lessons Learned for Next Rebuild

- **Attach an Elastic IP to the jump server on day one** — a plain public IP changes on stop/start and silently breaks every saved SSH/RDP session.
- **Never store the working private key on a cloud-synced drive letter** — keep the live copy local and back it up separately.
- **Request the ACM certificate for the root domain plus a wildcard** (`*.cloudbinocular.online`) together in one request — this cuts DNS validation down to effectively one record instead of one per name, halving the chance of a typo like the missing `.www` prefix.
- **Always copy AWS-generated values** (ALB DNS names, ACM CNAME records) directly from the console with the copy icon — never hand-type or reuse example/placeholder text.
- **When an SSH session hangs rather than refuses**, run `ssh -v` immediately — a hang right after the version string exchange points at the server/sshd, not the key or the network.
- **Double-check any install script's source URL is still current** before trusting a one-liner from an older guide — pipe-to-bash commands fail silently if the endpoint has moved.
- **EC2 Instance Connect** (browser-based, bypasses local network entirely) is the fastest way to tell whether an SSH problem is server-side or client/network-side.

---

See [`README.md`](README.md) for the project overview and full command runbook.
