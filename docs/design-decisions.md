## A. Entities
### Book

Field	|Type	|Rule	|Who may change it
|-------|-------|-------|-----------------|
book_id	|string, B001 format	|Unique, never reused, never edited after creation	|Set once by books.py at creation
title	|string	|Not blank after trimming	|books.py only
author	|string	|Not blank after trimming	|books.py only
category	|string	|One of the canonical categories (normalised before storing)	|books.py only
status	|string	|available, on_loan or lost	|See ownership note below
notes	|list of dicts	|Each note has note_id, author and message	|notes.py only


### Member
Field | Type | Rule | who can change it
------|------|------|------------------
member_id	|string, M001 format	|Unique, never reused, never edited after creation	|Set once by members.py at creation
name	|string	|Not blank after trimming. Dummy names only, no real personal data	|members.py only
status	|string	|active or suspended	|members.py only (suspend and reactivate actions)


### Loan
Field | Type | Rule | who can change it
------|------|------|------------------
loan_id	|string, L001 format	|Unique, never reused, never edited after creation	|Set once by loans.py at creation
book_id	|string	|Must refer to an existing book	|Never changed after creation
member_id	|string	Must refer to an existing member	|Never changed after creation
loan_date	|ISO date string (YYYY-MM-DD)	|Valid date, supplied by the caller	Never changed after creation
due_date	|ISO date string	|loan_date plus 14 days at creation, plus 7 days on a valid renewal	|loans.py (create, renew) only
return_date|	ISO date string or None	None while active. |When set, must not be before loan_date	|loans.py (return) only
renewals	|integer	|Whole number, 0 or 1	|loans.py (renew) only

