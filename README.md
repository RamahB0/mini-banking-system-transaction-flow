# Mini Banking System - Transaction Flow and Distribution Planning

Checkpoint answers for the "Computer Systems and Their Fundamentals: Database
Management systems" Super Skill, covering concurrency control for a
transfer-between-accounts scenario (Part 1) and distributed database design
for a three-branch bank (Part 2).

---

## Part 1 - Transaction Management (Conceptual)

### Scenario

Two users simultaneously initiate transfers that both touch the same
account (Account A): User 1 transfers $100 from A to B, while at the same
time User 2 transfers $50 into A from C.

### 1. Concurrency issue

This is a classic Lost Update scenario. If both transactions read
Account A's balance before either one writes its result back, the second
transaction's write overwrites the first transaction's write using stale
data, the first transaction's update is silently lost, and the account
ends up with the wrong balance. (If one transaction reads A's balance while
the other has written but not yet committed its own change, that also opens
the door to a Dirty Read, compounding the problem.)

### 2. Proposed locking mechanism

Use an Exclusive (X) lock on the account row for any transaction that
intends to write to the balance (debit or credit). Any transaction whose
sole purpose is view balances (a read-only operation) can take a
Shared (S) lock instead, so multiple balance-viewers can proceed
concurrently, but the moment a transaction needs to update the balance, it
must upgrade to (or directly acquire) an exclusive lock, which blocks every
other reader/writer on that same account row until it commits and releases
the lock.

### 3. Pessimistic vs. optimistic locking

Pessimistic locking is the better fit here. Optimistic locking assumes
conflicts are rare and only checks for them at commit time, acceptable
for something like a user editing their own profile. A shared bank account,
however, is a hot resource that can plausibly be touched by multiple
transfers around the same moment, so the chance of a genuine conflict is
not rare, and the cost of getting it wrong (an incorrect balance) is high.
Pessimistic locking prevents the conflict from ever happening by locking
the row as soon as it's accessed, rather than letting both transactions run
optimistically and aborting one after the damage could already be
committed.

### 4. Order of operations

Unsafe schedule (no locking) - reproduces the Lost Update:

| Time | T1: Transfer $100 (A to B) | T2: Transfer $50 (C to A) | Account A balance |
|------|----------------------------|----------------------------|--------------------|
| t1 | Read A = 500 | | 500 |
| t2 | | Read A = 500 | 500 |
| t3 | A = 500 - 100 = 400 | | 500 (uncommitted) |
| t4 | Write A = 400, commit | | 400 |
| t5 | | A = 500 + 50 = 550 (based on stale read) | 400 |
| t6 | | Write A = 550, commit | 550 - WRONG |

Verdict: Unsafe. The correct balance should be 500 - 100 + 50 = 450,
but T2 overwrote T1's debit because it computed its new value from a
balance it read before T1's write. This schedule is not conflict
serializable.

Safe schedule (exclusive locking enforced):

| Time | T1: Transfer $100 (A to B) | T2: Transfer $50 (C to A) | Account A balance |
|------|----------------------------|----------------------------|--------------------|
| t1 | Acquire X-lock(A); Read A = 500 | Request X-lock(A) - blocked | 500 |
| t2 | A = 500 - 100 = 400; Write A = 400 | (waiting) | 400 |
| t3 | Commit; release X-lock(A) | (waiting) | 400 |
| t4 | | Acquire X-lock(A); Read A = 400 | 400 |
| t5 | | A = 400 + 50 = 450; Write A = 450 | 450 |
| t6 | | Commit; release X-lock(A) | 450 - correct |

Verdict: Safe. Forcing T2 to wait for T1's exclusive lock makes this
schedule equivalent to the serial order T1 then T2, so it is conflict
serializable and produces the correct final balance.

---

## Part 2 - Distributed Database Planning (High-Level)

Branches: Tunis, Sousse, Sfax.

### 1. Horizontal fragmentation of Customers

Fragment the Customers table by branch, one fragment per branch:

- Customers_Tunis -> SELECT * FROM Customers WHERE branch = 'Tunis'
- Customers_Sousse -> SELECT * FROM Customers WHERE branch = 'Sousse'
- Customers_Sfax -> SELECT * FROM Customers WHERE branch = 'Sfax'

Each fragment shares the same schema but holds only the rows for its own
branch, stored at (or near) that branch's server. This satisfies:

- Completeness - every customer row belongs to exactly one branch fragment.
- Disjointness - a customer appears in only one branch's fragment.
- Reconstructability - UNION of the three fragments rebuilds the full table.

This keeps day-to-day teller lookups local to the branch that services most
of a customer's activity, improving data locality and cutting inter-site
network traffic.

### 2. Column to vertically separate

Split out contact/login information - email, phone number, and login
credentials - into a separate CustomerContacts table keyed by
customer_id (PK/FK back to Customers). Core identity fields
(customer_id, name, branch, national_id) stay in Customers, while
the rarely-joined, more sensitive contact/auth fields move out. This keeps
routine queries (e.g., "look up a customer by ID") lighter, and lets
stricter access control/encryption be applied specifically to the
contact/credentials fragment.

### 3. What should be replicated across all branches?

- Customer info (core identity only): Replicate lightly across
  branches - a customer might visit a branch other than their home branch,
  so having basic identity data (name, customer_id, home branch) available
  everywhere speeds up cross-branch lookups. This data is read-heavy and
  changes rarely, which is exactly the profile that benefits from
  replication.
- Account balances: Should not be fully replicated. Balances are
  write-heavy and require strong consistency - replicating them everywhere
  would mean constant synchronization on every transaction and a real risk
  of a branch reading a stale balance. Keep the authoritative balance at
  the account's owning branch/fragment (allocation, not replication); other
  branches query it live rather than hold their own copy.
- Transaction history: Partially replicate - send a copy to a small
  number of sites (e.g., a central head-office/reporting server) for
  auditing and compliance, rather than to every branch. It's written once
  and read for reporting, so it tolerates replication far better than
  balances do, but full replication at every branch is still unnecessary
  overhead.

### 4. Static or dynamic allocation for transaction history?

Static allocation. Each transaction record naturally belongs to the
branch where it originated, and once written it is immutable/append-only,
its access pattern (audits, statements, compliance reports) doesn't shift
over time the way a hot mutable resource would. Dynamic allocation earns
its complexity when data needs to migrate toward changing workloads;
historical transaction records don't have that need, so the simplicity of
assigning each branch's history statically to that branch's site (plus the
replicated central audit copy from Part 3) is the better trade-off here.
