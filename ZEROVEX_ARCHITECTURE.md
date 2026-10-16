# Zerovex Cloud: Architecture Specification v0.1 (DRAFT, awaiting approval)

Status: **Nothing in this document is implemented.** No code has been written or tested. Every "will" below is a plan, not a result.

---

## 0. How to read this

Each section has plain-language "Beginner note" boxes. The goal is that you can learn why each decision is made, not just what it is.

## 1. Goals and non-goals

**Goals**
- Build, from scratch, locally on Windows: a storage engine (ZVFS), a database (ZVDB), a network protocol (ZVNET), a security layer (ZVSEC), a service manager (ZVCORE), and an admin dashboard (ZVCONSOLE).
- Learn by building: correctness and clarity over speed.
- Every claim of "works" is backed by a test run whose output is reported.

**Non-goals (for now)**
- Public internet exposure. Everything binds to `127.0.0.1` only.
- Competing with production databases on performance.
- Multi-machine clustering or replication (possible far-future milestone).
- Inventing cryptography. We only *use* vetted primitives.

**Honest expectation:** a from-scratch database will have bugs that mature engines fixed long ago. This is a learning and engineering project. It should not hold data you cannot afford to lose until it has survived extensive crash and fuzz testing, and even then, keep backups.

## 2. Rules and what they mean in practice

| Rule | Concrete meaning |
|---|---|
| No silent substitution | ZVDB must not wrap SQLite, LevelDB, RocksDB, LMDB, etc. Python's `sqlite3` module is banned inside ZVDB/ZVFS. Using it in a *test* as an independent comparison oracle is allowed only if we say so explicitly. |
| Established crypto only | TLS 1.3 (Python `ssl`), `hashlib.scrypt` or Argon2id, HMAC-SHA256, AES-256-GCM / ChaCha20-Poly1305 via the `cryptography` package. Using a crypto *library* is not substituting a storage engine. |
| Spec before code | Each module gets an interface file and acceptance tests before its implementation. |
| Tests alongside code | A milestone is not "done" until its tests exist and pass. |
| Local only | Servers bind `127.0.0.1`. Binding anything else requires a documented, approved decision. |
| Approval for destructive ops | Deleting data directories, force-pushing, rewriting history, or wiping test volumes needs your explicit OK. |

## 3. Environment inspection (step 1 of the workflow)

I run in a Linux sandbox and **cannot see your Windows PC**, so I have not inspected it. Please run this in PowerShell and paste the output. It changes nothing.

```powershell
"--- OS ---";            (Get-CimInstance Win32_OperatingSystem | Select Caption,Version,OSArchitecture | Format-List | Out-String)
"--- CPU/RAM ---";       (Get-CimInstance Win32_Processor | Select Name,NumberOfCores | Format-List | Out-String)
                         "RAM GB: " + [math]::Round((Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory/1GB,1)
"--- Disks ---";         Get-PSDrive -PSProvider FileSystem | Select Name,@{n='FreeGB';e={[math]::Round($_.Free/1GB,1)}}
"--- Filesystem type ---"; Get-Volume | Select DriveLetter,FileSystem,DriveType | Format-Table | Out-String
"--- Tools ---"
foreach ($t in 'python','py','git','gcc','cl','rustc','cargo','node','code') {
  $c = Get-Command $t -ErrorAction SilentlyContinue
  if ($c) { "$t : " + $c.Source } else { "$t : NOT FOUND" }
}
python --version; git --version
```

Why each item matters: filesystem type (NTFS vs. exFAT changes durability guarantees), free disk (crash tests write a lot), Python version (we target 3.11+), and compiler presence (decides whether a Rust/C milestone is realistic).

## 4. System overview

```
                       ZVCONSOLE (admin dashboard, local web UI)
                               |  (ZVNET admin API only)
                               v
   clients ---ZVNET (TCP + TLS 1.3, framed binary protocol)---> ZVCORE (supervisor)
                               |                                     |
                         ZVSEC (authn, authz, audit)      config, health, resource monitor,
                               |                           restart policy, crash recovery
                               v
                     ZVDB (tables/keys, transactions, WAL)
                               |
                               v
                     ZVFS (segments, pages, blobs, virtual namespace)
                               |
                               v
                     Windows file APIs (via Python os / msvcrt)
```

**Dependency order (build bottom-up, but ZVDB first against a simple file layer):**
`util (crc, errors, logging)` -> `ZVDB v0.1 (self-contained)` -> `ZVFS` -> `ZVNET` -> `ZVSEC` -> `ZVCORE` -> `ZVCONSOLE`.

Beginner note: ZVDB v0.1 will talk to the OS files directly through a thin `FileBackend` interface. Later, ZVFS implements that same interface, so ZVDB gets swapped onto our own storage engine without rewriting the database. This is how we avoid being blocked by the lower layer.

### 4.1 Layer responsibilities

- **ZVFS**: stores bytes durably. Segmented append-only files, fixed-size pages with checksums, a blob store for large values, and a virtual namespace (`/vol/bucket/object`). Knows nothing about keys or users.
- **ZVDB**: stores structured data. Starts as a key-value engine with a write-ahead log (WAL), then adds transactions, secondary indexes, and a small query API. Knows nothing about networks or users.
- **ZVNET**: moves requests and responses. Defines frames, versioning, request IDs, errors. Knows nothing about what a request means.
- **ZVSEC**: decides who may do what. Credentials, sessions/tokens, roles, permission checks, tamper-evident audit log.
- **ZVCORE**: runs everything. Loads config, starts and stops services in order, health checks, restarts, resource limits, orchestrates recovery on boot.
- **ZVCONSOLE**: displays and controls. Only a client of the admin API; never touches files directly.

## 5. Repository structure

```
zerovex/
├── README.md
├── pyproject.toml              # tooling config (pytest, ruff, mypy)
├── .gitignore                  # includes data/, *.zvlog test artifacts
├── docs/
│   ├── ARCHITECTURE.md         # this file, kept current
│   ├── THREAT_MODEL.md
│   ├── ROADMAP.md
│   ├── LIMITATIONS.md          # honest list of known gaps (updated every milestone)
│   ├── adr/                    # Architecture Decision Records, one file per decision
│   │   └── 0001-record-format.md
│   └── specs/
│       ├── zvdb-v0.1.md        # on-disk format + semantics
│       ├── zvnet-v0.1.md       # wire protocol
│       └── zvsec-v0.1.md
├── src/zerovex/
│   ├── common/                 # errors, crc, logging, clocks (injectable for tests)
│   ├── zvfs/
│   ├── zvdb/
│   ├── zvnet/
│   ├── zvsec/
│   ├── zvcore/
│   └── zvconsole/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── crash/                  # kill-process and fault-injection tests
│   ├── fuzz/                   # corrupted-input and property tests
│   └── conftest.py
├── tools/                      # zvdb-inspect, zvdb-verify, zvdb-dump (debug CLIs)
└── data/                       # runtime data, git-ignored, never committed
```

**Version control policy:** Git. `main` is always green. Work happens on `feature/<milestone>` branches; you review a diff and test report before merge. Commit messages state what was tested. No force-pushes without your approval.

**Tooling (all free, local):** Python 3.11+, `pytest`, `hypothesis` (property-based tests), `ruff` (lint), `mypy` (types), `cryptography` (later, for ZVSEC/ZVNET).

## 6. Key technical decisions (ADR summaries)

**ADR-1: Python first, native later, only with evidence.**
Python is fast to iterate in, which suits learning. We will *measure* before rewriting anything. Candidates for a Rust rewrite later: the ZVDB storage core and the ZVFS page layer (memory safety and speed). Rust is preferred over C/C++ because it prevents whole classes of memory bugs, which matter in a storage engine that parses untrusted bytes. That decision is deferred until Milestone 8 and needs profiling data plus a working Rust toolchain.

**ADR-2: ZVDB v0.1 is a log-structured key-value store (Bitcask-style).**
All writes append to a log. An in-memory hash index maps each key to its latest position in the log. Reads are one hash lookup plus one disk read. Beginner note: this is the simplest design that is still a real, crash-safe database. Its tradeoff is that all keys must fit in RAM. Later versions can move to an LSM-tree or B+tree if we want range scans and larger-than-RAM keysets. We choose the simple one now so we can learn durability and recovery correctly before adding complexity.

**ADR-3: Durability is explicit and tested.**
A write is "durable" only after `fsync` (on Windows, Python's `os.fsync` calls `FlushFileBuffers`). We offer sync modes: `always` (fsync every commit), `batch` (group commit), `os` (no fsync, fast but can lose recent writes). Default is `always`. Known Windows caveat: you cannot fsync a directory the way POSIX allows, so file creation/rename durability needs care. We document this and test it as far as is possible.

**ADR-4: Every record carries a checksum.**
CRC-32 (stdlib `zlib.crc32`) detects accidental corruption and torn writes. It does **not** defend against a malicious attacker; integrity against tampering is a ZVSEC concern (HMAC/AEAD).

**ADR-5: Binary protocol with length-prefixed frames, TLS 1.3 from day one of ZVNET.**
Length-prefix framing avoids ambiguity about where a message ends. Strict maximum frame size prevents memory-exhaustion attacks.

**ADR-6: Passwords hashed with a memory-hard KDF.**
`hashlib.scrypt` (stdlib) with per-user random salt, or Argon2id if we accept the `argon2-cffi` dependency. Never store passwords; store only the hash and parameters. Compare in constant time.

**ADR-7: Clock and randomness are injectable.**
Code takes a `Clock` and RNG parameter so tests are deterministic. Security-relevant randomness always uses `secrets`/`os.urandom`.

## 7. Threat model (v0.1)

### 7.1 Assets
Stored user data; credentials and tokens; audit log integrity; encryption keys; service availability; config files.

### 7.2 Actors
| Actor | Capability |
|---|---|
| Other local user / process on your PC | Can try connecting to local ports, read files if permissions allow |
| Malicious or buggy client | Sends malformed, oversized, or abusive requests |
| Network attacker (only if we ever expose beyond localhost) | Sniff, modify, replay traffic |
| Accidental failure | Power loss, crash, full disk, bit rot, killed process |
| Insider/admin mistake | Wrong command, deleted data |
| Us (developer) | Bugs, including security bugs |

### 7.3 Trust boundaries
1. Network socket -> ZVNET parser (**untrusted bytes enter here**)
2. ZVNET -> ZVSEC (identity claims must be verified, not trusted)
3. ZVDB/ZVFS -> disk (disk contents are untrusted after a crash)
4. ZVCONSOLE -> admin API (browser is untrusted input source)

### 7.4 Threats and mitigations (STRIDE)

| # | Threat | Category | Mitigation | Verified by |
|---|---|---|---|---|
| T1 | Oversized/malformed frames exhaust memory or crash parser | DoS | Max frame size, bounded reads, strict parser, timeouts | Fuzz tests |
| T2 | Torn write after power loss leaves half a record | Tampering/Integrity | CRC per record, truncate invalid tail on recovery | Crash tests |
| T3 | Bit rot in sealed data file | Integrity | CRC on read, `verify` tool, report not hide corruption | Corruption tests |
| T4 | Credential guessing | Spoofing | scrypt/Argon2id, rate limiting, lockout backoff | Auth tests |
| T5 | Password database stolen | Info disclosure | Only salted memory-hard hashes stored | Code review, tests |
| T6 | Eavesdropping/replay on the wire | Info disclosure/Tampering | TLS 1.3; per-request IDs; token expiry | Integration tests |
| T7 | Privilege escalation via bad permission check | Elevation | Deny-by-default, central `authorize()` check, no bypass paths | Permission matrix tests |
| T8 | Path traversal in ZVFS names (`../../`) | Tampering | Normalise and validate virtual paths; never join user input to host paths | Fuzz tests |
| T9 | Audit log edited to hide actions | Repudiation | Append-only, hash-chained entries (each includes hash of previous) | Tamper test |
| T10 | Secrets leaked in logs | Info disclosure | Redaction in logger, never log credentials/tokens | Log-scan test |
| T11 | Two writers corrupt the same data dir | Integrity | OS file lock on data dir, single-writer rule | Concurrency tests |
| T12 | Disk full mid-write | Availability/Integrity | Detect `ENOSPC`, fail the write cleanly, stay consistent | Fault injection |
| T13 | Dashboard XSS/CSRF | Tampering | Escape output, CSRF tokens, localhost bind, auth on every call | Console tests |
| T14 | Dependency supply-chain | Tampering | Few dependencies, pinned versions, hash-pinned lockfile | Review |
| T15 | Another local user connects to localhost port | Spoofing | Authentication required even on localhost; no anonymous access | Auth tests |

### 7.5 Explicitly out of scope for now
Defending against an attacker with admin rights on your PC; side-channel attacks; hardware attacks; DDoS from the internet; formal verification. Each will be listed in `LIMITATIONS.md`.

## 8. Module interfaces (contracts)

These are Python typing sketches. They are proposals; changes need your approval, and each must be finalised in `docs/specs/` before dependent code is written.

### 8.1 Common errors
```python
class ZvError(Exception): ...
class CorruptionError(ZvError): ...      # data failed a checksum / structural check
class NotFoundError(ZvError): ...
class ConflictError(ZvError): ...        # e.g. transaction conflict, CAS mismatch
class LockedError(ZvError): ...          # data dir in use by another process
class PermissionDenied(ZvError): ...
class ProtocolError(ZvError): ...
class ResourceExhausted(ZvError): ...
```

### 8.2 FileBackend (the seam between ZVDB and ZVFS)
```python
class FileBackend(Protocol):
    def open_append(self, name: str) -> "AppendFile": ...
    def open_read(self, name: str) -> "ReadFile": ...
    def list(self) -> list[str]: ...
    def rename_atomic(self, src: str, dst: str) -> None: ...
    def delete(self, name: str) -> None: ...
    def lock_dir(self) -> "DirLock": ...           # raises LockedError

class AppendFile(Protocol):
    def append(self, data: bytes) -> int: ...      # returns offset written at
    def sync(self) -> None: ...                    # durable after return
    def size(self) -> int: ...
    def truncate(self, size: int) -> None: ...     # used by recovery
    def close(self) -> None: ...

class ReadFile(Protocol):
    def read_at(self, offset: int, length: int) -> bytes: ...
    def size(self) -> int: ...
    def close(self) -> None: ...
```

### 8.3 ZVDB
```python
class SyncMode(Enum): ALWAYS = 1; BATCH = 2; OS = 3

class ZvDb(Protocol):
    @classmethod
    def open(cls, path: str, *, sync: SyncMode = SyncMode.ALWAYS,
             backend: FileBackend | None = None) -> "ZvDb": ...
    def put(self, key: bytes, value: bytes) -> None: ...
    def get(self, key: bytes) -> bytes | None: ...
    def delete(self, key: bytes) -> bool: ...
    def keys(self) -> Iterator[bytes]: ...
    def batch(self) -> "WriteBatch": ...           # atomic multi-op (v0.2)
    def compact(self) -> "CompactionStats": ...
    def verify(self) -> "VerifyReport": ...        # scans everything, reports corruption
    def stats(self) -> dict: ...
    def close(self) -> None: ...                   # idempotent
```
Limits (enforced, tested): key <= 1 KiB, value <= 16 MiB (v0.1), keys non-empty.

### 8.4 ZVFS
```python
class ZvFs(Protocol):
    def create_volume(self, name: str, quota_bytes: int | None) -> None: ...
    def write(self, path: str, data: bytes | BinaryIO, *, overwrite: bool = False) -> "ObjectInfo": ...
    def read(self, path: str, offset: int = 0, length: int | None = None) -> bytes: ...
    def stat(self, path: str) -> "ObjectInfo": ...
    def list(self, prefix: str) -> Iterator["ObjectInfo"]: ...
    def delete(self, path: str) -> None: ...
    def verify(self) -> "VerifyReport": ...
```
`ObjectInfo` includes size, content hash (SHA-256), created/modified time, version.

### 8.5 ZVNET frame (v0.1 proposal)
```
Header (16 bytes, big-endian):
  magic      2B  0x5A56 ("ZV")
  version    1B
  type       1B   (REQUEST, RESPONSE, ERROR, PING, PONG, AUTH, CLOSE)
  flags      2B
  request_id 4B
  length     4B   (payload bytes, <= MAX_FRAME, default 1 MiB)
  reserved   2B   (must be 0)
Payload: length bytes (encoding: msgpack-like TLV, defined in zvnet spec)
```
Rules: reject unknown magic/version, reject length > MAX_FRAME before allocating, per-connection read timeout, idle timeout, max in-flight requests.

### 8.6 ZVSEC
```python
class ZvSec(Protocol):
    def create_user(self, name: str, password: str, roles: list[str], *, actor: "Principal") -> None: ...
    def authenticate(self, name: str, password: str, *, remote: str) -> "Session | None": ...
    def verify_token(self, token: str) -> "Principal | None": ...
    def authorize(self, who: "Principal", action: str, resource: str) -> None: ...  # raises PermissionDenied
    def audit(self, event: "AuditEvent") -> None: ...
    def verify_audit_chain(self) -> "VerifyReport": ...
```
Actions look like `db:read`, `db:write`, `fs:write`, `admin:user.create`. Deny by default.

### 8.7 ZVCORE
```python
class Service(Protocol):
    name: str
    def start(self) -> None: ...
    def stop(self, timeout: float) -> None: ...
    def health(self) -> "Health": ...              # OK / DEGRADED / FAILED + detail

class ZvCore(Protocol):
    def load_config(self, path: str) -> "Config": ...   # validated, typed, secrets not echoed
    def register(self, svc: Service, depends_on: list[str]) -> None: ...
    def start_all(self) -> None: ...                    # dependency order
    def stop_all(self) -> None: ...                     # reverse order
    def metrics(self) -> dict: ...                      # CPU, RAM, disk, open connections
```

### 8.8 ZVCONSOLE
A local-only web UI served by a small server we write, using only the admin API. Pages: health, services, DB stats, audit viewer, user management. Designed last.

## 9. Roadmap

| M | Milestone | Done when |
|---|---|---|
| M0 | Repo, tooling, CI-by-script (`make test` equivalent), this spec committed | `pytest` runs, lint passes |
| **M1** | **ZVDB v0.1: persistent KV with WAL/log, recovery** (detailed in section 10) | Acceptance tests in 10.7 pass |
| M2 | ZVDB v0.2: atomic batches, compaction, snapshot reads, stronger concurrency | Crash tests mid-compaction pass |
| M3 | ZVFS v0.1 implementing `FileBackend`, then objects and volumes | ZVDB runs on ZVFS; same tests pass |
| M4 | ZVNET v0.1 plaintext on localhost, framing, fuzzed parser | Fuzz suite and slow-client tests pass |
| M5 | ZVSEC v0.1: users, scrypt, tokens, permissions, audit chain; TLS 1.3 added to ZVNET | Permission matrix, tamper tests pass |
| M6 | ZVCORE v0.1: config, lifecycle, health, restart, resource metrics | Kill-and-recover tests pass |
| M7 | ZVCONSOLE v0.1 | Manual and automated UI checks |
| M8 | Profile; evaluate Rust rewrite of hot paths | Benchmarks justify (or reject) it |
| M9 | Hardening: long soak tests, broader fuzzing, security review | Documented results |

Each milestone ends with: test report, `LIMITATIONS.md` update, and a review request from you before merge.

## 10. Detailed plan: ZVDB v0.1 (first persistent database prototype)

### 10.1 Scope
**In:** `open/put/get/delete/keys/close/verify/stats`, durable append-only log, in-memory index, crash recovery, directory locking, multi-threaded access within one process, inspection tool.
**Out (deferred):** transactions across keys, compaction (basic version M2), range scans, networking, auth, encryption at rest.

### 10.2 On-disk layout
```
<dbdir>/
  LOCK                      # held exclusively while open
  MANIFEST                  # format version, created time, segment list (rewritten atomically)
  seg-000001.zvlog          # append-only segments
  seg-000002.zvlog          # new segment when active one exceeds 64 MiB
```
Beginner note: splitting the log into segments means old segments become read-only. Read-only files are far easier to reason about and later to compact.

### 10.3 Record format (little-endian)
```
Segment header (16B):  magic "ZVLG" | format_version u16 | reserved u16 | segment_id u64
Record:
  crc32      u32   covers everything after this field, to end of record
  seq        u64   monotonically increasing global sequence number
  type       u8    1 = PUT, 2 = DELETE (tombstone)
  key_len    u32
  val_len    u32   (0 for DELETE)
  key        bytes
  value      bytes
```
Why a sequence number: it lets recovery and (later) replication know order unambiguously.
Why tombstones: deleting appends a "this key is gone" record, because the log is never edited in place.

### 10.4 Core algorithms

**Write (`put`)**: validate sizes -> take write lock -> serialize record with CRC -> append to active segment -> if sync mode is `ALWAYS`, `fsync` -> update in-memory index `key -> (segment, offset, length, seq)` **only after** the append (and sync) succeeds -> release lock. If the append fails midway, truncate back to the last known-good size and raise; the index is unchanged.

**Read (`get`)**: look up index -> read record at offset -> **recheck CRC** -> return value. CRC failure raises `CorruptionError`; we never return bad data.

**Recovery (on `open`)**: acquire `LOCK` -> read MANIFEST -> scan each segment from the start, validating each record's CRC and structure, rebuilding the index (higher `seq` wins; tombstone removes key).
- Bad or incomplete record at the **tail of the last segment** = expected torn write after a crash: truncate there, log how many bytes were dropped.
- Bad record **in the middle** of a file or in a **sealed** segment = real corruption: stop and raise `CorruptionError` with segment and offset, unless opened in explicit `repair` mode (which needs your approval and keeps a backup copy first).

**Concurrency**: single writer at a time (mutex); many readers concurrently (readers read immutable positions with `pread`-style reads, never touching the writer's file offset). Python's GIL does not make this correct automatically; we test it.

**Locking on Windows**: `msvcrt.locking` on the `LOCK` file for exclusive access. A second process opening the same dir must get `LockedError`. We test this with a real second process.

### 10.5 Failure handling matrix (each row gets a test)
| Failure | Expected behaviour |
|---|---|
| Process killed mid-write | Reopen succeeds; all acknowledged writes present; partial record dropped |
| Truncated last record (every possible cut point) | Recovery truncates; earlier data intact |
| Flipped bit in a middle record | `CorruptionError` with location; no silent bad read |
| Garbage/zero-filled tail | Treated as torn tail if it fails CRC at end |
| Oversize key/value, empty key | Rejected with clear error, DB state unchanged |
| Disk full during append | Write fails cleanly; DB remains consistent and usable |
| Two processes open same dir | Second gets `LockedError` |
| Use after `close()` | Clear error; `close()` is idempotent |
| Invalid MANIFEST / wrong magic / future version | Refuse to open with clear message |
| Huge `key_len`/`val_len` in a corrupt header | Rejected *before* allocating memory |

### 10.6 Test plan (written first or alongside)
1. **Unit:** record encode/decode round trip; CRC detection; size-limit validation.
2. **Functional:** put/get/delete/overwrite; keys iteration; binary-safe keys and values (including `\x00`).
3. **Persistence:** write, close, reopen, compare. Repeat across segment rollover.
4. **Crash (real):** a child process writes in a loop, logging each *acknowledged* key to a side file; the parent hard-kills it (`TerminateProcess`) at random moments; reopen and assert every acknowledged key is present and nothing unacknowledged is corrupt. Run hundreds of iterations.
5. **Crash (simulated):** a fault-injecting `FileBackend` that drops or truncates writes at every byte boundary of a record.
6. **Property-based (`hypothesis`):** random sequences of put/delete/reopen compared against a plain Python `dict` model. The dict is the oracle.
7. **Corruption/fuzz:** flip random bits and truncate random files, require either clean recovery or a `CorruptionError`, never a hang, crash, or wrong value.
8. **Concurrency:** N threads writing and reading; final state must match a serial model; two-process lock test.
9. **Limits/soak:** many keys, large values, rollover at small segment size to exercise the path quickly.

### 10.7 Acceptance criteria for M1
- All tests in 10.6 pass, and I report the **actual pytest output** (pass/fail counts, number of crash-test iterations, any failures found and fixed).
- `zvdb-verify` tool scans a DB and reports OK or exact corruption location.
- `docs/specs/zvdb-v0.1.md` matches the code.
- `LIMITATIONS.md` lists everything not covered (e.g., keys must fit in RAM; fsync semantics depend on hardware honouring flush; no protection against malicious tampering).
- **Important caveat I will state in the report:** hard-killing a process tests *process* crashes. It does **not** prove survival of true power loss, because the OS cache survives a process kill. True power-loss behaviour can only be approximated through the simulated fault-injection backend, and the report will say so.

### 10.8 Implementation order within M1 (each step reviewable)
1. Scaffolding, lint, test runner, `common` errors and CRC helper
2. Record codec + unit tests
3. `FileBackend` (local-disk implementation) + fault-injecting test backend
4. Log append + index + `get/put/delete` (no recovery yet) + functional tests
5. Recovery + persistence tests
6. Locking + concurrency tests
7. Crash and fuzz suites
8. `verify` and inspect tools, docs, limitations, review request

## 11. Open questions for you (approval needed)

1. **Python version:** OK to target 3.11+? (Depends on what your inspection output shows.)
2. **ZVDB data model:** approve the Bitcask-style KV start (keys in RAM), or do you want a B+tree/LSM design from the beginning, accepting a steeper learning curve?
3. **Default durability:** `fsync` on every write (safe, slower) as default?
4. **Dependencies:** OK with `pytest`, `hypothesis`, `ruff`, `mypy` for dev, and `cryptography` later for ZVSEC/ZVNET? (None are storage engines.)
5. **Git:** do you already have a repo or GitHub account, or should I plan for local-only Git?
6. **Rust:** do you want the toolchain installed now, or keep it deferred to M8?

When you reply with your approvals or edits, plus the PowerShell output, I'll start at M0/M1 step 1.
