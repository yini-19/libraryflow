## 2A. Entities

### Book

| Field | Type | Rule | Who may change it |
|---|---|---|---|
| book_id | string, B001 format | Unique, never reused, never edited after creation | Set once by books.py at creation |
| title | string | Not blank after trimming | books.py only |
| author | string | Not blank after trimming | books.py only |
| category | string | One of the canonical categories (normalised before storing) | books.py only |
| status | string | available, on_loan or lost | See 2F |
| notes | list of dicts | Each note has note_id, author and message | notes.py only |

### Member

| Field | Type | Rule | Who may change it |
|---|---|---|---|
| member_id | string, M001 format | Unique, never reused, never edited after creation | Set once by members.py at creation |
| name | string | Not blank after trimming. Dummy names only, no real personal data | members.py only |
| status | string | active or suspended | members.py only (suspend and reactivate actions) |

### Loan

| Field | Type | Rule | Who may change it |
|---|---|---|---|
| loan_id | string, L001 format | Unique, never reused, never edited after creation | Set once by loans.py at creation |
| book_id | string | Must refer to an existing book | Never changed after creation |
| member_id | string | Must refer to an existing member | Never changed after creation |
| loan_date | ISO date string (YYYY-MM-DD) | Valid date, supplied by the caller | Never changed after creation |
| due_date | ISO date string | loan_date plus 14 days at creation, plus 7 days on a valid renewal | loans.py (create, renew) only |
| return_date | ISO date string or None | None while active. When set, must not be before loan_date | loans.py (return) only |
| renewals | integer | Whole number, 0 or 1 | loans.py (renew) only |

## 2B: Where Active Loan State Lives

Oct 28, 2026

### Decision

A loan's active or returned state is not stored. It is derived from `return_date`:

- `return_date` is `None`: the loan is **active**.
- `return_date` holds a date: the loan is **returned**.

### Related derived values

Two other facts are worked out the same way and never stored:

- **Overdue:** the loan is active and `due_date` is earlier than the `today` value passed into the function.
- **A member's active loan count:** the number of loans with that `member_id` and no `return_date`. The member record keeps no loan list of its own.

### Why this choice

Deriving state from the date removes duplicate facts, and duplicates are where contradictions start.

1. **One source of truth.** A stored `state` field next to `return_date` could disagree with it, such as a "returned" loan with no date. Deriving state makes that impossible.
2. **No stale data.** An `overdue` flag would be wrong the day after it was saved. Computing it from `today` keeps results correct and testable.
3. **Simpler JSON validation.** On load there is no second field to cross-check against `return_date`.
4. **Supports the invariants.** Invariant 1 (every active loan's book is `on_loan`) and invariant 3 (at most 3 active loans per member) both need one reliable definition of "active".

### Trade-off accepted

Counting a member's active loans means scanning the loan collection each time. At this project's size the cost is negligible, and it buys the removal of a duplicate fact.

### How the decision is enforced

- Only `loans.py` may set `return_date`, and only inside the return action.
- A return date earlier than `loan_date` is rejected.
- Loading JSON checks that `return_date` is `None` or a valid date that is not earlier than `loan_date`.

### Test cases this decision creates

| Case | Expected result |
| --- | --- |
| Return a loan | It stops counting as active with no other field updated |
| Member returns one of 3 active loans | The returned loan no longer counts toward the 3-loan limit |
| Loan past `due_date` with `return_date` set | Not overdue |
| Loan past `due_date` with `return_date` empty | Overdue |



## 2C. Error convention

**Decision:** Every rule failure is reported by raising an exception. No function returns `False`, `None` or an error string to signal failure.

**Exception types** (all share one base class, `LibraryFlowError`):
- `ValidationError`: malformed input (blank title, bad category, note under 5 characters, invalid date)
- `NotFoundError`: a referenced book, member or loan does not exist
- `RuleViolationError`: valid input, but a rule forbids it (fourth loan, lending an `on_loan` book, second renewal)
- `StorageError`: a file cannot be loaded safely (broken JSON, duplicate IDs, loan pointing to a missing book)

**Why exceptions:**
- A failure cannot be ignored the way a returned `False` can.
- The CLI uses one pattern for every action: try, catch `LibraryFlowError`, print the reason.
- Tests are direct: `assertRaises`, then check the data did not change.

**Rules:**
- Logic modules never print or read input. Only the CLI catches and prints.
- Every exception is raised before the first change to data, so a failed action changes nothing and a rejected note uses no ID.
- The CLI catches only `LibraryFlowError`, so real bugs are not hidden.

**Test pattern:** every state-changing rule gets two tests: one asserting the exception, one asserting the data equals a copy taken before the call.

## 2D. Mutate in place, or work on a copy?

**Decision:** Functions change the existing collections in place, but only after every check has passed.

**How a change is made:**
1. Run all checks (existence, status, limits, dates). Raise on the first failure.
2. Compute everything the change needs (new ID, due date) before touching data.
3. Apply the changes in one block with nothing in between that can fail.

**Lend example:** the new loan is added and the book is set to `on_loan` back to back. The checks and the new `loan_id` are done first.

**Why in place:**
- Simple, and matches the validate-before-mutate habit from CampusFlow.
- No copying or swapping of whole collections.

**Trade-off:** A bug in step 3 could leave data half changed. Keeping step 3 to plain assignments, with no calls that can raise, keeps that risk small.

**How it is proven:** tests take a `copy.deepcopy` of the data before a failing call and assert the data equals it afterwards.

## 2E. Time handling

**Decision:** Any function that depends on "today" receives the date as an input. Logic modules never read the system clock.

**Where the clock is read:** only in the CLI layer, once per action. It passes that date down (for example, `today="2026-10-28"`).

**Date format:** ISO strings (`YYYY-MM-DD`), validated on input and on JSON load. Dates are compared as dates, never as raw text.

**Functions that take `today`:** lend, return, renew, mark lost, overdue check, Overdue Queue, and the member borrowing check (overdue loans block borrowing).

**Date rules:**
- Due date = loan date + 14 days. A renewal adds 7 days.
- A loan is overdue only when `due_date` is earlier than `today`. A loan due today is not overdue.
- A return date cannot be earlier than the loan date, and cannot be later than `today`.
- A loan date cannot be later than `today`.

**Why:**
- Tests can use fixed dates, so overdue behaviour is checked without waiting days.
- Results are repeatable. The same input always gives the same output.
- A hidden clock read inside a function is the kind of dependency that makes tests fail only on certain days.

**Tests:**
- A loan checked the day before, on, and the day after its due date: not overdue, not overdue, overdue.
- A return dated before the loan date is rejected and data is unchanged.
- A loan created on the last day of a month gets the correct due date.
- A renewal of an overdue loan is rejected.

## 2F. Marking a book lost while it has an active loan

**Decision:** `books.py` rejects marking a book lost if the book is currently `on_loan`. The book must be returned first (via `loans.py`), then it can be marked lost.

**Reason:** The book's status is written by two modules. `loans.py` owns `available` and `on_loan`, and `books.py` owns `lost` and recover. If a lost book could still have an active loan, returning it would set it back to `available` and silently overwrite `lost`. Rejecting the combination removes that case, so neither module has to know about the other's states.

**Status ownership**

| Transition | Owner | Allowed when |
|---|---|---|
| available -> on_loan | loans.py (lend) | no checks fail |
| on_loan -> available | loans.py (return) | an active loan exists for the book |
| available -> lost | books.py | book is `available` |
| lost -> available | books.py (recover) | book is `lost` |

Any other transition is rejected. In particular, `on_loan -> lost` is rejected, and `lost -> on_loan` is impossible because lost books can't be loaned.

**Invariants this creates**
1. A book with status `lost` has no active loan.
2. A book has status `on_loan` if and only if exactly one active loan references it.
3. A book never has more than one active loan.

**Consequences**
- `books.py` needs to know whether a book is on loan. It reads only the book's own status field, so it does not need to query loans, and there is no circular dependency between `books.py` and `loans.py`.
- Recover is simple: only a `lost` book can be recovered, and it can't have a loan, so it always goes to `available`.
- JSON load must check invariants 1 to 3 across the book and loan collections, because a hand-edited file could break them. A violation fails the whole load and leaves memory unchanged.

**Trade-off (deliberate scope limit):** In a real library a book is often lost *by* a member while on loan. This design doesn't model that. Supporting it would need a new loan outcome (for example "closed as lost") plus rules for what recover does to that loan. That is out of scope for this practice project.

**Tests implied**
- Mark an `available` book lost: succeeds.
- Mark an `on_loan` book lost: rejected, book and loan unchanged.
- Recover a `lost` book: becomes `available`.
- Recover a book that isn't lost: rejected.
- Lend a `lost` book: rejected.
- Load a file with a lost book that has an active loan: rejected, memory unchanged.

## 2G. Pure logic vs input/output

**Decision:** Logic modules never print or read input. Only `cli.py` and `understanding_check.py` talk to the user, and only `cli.py` reads the clock (see 2E).

**Pure logic (no `print`, no `input`):**
- `validation.py`: field and date validation helpers
- `books.py`, `members.py`: create, validate, change status
- `loans.py`: lend, return, renew, mark lost, recover, overdue check
- `notes.py`: add and read book notes
- `search.py`: filtering and sorting, returns a new list
- `reports.py`: builds report totals as data and a separate function that formats the lines
- `storage.py`: save, validate and load
- `understanding_questions.py`: question data only
- `understanding_check.py`: the scoring function only

**Thin layer (input and output allowed):**
- `cli.py`: shows menus, reads input, re-prompts on bad input, calls logic, catches `LibraryFlowError` (see 2C), prints results
- The interactive part of `understanding_check.py`: asks questions and collects answers, then calls the scoring function

**Rules:**
- Interactive functions take `input_fn` and `output_fn` parameters, so tests can pass fake ones instead of typing.
- Functions return data. They do not print it. Formatting into text lines is a separate function from calculating.
- The CLI contains no business rules. If a rule is checked in `cli.py`, it moves to a logic module.

**Why:**
- Logic that returns data can be tested directly, with no captured output or fake keyboard.
- A rule lives in one place, so a mentor change touches one function and its test.
- The CLI stays small, so there are fewer places for bugs.

**Tests:**
- Search, report totals and quiz scoring are tested by calling them with data and checking what they return.
- A CLI menu is tested with fake input, including an invalid choice such as `Z`, which must re-prompt instead of crashing.

## 2H. Save file format

**Decision:** One JSON file with an envelope: a version number plus a separate list for each collection.

```json
{
  "version": 1,
  "books": [
    {"book_id": "B001", "title": "...", "author": "...", "category": "Science",
     "status": "available",
     "notes": [{"note_id": "N001", "author": "Staff A", "message": "..."}]}
  ],
  "members": [
    {"member_id": "M001", "name": "...", "status": "active"}
  ],
  "loans": [
    {"loan_id": "L001", "book_id": "B001", "member_id": "M001",
     "loan_date": "2026-10-01", "due_date": "2026-10-15",
     "return_date": null, "renewals": 0}
  ]
}
```

**Why an envelope:**
- There are three collections, and a plain list cannot hold them.
- The `version` field lets a future change of format be detected and migrated instead of failing in confusing ways.

**Next IDs are not stored.** On load, the next book, member, loan and note IDs are worked out from the highest number already present.
- This avoids a stored counter that could disagree with the data.
- A rejected action never consumes an ID, because IDs come from records that exist.

**Loading rules:**
1. Parse the file. Broken JSON raises `StorageError`.
2. Check the version. An unsupported one raises `StorageError`.
3. Validate the whole structure before touching memory: required keys, legal statuses, valid ISO dates, unique IDs, note shape and unique note IDs per book, `renewals` of 0 or 1.
4. Check relationships: every loan points to an existing book and member, every active loan's book is `on_loan`, every `on_loan` book has exactly one active loan, no member has more than 3 active loans, no `lost` book has an active loan.
5. Only if every check passes, replace the data in memory. Otherwise memory is unchanged.

**Saving:** write every field, including notes and loans. Use dummy names only.

**Tests:**
- Save then reload preserves everything, and the next IDs do not collide.
- Broken JSON, duplicate IDs, a loan pointing to a missing book, and an unsupported version are each rejected, and memory is unchanged.
