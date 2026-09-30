# bonedigger — ujust report tool

Load when working on the client-side diagnostic reporting tool in `projectbluefin/common`: `system_files/bluefin/usr/share/ublue-os/just/60-bonedigger.just` (the recipe) and `system_files/bluefin/usr/libexec/bonedigger-report` (the script the recipe invokes). The `ujust-report-config.yaml` OTel collector config in `system_files/bluefin/usr/share/ublue-os/otel/` is **image content** owned by `projectbluefin/common` — see [Where the code lives](#where-the-code-lives--do-not-get-this-wrong); the script no longer runs an OTel collector.

## Commands

```bash
ujust report         # collect diagnostics, review locally, upload to gist, open issue
```

## What `ujust report` collects

| Field | Source |
|-------|--------|
| Image name, tag, flavor, ref | `/usr/share/ublue-os/image-info.json` |
| Booted image + digest | `bootc status --json` |
| Staged image | `bootc status --json` |
| Kernel version, architecture | `uname -r`, `uname -m` |
| GNOME version | `gnome-shell --version` |
| Active GNOME extensions | `gnome-extensions list --enabled` |
| Installed Flatpaks | `flatpak list --columns=application,version` |
| Load average | `/proc/loadavg` |
| Memory usage | `free -h --si` |
| Failed systemd units | `systemctl list-units --state=failed` |
| Current boot kernel errors | `journalctl -b 0 -k -p err..emerg` (attached as `journal.txt`) |
| Current boot system errors | `journalctl -b 0 -p err..emerg` (attached as `journal.txt`) |
| Key service logs | `journalctl -b 0 -u <svc>` for gnome-shell, gdm, NetworkManager, bluetooth, rpm-ostree, systemd-coredump (attached as `journal.txt`) |
| Groups (membership only) | `groups` (username redacted) |
| GPU info | `nvidia-smi -q` (NVIDIA), DRM sysfs (AMD), `lspci` (all) |
| Crash / panic detection | Previous boot end state, panic keywords, kernel errors, hardware fingerprint, crash artifact status |

## PII scrubbing

All scrubbing happens on-device before any upload. Nothing identifying leaves the machine raw.

### General scrubbing (applied to all collected data)

| Data | Scrubbed to |
|------|-------------|
| `/home/<username>/` paths | `/home/[REDACTED]/` |
| `/var/home/<username>/` paths | `/var/home/[REDACTED]/` |
| `groups` leading username | `[REDACTED] : group1 group2 ...` |
| NVIDIA GPU UUID | `[REDACTED]` |
| NVIDIA Serial Number | `[REDACTED]` |
| NVIDIA PCIe Bus Id | `[REDACTED]` |
| NVIDIA Minor Number | `[REDACTED]` |
| `USER=`, `LOGNAME=` env vars in logs | `[REDACTED]` |
| Email addresses in logs | `[REDACTED-email]` |
| `machine-id` | Hashed to 8-char anonymous `HOST_ID` (SHA256, not reversible) |
| `host.id`, `host.name`, `host.ip`, `host.mac` | Deleted by OTel resource/privacy processor |
| `_MACHINE_ID`, `_BOOT_ID`, `_UID`, `_GID`, `_CMDLINE`, `_EXE`, `_COMM` | Deleted from journald log attributes |
| `process.owner`, `process.command_line`, `process.executable.path` | Deleted by OTel processor |

### Kernel log scrubbing — `scrub_kernel_log()` (applied to all kernel excerpts)

Applied to every `journalctl -b -1 -k` excerpt. **Order matters** — MAC must run before IPv6 to get the right label.

| Data | Pattern | Scrubbed to |
|------|---------|-------------|
| MAC addresses | `([0-9a-fA-F]{2}:){5}[0-9a-fA-F]{2}` | `[MAC-REDACTED]` |
| IPv4 addresses | `\b([0-9]{1,3}\.){3}[0-9]{1,3}\b` | `[IP-REDACTED]` |
| IPv6 full (≥4 groups) | `([0-9a-fA-F]{1,4}:){3,7}[0-9a-fA-F]{0,4}` | `[IP-REDACTED]` |
| IPv6 compressed (`::`) | `[0-9a-fA-F]{0,4}(:[0-9a-fA-F]{0,4})*::[0-9a-fA-F:]+` | `[IP-REDACTED]` |
| UUIDs / GUIDs | `[0-9a-fA-F]{8}-...-[0-9a-fA-F]{12}` | `[UUID-REDACTED]` |
| Disk/NVMe serials | `(eui\|naa\|wwn\.)[0-9a-fA-F]+` | `[SERIAL-REDACTED]` |
| Home paths | same as general scrubbing | `[REDACTED]` |

**IPv6 regex rationale:** The 4-group minimum (`{3,7}`) avoids false-positives on `HH:MM:SS` timestamps (only 3 groups). The `::` pattern is a separate pass to catch loopback (`::1`), link-local (`fe80::1`), etc.

**`Linux version` line is intentionally excluded** from the hardware fingerprint — it can contain build host strings (e.g. `builduser@buildhost`). Extract only `DMI: .* BIOS` lines.

## Crash / Panic Detection section

Implemented in `projectbluefin/common/system_files/bluefin/usr/share/ublue-os/just/60-bonedigger.just`. All data sourced from `journalctl -b -1` (previous boot). All kernel excerpts pass through `scrub_kernel_log()` before landing in `summary.md`.

### Boot end-state classifier (4 buckets — never assume)

| Status | Condition |
|--------|-----------|
| `previous boot journal unavailable` | `journalctl -b -1` returns no output |
| `clean shutdown` | Shutdown markers found in tail-200 of boot -1 full journal |
| `suspend entered — no resume recorded before next boot` | Last `PM: suspend (entry\|exit)` line in boot -1 kernel log is `entry` |
| `abrupt end — resumed from suspend, no clean shutdown recorded` | Last PM line is `exit` (resumed OK, then crashed) |
| `abrupt end — no shutdown or suspend markers found` | Neither shutdown nor any PM markers present |

**Shutdown grep is scoped to `tail -200`** (not the full journal). Rationale: shutdown markers always appear at the end of a clean boot; streaming the full journal on the crash path (when no marker is found) can drain hundreds of thousands of lines with no progress indicator to the user.

**Three PM buckets, not two.** A boot that resumed from suspend and then crashed must not be reported as "no suspend markers found" — it had suspend markers, just no clean shutdown after.

### Data collected (only when boot -1 is available)

| Sub-section | Command | Notes |
|-------------|---------|-------|
| Panic keyword scan | `journalctl -b -1 -k … \| grep -iE 'panic\|oops\|BUG:\|Call Trace\|RIP:\|…' \| tail -20` | `tail` not `head` — crash is at end |
| Last kernel errors | `journalctl -b -1 -k -p err..emerg … \| tail -30` | |
| Context window (last 30 kernel lines) | `journalctl -b -1 -k … \| tail -30` | Suppressed for clean shutdowns with no findings |
| Hardware fingerprint | `grep -E 'DMI: .* BIOS' \| head -1` | DMI model + BIOS version only |

### Crash artifact status (always collected, independent of boot -1)

| Artifact | How detected |
|----------|-------------|
| pstore | `mountpoint -q /sys/fs/pstore` + `find` file count; "empty" ≠ "no crash" — may have been cleared on boot |
| kdump | `systemctl is-enabled/is-active kdump.service` (service status, not `/var/crash` directory) |
| Userspace coredumps | `coredumpctl list --since "7 days ago" \| tail -10` (home paths scrubbed) |

### `set -euo pipefail` safety rules

- Every `journalctl … | grep … | tail` pipeline ends with `|| true` inside `$()` — grep exits 1 on no match
- Shutdown classification uses `if journalctl … | tail -200 | grep -qiE …` — safe in `if` conditions
- `systemctl is-enabled kdump.service &>/dev/null` — safe in `if` condition
- `${PSTORE_COUNT:-0}` — guards against empty find output

## Deep hardware metrics capture (OTel) — retired

The shipped `bonedigger-report` no longer runs an OpenTelemetry collector. The 35-second `otelcol-contrib` capture, the `metrics.otlp.jsonl` / `logs.otlp.jsonl` outputs, the binary-resolution ladder, the `podman run docker.io/otel/opentelemetry-collector-contrib` fallback, and the `python3`-based `/output/` config-path substitution are **not implemented** and are not part of the v1 contract. The `ujust-report-config.yaml` shipped under `system_files/bluefin/usr/share/ublue-os/otel/` in `projectbluefin/common` is image content owned there; this skill no longer claims it.

Replacement diagnostics live in the **smart-log profiles** the user selects during `submit_draft` (`collect_profiles` in `bonedigger-report`): `desktop-graphics.md`, `sleep-crash.md`, `update-boot.md`, `networking.md`, `flatpak-application.md`. Each is a redacted markdown excerpt (passed through `scrub_journal_log` / `scrub_kernel_log`) capped per-file by `limit_file` and posted as a public gist by `publish_smart_logs` when the user opts in.

## Upload flow

The full submit path lives in `submit_draft()` in `projectbluefin/common/system_files/bluefin/usr/libexec/bonedigger-report`. Steps, in order:

1. **Preview** (`preview_draft`). `gum pager` over `$DRAFT_DIR/issue.md` and each selected profile file. On non-TTY stdin/stdout (`bash < issue.md` style), falls back to plain `cat`. No `glow` is invoked — the preview is plain text.
2. **Queue preference** (bug reports only). `choose_queue_preference` offers `"No queue preference" / "Submit to the clanker queue for machine analysis" / "I only want human interaction"`, persists the resulting label (`3-clanker-queue`, `3-human-queue`, or empty) to `$DRAFT_DIR/queue-label.txt`, and writes an HTML marker to `$DRAFT_DIR/issue.md` so downstream intake can read the preference back.
3. **Consent**. `gum confirm` with a message that varies with whether smart logs were selected — `"Create this public GitHub issue?"` when no profiles were chosen, `"Publish the selected smart logs publicly and create this issue?"` when at least one was. Decline keeps the draft for `--resume`.
4. **`ensure_gh_ready`**. Confirms `gh` is on `$PATH` and `gh auth status --active` succeeds; offers to `brew install gh` / `gh auth login --web --skip-ssh-key` if not. Returns non-zero and keeps the draft on failure.
5. **`publish_smart_logs`** (only when `PROFILE_FILES` is non-empty). `gh gist create --public --desc "ujust report smart logs $(date -I)" "${PROFILE_FILES[@]}"`. The returned gist URL is cached in `$DRAFT_DIR/gist-url.txt` and appended to `$DRAFT_DIR/issue.md` as a `### Selected smart logs` block — so a re-run does not double-post.
6. **`create_issue`**. `gh issue create --repo <repo> --title <title> --body-file <issue.md>` plus `--label <queue-label>` when one was chosen. On failure the draft is kept.
7. **`persist_local_copy`**. Copies `issue.md` to `${XDG_STATE_HOME:-$HOME/.local/state}/ujust-report/last/summary.md` and any non-`issue.md` profile files (e.g. `desktop-graphics.md`) alongside. `journal.txt` is only copied if the draft already contains one — `bonedigger-report` does not generate one.
8. **Cleanup and offer**. On full success: `rm -rf "$DRAFT_DIR"`, print `Local copy kept at: …`, then `offer_browser` (`gum confirm` → `xdg-open` of the issue URL). On any earlier failure the draft survives for `ujust report --resume <draft>`.

## Environment variable overrides

| Variable | Default | Purpose |
|----------|---------|---------|
| `IMAGE_INFO_FILE` | `/usr/share/ublue-os/image-info.json` | Image metadata path |
| `BONEDIGGER_BRAND` | `🫐 Bluefin Bug Report` | Brand name shown in gum header |

`bonedigger-report` reads only these two variables (`bonedigger-report:5-6`). `BONEDIGGER_ISSUE_URL` is **spec, not implemented** — it belongs to the unimplemented [Console QR codes](#console-qr-codes-ujust-report--not-implemented) section.

## Console QR codes (`ujust report`) — not implemented

**Nothing in this section ships.** `bonedigger-report` never invokes `qrencode` and never prints a QR code; the text below is a design spec for future work, kept here alongside the PII/clipboard backlog rows.

After the summary renders, print a QR code so a user on their phone can scan it
and open the report to voice-dictate into it. Full spec:
[`bonedigger-qrcode.md`](bonedigger-qrcode.md).

Two QRs, both URL-only (no PII):

1. **Pre-upload** — right after the summary renders, before the upload confirm:
   QR of the canonical issue-report URL (`BONEDIGGER_ISSUE_URL` / `BUG_REPORT_URL`)
   so the form opens on the phone.
2. **Post-upload** — after `gh gist create --public` succeeds: QR of the public
   gist URL so the report opens on any device.

Render with `qrencode -t ANSIUTF8 -s <size> -m 4 "<url>"` (primary), falling back
to a bundled pure-bash generator when `qrencode` is absent. Wrap in a single
`print_qrcode <url>` helper so the renderer is one choke point; the helper must
round-trip (decoded output equals the input URL).

## Dependencies

- `gum` — TUI prompts and styling
- `gh` — GitHub CLI for gist upload, issue creation, and auth check
- `qrencode` — spec-only, for the unimplemented [Console QR codes](#console-qr-codes-ujust-report--not-implemented); the shipped script never calls it
- `bootc` — reads booted image status
- `jq` — parses JSON from bootc and image-info
- `gnome-shell`, `gnome-extensions`, `flatpak` — collects system info
- `wl-copy` / `xclip` (optional) — clipboard fallback when not authenticated

## Report output structure

`bonedigger-report` keeps two on-disk locations; no `trap - EXIT` cleanup. A draft is created only on the bug-report and feature-request paths (`create_draft` is called from `start_bug_report` and `start_feature_request`); `--confirm` and the "Get help" path never create one. Drafts survive submission-cancelled runs on purpose so `--resume` can finish them.

```
${XDG_STATE_HOME:-$HOME/.local/state}/ujust-report/drafts/draft-XXXXXX/   # created by create_draft, called only from start_bug_report / start_feature_request
  issue.md            — body submitted with `gh issue create` (always)
  title.txt           — title submitted with the issue (always)
  repo.txt            — owner/repo the issue is filed against (always)
  queue-label.txt     — `3-clanker-queue` / `3-human-queue` / empty, persisted by choose_queue_preference
  bug-report.txt      — present only on the bug-report path (flag for queue prompt)
  profile-files.txt   — basenames of selected profile files in this draft
  gist-url.txt        — public gist URL once `publish_smart_logs` succeeds
  desktop-graphics.md / sleep-crash.md / update-boot.md / networking.md /
  flatpak-application.md   — selected smart-log profile outputs (only those chosen)
```

The local "last submission" copy is written by `persist_local_copy` only — `keep_draft` just prints the resume hint and never touches `last/`. It is overwritten on each successful submit:

```
${XDG_STATE_HOME:-$HOME/.local/state}/ujust-report/last/
  summary.md          — copy of issue.md (always)
  journal.txt         — copy of the draft's journal.txt if it had one (bonedigger-report
                        does not generate one; copy is a no-op otherwise)
  <profile>.md        — copy of every selected smart-log profile (desktop-graphics.md etc.)
```

Resume an unsubmitted draft with `ujust report --resume <draft-directory>`. Successful submission removes the draft directory but leaves `last/` in place.

## Consumer context (read before proposing design changes)

- **Bluefin, Aurora, and Dakota all use GitHub as their backend.** They upload reports as GitHub Gists and file GitHub Issues. They do NOT use external paste services.
- `BUG_REPORT_URL` in `/etc/os-release` is the canonical source for the distro's issue tracker — no env var needed for this.
- If adding non-GitHub paste support (e.g. for Fedora, Debian, Ubuntu), use a small hardcoded lookup table keyed on the `BUG_REPORT_URL` domain. Do not add custom `os-release` fields or new env vars for this.

## Where the code lives — do not get this wrong

The recipe, the script it invokes, and the OTel collector config that ships under `/usr/share/ublue-os/otel/` are **image content**, not CI tooling. They live in `projectbluefin/common`:

| File | Path in common |
|------|----------------|
| `ujust report` recipe | `system_files/bluefin/usr/share/ublue-os/just/60-bonedigger.just` |
| `ujust report` script (invoked by the recipe) | `system_files/bluefin/usr/libexec/bonedigger-report` |
| OTel collector config | `system_files/bluefin/usr/share/ublue-os/otel/ujust-report-config.yaml` |

`common` ships these files to every image via `common.bst`. Dakota and bluefin inherit them automatically — do **not** add copies to those repos. The OTel config is shipped but the current `bonedigger-report` does not run it; see [Deep hardware metrics capture (OTel) — retired](#deep-hardware-metrics-capture-otel--retired).

**Sync workflows are the wrong answer.** If you find yourself creating a workflow to copy these files from bonedigger to common (or anywhere else), stop: the file is in the wrong repo. Edit it directly in common.
