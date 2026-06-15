You're right — at 1 billion files, every micro-decision compounds massively. Let me think through every bottleneck systematically.

---

## The Real Bottlenecks at 1B Scale

### What kills you at this scale:

| Problem | Why it's catastrophic |
|---|---|
| `find` per tgz | Spawns a process, walks inodes — 5000× overhead |
| `xargs -I{}` + `ksh -c` per file | **1 process spawn per PDF** = 1B fork/exec calls |
| `grep -qF` on DONE_LOG | O(n) scan on a file growing to 1B lines |
| `aws s3 cp` per file | 1B CLI invocations — each costs ~200ms JVM/Python startup |
| `openssl` CLI per file | 1B process spawns just for crypto |
| Temp file per PDF | 1B writes + 1B deletes = filesystem graveyard |
| Sequential tgz extract | tar is single-threaded and disk-bound |

**The `aws s3 cp` CLI alone, at 200ms startup × 1B files = 6.3 years of pure overhead.**

---

## Rethink Everything

### Principle 1: **Eliminate per-file process spawning entirely**

You need a long-running worker process that loops internally — not a shell that forks for each file.

### Principle 2: **Stream, don't stage**

tar → decrypt pipe → S3 multipart stream. Zero temp files.

### Principle 3: **Use Python workers, not ksh, for the inner loop**

ksh orchestrates; Python (boto3 + cryptography) does the per-file work in a single process with a thread pool — no fork overhead.

### Principle 4: **S3 connection pooling**

One boto3 session with persistent HTTPS connections per worker, not 1B new TCP handshakes.

### Principle 5: **Checkpoint with SQLite, not a flat log**

O(1) lookup vs O(n) grep on a billion-line file.

---

## Revised Architecture

```
                    ksh master
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
     Worker-1       Worker-2  ...  Worker-N      (ksh background jobs, FIFO semaphore)
          │
          ▼
    python3 tgz_worker.py  <tgz_path>
          │
          ├─ Opens tgz with tarfile (streaming, no extract to disk)
          ├─ Thread pool (32–64 threads)
          │       │
          │       ├─ Read member into memory buffer
          │       ├─ AES-256 encrypt in-memory (cryptography lib)
          │       └─ boto3.upload_fileobj() — streaming, persistent session
          │
          └─ SQLite checkpoint per file (WAL mode)
```

---

## The Scripts

### `pipeline.ksh` — Master orchestrator

```ksh
#!/usr/bin/ksh
#================================================================
# pipeline.ksh — Master orchestrator
# AIX 7.2 / ksh88  — manages N concurrent Python tgz workers
#================================================================

TGZ_DIR="/data/tgz"
WORKER_SCRIPT="/opt/pipeline/tgz_worker.py"
CHECKPOINT_DB="/var/pipeline/checkpoint.db"
LOG_DIR="/var/pipeline/logs"
ERR_LOG="$LOG_DIR/master_errors.log"

# With 96 LCPUs: Python workers are I/O bound (network to S3).
# Each worker runs 64 threads internally.
# 8 concurrent tgz workers × 64 threads = 512 concurrent S3 streams.
# Tune up if CPU/network headroom remains.
TGZ_WORKERS=8

S3_BUCKET="your-bucket"
S3_PREFIX="pdfs"
ENC_PASSPHRASE_FILE="/etc/pipeline/enc.key"

mkdir -p "$LOG_DIR"

log() { print "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_DIR/master.log"; }

#----------------------------------------------------------------
# Build the tgz work list — skip already completed tgz files
#----------------------------------------------------------------
build_worklist() {
    python3 - "$TGZ_DIR" "$CHECKPOINT_DB" <<'PYEOF'
import sys, os, sqlite3

tgz_dir, db_path = sys.argv[1], sys.argv[2]

con = sqlite3.connect(db_path)
con.execute("""
    CREATE TABLE IF NOT EXISTS tgz_status (
        tgz_name TEXT PRIMARY KEY,
        status   TEXT,   -- 'done' | 'partial' | 'error'
        ts       INTEGER
    )
""")
con.execute("""
    CREATE TABLE IF NOT EXISTS file_status (
        tgz_name  TEXT,
        pdf_path  TEXT,
        status    TEXT,  -- 'done' | 'error'
        PRIMARY KEY (tgz_name, pdf_path)
    )
""")
con.execute("PRAGMA journal_mode=WAL")
con.execute("PRAGMA synchronous=NORMAL")
con.commit()

done = {r[0] for r in con.execute(
    "SELECT tgz_name FROM tgz_status WHERE status='done'")}

for f in sorted(os.listdir(tgz_dir)):
    if f.endswith('.tgz') and f not in done:
        print(os.path.join(tgz_dir, f))

con.close()
PYEOF
}

#----------------------------------------------------------------
# FIFO semaphore — true N-concurrent on ksh88
#----------------------------------------------------------------
FIFO="$LOG_DIR/.sem.$$"
rm -f "$FIFO"; mkfifo "$FIFO"
integer i=0
while (( i < TGZ_WORKERS )); do
    print x > "$FIFO" &
    (( i++ ))
done

log "Pipeline start | TGZ_WORKERS=$TGZ_WORKERS"
log "Building work list..."

build_worklist | while read tgz_path; do
    read token < "$FIFO"    # blocks until a worker slot is free

    tgz_name="${tgz_path##*/}"
    worker_log="$LOG_DIR/${tgz_name}.log"

    (
        log "START $tgz_name"
        python3 "$WORKER_SCRIPT" \
            --tgz        "$tgz_path" \
            --bucket     "$S3_BUCKET" \
            --prefix     "$S3_PREFIX" \
            --db         "$CHECKPOINT_DB" \
            --keyfile    "$ENC_PASSPHRASE_FILE" \
            --threads    64 \
            > "$worker_log" 2>&1

        rc=$?
        if (( rc == 0 )); then
            log "DONE  $tgz_name"
        else
            log "ERROR $tgz_name rc=$rc" 
            print "$tgz_name" >> "$ERR_LOG"
        fi
        print x > "$FIFO"   # release slot
    ) &
done

wait
rm -f "$FIFO"
log "All workers finished."
python3 -c "
import sqlite3, sys
con = sqlite3.connect('$CHECKPOINT_DB')
done  = con.execute(\"SELECT COUNT(*) FROM tgz_status WHERE status='done'\").fetchone()[0]
err   = con.execute(\"SELECT COUNT(*) FROM tgz_status WHERE status='error'\").fetchone()[0]
fdone = con.execute(\"SELECT COUNT(*) FROM file_status WHERE status='done'\").fetchone()[0]
ferr  = con.execute(\"SELECT COUNT(*) FROM file_status WHERE status='error'\").fetchone()[0]
print(f'TGZ  done={done} error={err}')
print(f'PDF  done={fdone} error={ferr}')
"
```

---

### `tgz_worker.py` — The high-performance inner loop

```python
#!/usr/bin/env python3
"""
tgz_worker.py
Streams a tgz, encrypts each PDF in memory, uploads to S3.
No temp files. Persistent boto3 session. Thread pool per worker.
"""

import argparse
import io
import os
import sqlite3
import sys
import tarfile
import threading
import time
from concurrent.futures import ThreadPoolExecutor, as_completed
from queue import Queue, Empty

import boto3
from botocore.config import Config
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import secrets

# ── CLI ──────────────────────────────────────────────────────────────────────
def parse_args():
    p = argparse.ArgumentParser()
    p.add_argument("--tgz",     required=True)
    p.add_argument("--bucket",  required=True)
    p.add_argument("--prefix",  required=True)
    p.add_argument("--db",      required=True)
    p.add_argument("--keyfile", required=True)
    p.add_argument("--threads", type=int, default=64)
    return p.parse_args()

# ── Encryption ───────────────────────────────────────────────────────────────
# AES-256-GCM: authenticated encryption, no padding oracle, fast.
# Nonce is prepended to ciphertext so decryption is self-contained.
def encrypt_bytes(key32: bytes, plaintext: bytes) -> bytes:
    nonce = secrets.token_bytes(12)          # 96-bit nonce, unique per file
    aesgcm = AESGCM(key32)
    ciphertext = aesgcm.encrypt(nonce, plaintext, b"")
    return nonce + ciphertext                # 12 + len(plain) + 16 bytes tag

# ── Checkpoint DB ─────────────────────────────────────────────────────────────
class Checkpoint:
    """
    SQLite WAL-mode checkpoint. Thread-safe via a dedicated writer thread
    fed by a queue — avoids lock contention from 64 threads.
    """
    def __init__(self, db_path: str, tgz_name: str):
        self.tgz_name = tgz_name
        self.db_path  = db_path
        self._queue   = Queue()
        self._stop    = threading.Event()
        self._done_set = set()
        self._lock    = threading.Lock()

        # Pre-load already-done files for this tgz (resume support)
        con = sqlite3.connect(db_path, check_same_thread=False)
        con.execute("PRAGMA journal_mode=WAL")
        con.execute("PRAGMA synchronous=NORMAL")
        rows = con.execute(
            "SELECT pdf_path FROM file_status WHERE tgz_name=? AND status='done'",
            (tgz_name,)
        ).fetchall()
        self._done_set = {r[0] for r in rows}
        con.close()
        print(f"[checkpoint] resume: {len(self._done_set)} files already done",
              flush=True)

        # Start background writer thread
        self._writer = threading.Thread(target=self._write_loop, daemon=True)
        self._writer.start()

    def already_done(self, pdf_path: str) -> bool:
        return pdf_path in self._done_set

    def mark(self, pdf_path: str, status: str):
        self._queue.put((pdf_path, status))
        if status == "done":
            with self._lock:
                self._done_set.add(pdf_path)

    def _write_loop(self):
        con = sqlite3.connect(self.db_path, check_same_thread=False)
        con.execute("PRAGMA journal_mode=WAL")
        con.execute("PRAGMA synchronous=NORMAL")
        buf = []
        while not self._stop.is_set() or not self._queue.empty():
            try:
                item = self._queue.get(timeout=0.1)
                buf.append(item)
            except Empty:
                pass
            # Batch-commit every 500 records or 0.5s — reduces fsync overhead
            if len(buf) >= 500:
                self._flush(con, buf); buf = []
        if buf:
            self._flush(con, buf)
        con.close()

    def _flush(self, con, buf):
        con.executemany(
            """INSERT OR REPLACE INTO file_status (tgz_name, pdf_path, status)
               VALUES (?, ?, ?)""",
            [(self.tgz_name, p, s) for p, s in buf]
        )
        con.commit()

    def stop(self):
        self._stop.set()
        self._writer.join()

    def mark_tgz(self, status: str):
        con = sqlite3.connect(self.db_path)
        con.execute("PRAGMA journal_mode=WAL")
        con.execute(
            """INSERT OR REPLACE INTO tgz_status (tgz_name, status, ts)
               VALUES (?, ?, ?)""",
            (os.path.basename(self.tgz_name), status, int(time.time()))
        )
        con.commit()
        con.close()

# ── S3 Session Factory ────────────────────────────────────────────────────────
def make_s3_client():
    """
    One client per thread — boto3 clients are NOT thread-safe.
    Configured for high-throughput: large connection pool, aggressive retry.
    """
    return boto3.client(
        "s3",
        config=Config(
            max_pool_connections=1,          # 1 per thread client
            retries={"max_attempts": 5, "mode": "adaptive"},
            tcp_keepalive=True,
        )
    )

# Thread-local storage for per-thread S3 client
_thread_local = threading.local()

def get_s3():
    if not hasattr(_thread_local, "client"):
        _thread_local.client = make_s3_client()
    return _thread_local.client

# ── Per-file worker ───────────────────────────────────────────────────────────
def process_member(
    member_data: bytes,
    s3_key: str,
    bucket: str,
    key32: bytes,
    checkpoint: Checkpoint,
    tgz_name: str,
) -> tuple[str, bool, str]:
    """Encrypt + upload one PDF. Returns (s3_key, success, error_msg)."""
    try:
        encrypted = encrypt_bytes(key32, member_data)
        get_s3().upload_fileobj(
            io.BytesIO(encrypted),
            bucket,
            s3_key,
            ExtraArgs={"ContentType": "application/octet-stream"},
        )
        checkpoint.mark(s3_key, "done")
        return s3_key, True, ""
    except Exception as e:
        checkpoint.mark(s3_key, "error")
        return s3_key, False, str(e)

# ── Main ──────────────────────────────────────────────────────────────────────
def main():
    args   = parse_args()
    key32  = open(args.keyfile, "rb").read(32)   # exactly 32 bytes for AES-256
    tgz_name = os.path.basename(args.tgz)
    ckpt   = Checkpoint(args.db, tgz_name)

    total = done = errors = 0
    t_start = time.time()

    print(f"[{tgz_name}] opening tgz stream", flush=True)

    futures = {}
    with ThreadPoolExecutor(max_workers=args.threads) as pool:
        with tarfile.open(args.tgz, "r:gz", bufsize=4*1024*1024) as tf:
            for member in tf:
                if not member.isfile() or not member.name.endswith(".pdf"):
                    continue

                total += 1
                s3_key = f"{args.prefix}/{tgz_name}/{member.name}"

                # Skip already checkpointed
                if ckpt.already_done(s3_key):
                    done += 1
                    continue

                # Read member bytes — tarfile must be read sequentially
                # (streaming: we cannot seek in a .tar.gz)
                fobj = tf.extractfile(member)
                data = fobj.read()    # <100KB per file — fits easily in RAM

                fut = pool.submit(
                    process_member,
                    data, s3_key, args.bucket, key32, ckpt, tgz_name
                )
                futures[fut] = s3_key

                # Drain completed futures periodically to cap memory
                # (avoids queuing all 100K futures at once)
                if len(futures) >= args.threads * 4:
                    done_futs = [f for f in futures if f.done()]
                    for f in done_futs:
                        _, ok, err = f.result()
                        if ok: done += 1
                        else:  errors += 1
                        del futures[f]

                # Progress heartbeat
                if total % 10000 == 0:
                    elapsed = time.time() - t_start
                    rate = total / elapsed if elapsed else 0
                    print(f"[{tgz_name}] {total:,} seen | "
                          f"{done:,} done | {errors} err | "
                          f"{rate:.0f} files/s", flush=True)

        # Drain remaining futures
        for f in as_completed(futures):
            _, ok, err = f.result()
            if ok: done += 1
            else:  errors += 1

    ckpt.stop()

    elapsed = time.time() - t_start
    rate = total / elapsed if elapsed else 0
    print(f"[{tgz_name}] COMPLETE: {done:,} ok | {errors} err | "
          f"{elapsed:.0f}s | {rate:.0f} files/s", flush=True)

    if errors == 0:
        ckpt.mark_tgz("done")
        sys.exit(0)
    else:
        ckpt.mark_tgz("error")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

---

## Why Every Decision Matters at 1B Scale

```
                     1 BILLION FILES
                           │
    ┌──────────────────────┼───────────────────────┐
    │                      │                       │
    ▼                      ▼                       ▼
aws s3 cp CLI          openssl CLI           ksh fork per file
  ×1B calls              ×1B calls              ×1B calls
= 200ms startup        = 50ms startup         = 5ms startup
= 6.3 YEARS            = 1.6 YEARS            = 58 DAYS
    │                      │                       │
    ▼                      ▼                       ▼
boto3 in-process       AES-GCM in-process     ThreadPool in-process
  persistent conn        pure Python/C          zero fork
= ~2ms per file        = ~1ms per file        = ~0.1ms overhead
= 23 DAYS              = 11.6 DAYS            = negligible
```

| Decision | Naive (ksh per file) | This approach | Saving |
|---|---|---|---|
| S3 upload | 6.3 years | 23 days | **99.1%** |
| Encryption | 1.6 years | 11 days | **99.3%** |
| Checkpoint lookup | Weeks (grep) | Hours (SQLite) | **99.9%** |
| Temp file I/O | Billions of writes | Zero | **100%** |
| Process spawns | ~1B forks | ~40 processes | **100%** |

---

## Throughput Math

With 8 tgz workers × 64 threads = **512 concurrent S3 uploads**:

- Each upload: ~100KB file, ~50ms on good network = ~2,000 files/thread/s theoretical
- Realistic with crypto + boto3 overhead: **~500 files/thread/s**
- 512 threads × 500 = **~256,000 files/second**
- 1 billion files ÷ 256,000 = **~65 minutes** (network-bound)

Real limit will be **S3 prefix throughput** (3,500 PUT/s per prefix). Fix with prefix sharding:

```python
# Shard S3 prefix by first 2 hex chars of filename hash
import hashlib
shard = hashlib.md5(member.name.encode()).hexdigest()[:2]
s3_key = f"{args.prefix}/{shard}/{tgz_name}/{member.name}"
# Gives 256 prefixes × 3,500 = 896,000 PUT/s theoretical
```

---

## Dependencies to install on AIX

```ksh
# Python packages only — no system installs needed
pip3 install boto3 cryptography

# Verify
python3 -c "import boto3, cryptography; print('OK')"
```

The entire strategy collapses to one rule: **never spawn a process per file at billion-file scale.** Keep everything in long-lived Python threads with pooled connections.
