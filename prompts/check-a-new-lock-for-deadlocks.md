# Check a New Lock for Deadlocks

Use when a change adds a row lock, an explicit `SELECT ... FOR UPDATE`, or a loop that writes several rows in one transaction, and you need to know whether it can deadlock with the rest of the system. It suits any relational database with row locks. The examples use Postgres.

## Prompt

> A change adds or widens locking in `<function or code path>`. Find out whether it can deadlock with anything else that writes the same rows, how often, and what a loss costs. Work only in scratch databases you create and drop yourself. Never touch a shared test database or its template, and never run against a real system.
>
> 1. **List every writer of the locked tables.** Search the schema and the app for every function, trigger, scheduled job and request path that updates, deletes from or locks the tables involved, and include background jobs like retention or cleanup. For each one, write down the order in which it takes row locks: which table first, and in what row order (sorted, by id, or arbitrary). Remember that an `UPDATE ... FROM other_table` locks only the target table's rows, and that a trigger can add reads, but not locks, on another table.
> 2. **Find the opposite orders.** Two writers that take table X then Y, and Y then X, on the same rows can deadlock. So can two writers that lock several rows of one table in different row orders. Make a table: writer, lock order, rows. Mark every pair with an opposite order.
> 3. **Build fixtures at three points:** the base branch, the change without its new lock (if the change came in rounds), and the full change. Create each from a template copy, apply the migrations as the real owner role, and load one fixture that puts a single client in the state both writers act on: for example, a draft that is past its expiry date, and a second document that can be erased.
> 4. **Force each interleaving.** Use a third session to hold one row (`SELECT ... FOR UPDATE`, then sleep). Start writer A so it locks what it can and queues. Wait until `pg_stat_activity` shows it waiting on a lock. Then start writer B, wait again, and let the third session commit. Run both queue orders. Wait on lock state, not on fixed sleeps.
> 5. **Measure without forcing.** Start both writers at the same moment, 20 or more times per fixture, and count: deadlocks where A lost, deadlocks where B lost, and runs where both finished. Compare the base, the change without the lock, and the full change, so you can say whether the lock caused it, made it worse, or only moved which side loses. Say that starting both at once overstates the real rate.
> 6. **Price a loss for each side.** Is that side one transaction, so a loss rolls back unrelated work, such as a daily job for every customer? Does the caller retry on deadlock, or map the error to a response? Show it with one forced run that includes a second, unrelated record the losing job should have handled.
> 7. **Try the smallest fix.** Usually that means making the odd writer take its locks in the shared order: lock the rows it may touch first, sorted by key, before it updates the other table, and sort its loops. Prove it: 0 deadlocks in the same runs, the forced interleavings all finish, and a plain run with no concurrency leaves identical state, side effects and audit rows with and without the fix. Keep the security attributes and owner of any replaced function the same.
> 8. **Report** one verdict line, the lock-order table, the counts table for each fixture, the cost of each kind of loss, the fix and its proof, and what you did not run (tests first, mutants, the full suite). Drop every scratch database and confirm none is left.

## Notes

- Name scratch databases with a prefix only you use, and refuse to create or drop any other name in your helper.
- Back-dating rows past a trigger, for example with `session_replication_role = replica`, belongs in scratch fixtures only. It never belongs in application code or in a migration.
- Check what else is running on the shared database server before copying a template: a release may be rebuilding it.
