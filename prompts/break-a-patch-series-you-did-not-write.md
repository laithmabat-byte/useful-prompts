# Break a Patch Series You Did Not Write Prompt

A brief for an AI coding agent that tries to break a prepared patch series
before it merges. The author's tests show what the author thought of. This
brief looks for the rest: the base that moved, the mock that hides the real
framework, and the claim nobody ran.

Use it on any series that was prepared but not yet merged, when the reviewer
did not write it. It needs no production access.

## Prompt

```text
You did not write this patch series. Try to break it before it merges.

Series: <path to patches and notes>
Base: <branch or commit>
Questions to answer: <the specific risks you want checked>

Rules:
- Work in your own throwaway clone. Do not touch the author's clone.
- No real systems, no secrets, no heavy jobs.
- Do not change the author's patches. Report; they fix.

Do this:
1. Read the notes and every patch before running anything. List each claim,
   and mark what the notes say was not run.
2. Re-read the base now. Apply the series to the real current tip, not only the
   commit it was cut on. A base that moved is a finding.
3. Run the author's tests and confirm both results: red at the tests commit,
   green at the tip. If a harness has a safety proof, confirm it printed first.
4. Find where the tests stand in for something real: a stub that never runs the
   command, a mock of the framework, a function the test never loads. Then call
   the real code for that step, without a site or a network if you can.
5. Trace what happens when the new code fails: who catches it, and what is lost.
6. Attack the time, order and state assumptions: frozen clocks against real
   timers, queued work that also runs, empty and short data, duplicates.
7. Write your own mutants, on copies. Prove the unmutated copy passes first, and
   check any kill is for the right reason, not a syntax error in the mutant.
8. Run the project's linter, and compare against the base so old hits are not
   counted as new.
9. Apply all prepared series together, in two orders, and compare the trees.

Before reporting, separate what you ran from what you read. If your own harness
misled you, keep that output and say so.

Report with one verdict line: merge, or block with a concrete failing input.
Then a table of question, answer and how you know it, what ran, what is worth
knowing but not a blocker, and what still needs a real environment. A finding
needs an input and a result, and says whether that input can be reached.
```
