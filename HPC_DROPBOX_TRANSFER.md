# Transferring data between the Duke-NUS HPC and Dropbox

Moving large datasets between HPC storage and Dropbox with `rclone`, without
routing anything through your laptop.

Install once into your HPC home; run from the DTN (preferred) or the login node.
Always verify with `rclone check` before deleting anything from the HPC.

---

## 1. One-time setup

### Install rclone

A single static binary — no admin rights, no modules.

```bash
ssh <you>@<hpc-login-node>
cd ~
wget https://downloads.rclone.org/rclone-current-linux-amd64.zip
unzip -q rclone-current-linux-amd64.zip
mkdir -p ~/bin && cp rclone-*-linux-amd64/rclone ~/bin/
rm -rf rclone-*-linux-amd64*
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
rclone version
```

Install it in your **HPC home**, not the DTN's. The DTN mounts the same storage,
so one install serves both nodes (see §2).

### Connect Dropbox

The HPC has no browser, so authentication is split across two machines. Install
rclone on your laptop too (`brew install rclone` on macOS).

On the HPC:

```bash
rclone config
```

Answer:

| Prompt | Answer |
|---|---|
| `n/s/q` | `n` (new remote) |
| `name` | `dropbox` |
| `Storage` | `dropbox` (or its menu number) |
| `client_id` | *(blank — but see §5 on rate limits)* |
| `client_secret` | *(blank)* |
| `Edit advanced config?` | `n` |
| **`Use web browser to automatically authenticate?`** | **`n`** |

It prints a command to run elsewhere. On your **laptop**:

```bash
rclone authorize "dropbox"
```

Sign in through the browser window that opens, then copy the token it prints and
paste it back at the HPC's `config_token>` prompt. Confirm with `y`, quit with `q`.

Test:

```bash
rclone lsd dropbox:
```

> **Never paste rclone's config output anywhere.** It contains a long-lived
> refresh token granting full access to your Dropbox. If one leaks, revoke it
> immediately: dropbox.com → Settings → Connected apps → disconnect rclone, then
> re-authorise. To show someone that your setup works, use `rclone lsd dropbox:`,
> which exposes nothing.

---

## 2. Which node to run on

**Use the DTN.** It is built for data movement and keeps long transfers off the
shared login node.

```bash
ssh <you>@<dtn-node>
```

### Paths differ between nodes

The DTN mounts the same storage at a different root. The rule is simply
`/data/` → `/mnt/oneFS/`:

| On the login node | On the DTN |
|---|---|
| `/data/<group>/Data/<project>` | `/mnt/oneFS/<group>/Data/<project>` |
| `/data/<group>/home/<user>` | `/mnt/oneFS/<group>/home/<user>` |

### The DTN has its own home directory

`$HOME` on the DTN is **not** your HPC home, so `~/bin/rclone` and
`~/.config/rclone/` are not there. Point at the HPC copies instead:

```bash
HPCHOME=/mnt/oneFS/<group>/home/<you>
export PATH="$HPCHOME/bin:$PATH"
export RCLONE_CONFIG="$HPCHOME/.config/rclone/rclone.conf"
rclone lsd dropbox:          # confirm it works before transferring
```

Put those three lines in the DTN's `~/.bashrc` if you use it often.

### If the DTN gives "Permission denied"

The DTN may authenticate you as a domain account with different group
memberships than the HPC. Check:

```bash
id                           # are the groups you need listed?
ls -l /mnt/oneFS/<group>/Data/
```

If a group you have on the HPC is missing here, the DTN cannot read that share.
Either raise a ticket with HPC support to have the group added, or **run from
the login node instead** — rclone is network-bound rather than CPU-bound, so a
single transfer process there is acceptable.

---

## 3. Uploading to Dropbox

Dropbox paths in rclone are **relative to your Dropbox root**. Drop the
`/Users/<you>/Dropbox/` prefix your laptop shows:

| Laptop path | rclone path |
|---|---|
| `/Users/you/Dropbox/My Lab/Data/` | `dropbox:My Lab/Data/` |

Quote paths containing spaces or parentheses.

Confirm the destination exists first:

```bash
rclone lsd "dropbox:My Lab/Data"
```

Large transfers run for hours, so they must survive logout.

### Preferred: run inside `screen`

```bash
screen -S rclone_myproject

SRC=/mnt/oneFS/<group>/Data/<project>
DST="dropbox:My Lab/Data/<project>"
rclone copy "$SRC" "$DST" --transfers 4 --checkers 8 --progress
```

Detach with **Ctrl-A** then **D**; the transfer keeps running. Reattach any time:

```bash
screen -ls                       # list your sessions
screen -r rclone_myproject       # reattach
```

`--progress` gives a live updating display, which only works in a session you
can reattach to. This is what most people on the cluster already do.

### Alternative: `nohup`

If you prefer not to use `screen`:

```bash
nohup rclone copy "$SRC" "$DST" \
  --transfers 4 --checkers 8 \
  --stats 5m --stats-one-line \
  --log-file ~/rclone_upload.log --log-level INFO \
  > ~/rclone_upload.out 2>&1 &
echo "started, pid $!"
```

**Omit `--progress` under `nohup`** — the two conflict. Use `--stats 5m
--stats-one-line` instead, which writes a periodic summary to the log.

**Use `copy`, never `sync`.** `sync` deletes anything at the destination that is
absent from the source — the opposite of what you want from an archive.

### Monitoring a `nohup` transfer

rclone logs a file only when it **finishes**, so the log goes quiet for long
stretches while a multi-GB file uploads. Silence is not a stall. The stats lines
are the real progress indicator:

```bash
grep -vE "Copied|nothing to transfer" ~/rclone_upload.log | tail -3
```

```
23:20:54 INFO : 1.074 GiB / 116.312 GiB, 1%, 3.915 MiB/s, ETA 8h22m18s (xfr#11/1273)
```

Note `--stats-one-line` produces that single line, **not** a block prefixed
`Transferred:` — grepping for that string finds nothing and looks like failure.

Confirm the process is alive:

```bash
pgrep -af "rclone copy" | grep <project>
```

And check for errors (22 bytes of `nohup: ignoring input` is normal and harmless):

```bash
cat ~/rclone_upload.out
```

### What throughput to expect

With rclone's shared default app key, roughly **3–5 MiB/s** — around 8 hours for
100 GB. Several people often run transfers on the login node at once, so you are
also sharing the uplink.

That is usually fine for an overnight transfer. If you need it faster, use your
own Dropbox app key (§6).

**Resuming:** `rclone copy` skips files already present at the destination, so an
interrupted transfer resumes by re-running the identical command. No special
flags, nothing to clean up.

### Being a good neighbour

On a shared node, cap your bandwidth:

```bash
--bwlimit 50M                    # or 8M during working hours
--bwlimit "08:00,10M 18:00,off"  # throttled by day, unrestricted overnight
```

---

## 4. Verifying — do this before deleting anything

A transfer that appears to finish is not proof the data arrived intact.

```bash
rclone check "$SRC" "$DST" --one-way --log-file ~/rclone_check.log
grep -cE "ERROR|differ|missing" ~/rclone_check.log
```

**That count must be `0`.** `--one-way` confirms everything in the source exists
and matches at the destination, ignoring extra files there.

`rclone check` compares sizes and hashes, not just filenames. Only once it
passes is your data genuinely in two places.

For a size sanity check alongside it:

```bash
rclone size "$SRC"
rclone size "$DST"
```

---

## 5. Downloading from Dropbox back to the HPC

The same command with the arguments reversed:

```bash
screen -S rclone_download

SRC="dropbox:My Lab/Data/<project>"
DST=/mnt/oneFS/<group>/Data/<project>
rclone copy "$SRC" "$DST" --transfers 4 --checkers 8 --progress
```

Then verify in the other direction:

```bash
rclone check "$SRC" "$DST" --one-way --log-file ~/rclone_check_dl.log
grep -cE "ERROR|differ|missing" ~/rclone_check_dl.log
```

Check free space before pulling a large dataset down — HPC group quotas are
easier to exhaust than Dropbox.

Useful for inspecting without downloading:

```bash
rclone ls   "dropbox:My Lab/Data/<project>" | head
rclone size "dropbox:My Lab/Data/<project>"
rclone copy "dropbox:My Lab/Data/<project>/report.pdf" ./     # one file
```

---

## 6. Pitfalls

**Uploading into a shared folder syncs to everyone.** Putting 100 GB into a
shared lab folder starts a 100 GB download on every colleague whose Dropbox
syncs that tree — often overnight, often onto a laptop that cannot hold it. Tell
people first so they can set the folder to online-only, or put bulk raw data in
your personal folder and share only the derived outputs.

**Rate limiting.** rclone's bundled Dropbox app key is shared among all rclone
users worldwide and throttled accordingly — expect 3–5 MiB/s, about 8 hours for
100 GB. That is fine overnight. When you need better, create your own key at
[dropbox.com/developers/apps](https://www.dropbox.com/developers/apps) (scoped
app, `files.content.read` + `files.content.write`) and enter the `client_id` and
`client_secret` during `rclone config`.

**Login-node contention.** Several people often run rclone on the login node
simultaneously (`pgrep -af rclone` to see). You share the uplink with them, so a
slow rate is not always the app key.

**Many small files are slow.** Dropbox handles a few large files far better than
thousands of small ones. If a directory holds many small files, `tar` it first.
Sequencing data (large pod5/BAM files) is already well suited.

**Compute nodes have no internet.** Run rclone on the DTN or login node, never
inside a batch job.

**The DTN is not for storage.** It mounts shared storage; its own local
filesystem is scratch and may be cleared without warning.

**Token handling.** See the warning in §1. Treat `~/.config/rclone/rclone.conf`
as a credential file: `chmod 600` it, and never commit it to a repository.

---

## 7. Quick reference

```bash
# --- setup, once, on the HPC login node ---
cd ~ && wget https://downloads.rclone.org/rclone-current-linux-amd64.zip
unzip -q rclone-current-linux-amd64.zip
mkdir -p ~/bin && cp rclone-*-linux-amd64/rclone ~/bin/ && rm -rf rclone-*-linux-amd64*
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc && source ~/.bashrc
rclone config                    # new remote "dropbox"; answer "n" to the browser prompt
                                 # run `rclone authorize "dropbox"` on your laptop, paste the token

# --- on the DTN: use the HPC install ---
HPCHOME=/mnt/oneFS/<group>/home/<you>
export PATH="$HPCHOME/bin:$PATH"
export RCLONE_CONFIG="$HPCHOME/.config/rclone/rclone.conf"

# --- look around ---
rclone lsd dropbox:                           # top-level folders
rclone lsd "dropbox:My Lab/Data"              # inside one
rclone size "dropbox:My Lab/Data/<project>"   # how big

# --- upload, inside screen (preferred) ---
screen -S rclone_myproject
SRC=/mnt/oneFS/<group>/Data/<project>
DST="dropbox:My Lab/Data/<project>"
rclone copy "$SRC" "$DST" --transfers 4 --checkers 8 --progress
#   Ctrl-A then D to detach, `screen -r rclone_myproject` to come back

# --- upload, via nohup (alternative; no --progress) ---
nohup rclone copy "$SRC" "$DST" --transfers 4 --checkers 8 \
  --stats 5m --stats-one-line --log-file ~/rclone_upload.log --log-level INFO \
  > ~/rclone_upload.out 2>&1 &

grep -vE "Copied|nothing" ~/rclone_upload.log | tail -3   # the stats line
pgrep -af "rclone copy"                                   # still alive?

# --- verify (count MUST be 0) ---
rclone check "$SRC" "$DST" --one-way --log-file ~/rclone_check.log
grep -cE "ERROR|differ|missing" ~/rclone_check.log

# --- download (arguments reversed) ---
screen -S rclone_download
rclone copy "$DST" "$SRC" --transfers 4 --checkers 8 --progress
```

| Flag | Why |
|---|---|
| `copy` | Never `sync` — `sync` deletes at the destination |
| `--transfers 4` | Parallel file transfers; 4 is polite and near-optimal for Dropbox |
| `--checkers 8` | Parallel existence checks |
| `--stats 5m --stats-one-line` | Periodic summary; use instead of `--progress` under `nohup` |
| `--log-file` | A record you can check after logging out |
| `--one-way` (check) | Source must exist at destination; extras there are ignored |
| `--bwlimit 50M` | Cap bandwidth on a shared node |
| `--dry-run` | Preview without transferring |

**Interrupted?** Re-run the identical command — `copy` skips what is already
there.
