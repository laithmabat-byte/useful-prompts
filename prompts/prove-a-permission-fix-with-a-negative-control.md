# Prove a Permission Fix with a Negative Control Prompt

A brief for an AI coding agent that fixes a role-permission defect ("the
cashier gets a 'No permission for X' popup", "the sale is refused at submit")
and proves the fix on a real server and in a real browser, in a way that cannot
pass by accident. Permission bugs hide from unit tests (the test user holds
more roles than the real one) and from browser scripts that only open a page
(the refusal comes later, on an action). The proof here has four parts: the
smallest set of rights found by reading code, a test that fails on the old role,
a mutation run that reverts each piece of the fix, and a browser run as the
real low-privilege person with a negative control: the same script on the role
as it was before the fix must FAIL, with the very refusal the user reported.

Written for a framework with role-based permission rows stored in the database
(Frappe/ERPNext "Custom DocPerm" is the example), a session user with one role,
and a browser-driven UI. Adjust the names to the stack.

---

## Copy-Paste Prompt

````markdown
ROLE
You fix one permission defect with the least right that works, and you prove it
so that the proof would have failed before the fix. You do not push or merge.
Anything you did not run is written "NOT RUN".

INPUTS
- The defect as the user saw it: the role, the screen, the exact message.
- A test site with the app installed, and a throwaway scratch site for the
  browser run (one at a time; delete it afterwards).

STEPS
1. FIND THE EXACT PATH. Read the code that raises the message and the code path
   of the whole action (open, sell, pay, print, list, close), not only the first
   refusal: after the first right is granted the next one appears. For every
   refusal record the document type, the right (read/write/create/submit/export)
   and the file:line that asks for it. List what else the same user can then do
   through the plain API, and say so.
2. GRANT THE MINIMUM. Add only the rights the path needs; switch off the rights
   that a role row gets by default (export, read) where the path does not need
   them. Never replace the standard rows of a document type for other roles
   (copy them first). Where a right is too wide for what the action needs
   (create/write/submit on a record the user should only make as part of a
   sale), add a permission hook that denies everything except that case. Do not
   edit the framework or a vendored app; if that is the cleaner fix, write it
   down as a proposal.
3. MAKE IT REACH EXISTING SITES. An idempotent patch gives the same rows to a
   site built before the fix and does nothing where the role is absent or the
   row exists.
4. TESTS THAT FAIL WITHOUT THE FIX. Rebuild the old role in the test (patch the
   preset or delete the rows), assert the refusal with its exact exception, then
   build the new role and assert the action completes. Judge the premise by the
   rights a user holds, not by a list of role names: a full-app site adds roles.
   Test the walls too: what the user must still NOT do.
5. MUTATIONS. For each piece of the fix (each right, each switched-off right, the
   hook and each branch of it, the patch, its registration) revert it in a copy
   of the app on the bench, run the one test that guards it, and require FAILED;
   then restore. A survivor is red: add a test, never remove the mutation.
6. BROWSER PROOF WITH A NEGATIVE CONTROL. A script logs in as a person who holds
   ONLY that role, and performs the whole action. It must fail on any permission
   refusal and any message dialog: watch every server response (403 or an
   exception type of permission), and record dialogs with a MutationObserver
   from the first byte of the page so one that closes at once is seen. Prove the
   result from the server (the count of submitted documents grew), not from text
   on the page. Run it (a) on the role rebuilt as it was before the fix: it must
   FAIL with the user's reported message; (b) on the fixed role: it must pass, in
   each language and on a phone-sized viewport; (c) on every variant of the data
   the path depends on (an item with batches or serial numbers, one without).
   Keep screenshots of each phase of both runs.
7. DO NOT WEAKEN A CHECK to make a run green. A script race (a click before the
   page rebinds its handler) is fixed by waiting for the page, and a framework
   quirk that needs no permission is recorded as a finding, not hidden.

REPORT
Commit shas, one line per test module, each mutation and its verdict, the
negative-control run (the refusal it raised) and the passing runs, what the
user can do through the plain API now, and what did not run.
````
