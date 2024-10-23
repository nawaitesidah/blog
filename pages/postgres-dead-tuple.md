# Postgres’ Dead Tuples

*2024-10-24*

When we delete a tuple, Postgres still keep the tuple in the disk, only marking it as deleted. This is called dead tuples.

Dead tuples uses extra space, so when there’s many deletes (and updates!), even without any new inserts, postgres will use more and more disk space. 

The question become

> ### Why is there a need for dead tuple? Why not just actually delete it from the disk?
>

There’s two reasons:

1. Physical storage layout
    - Postgres does not physically store a tuple into a file in the disk, but postgres stores a group of tuples together into a file, which is called page.
    
    A page is 8KB in size, so there could be multiple tuples stored together in a page.
    - Due to this, postgres can’t delete just one row. Otherwise postgres would need to rewrite a whole page excluding the deleted rows, which will be inefficient.
    
    PS. This is actually what VACUUM FULL does, rewriting pages without dead tuples.
2. MVCC (multiversion concurrency control)
    - Postgres uses MVCC, it keeps multiple version of a row, so that a transaction can return a consistent result.
    
    Example: When using repeatable read isolation level, even if other transaction has deleted a row and committed, the repeatable read transaction will still see the data, exactly the same way as when it started the transaction.
    - In order to do that, postgres need to keep the deleted tuples, just for the existing transactions to read.
    
    - **This is the case for update too!**
    
    Example: In a repeatable read transaction, we’ll read the same value as long as we’re still in that transaction, regardless of what other transactions did to that tuple.
    
    This is possible because postgres do not update the tuple directly on the page. But postgres handle update by doing delete on the old tuple (marked as dead tuple) and insert the updated rows as new tuples.

The next question will be:

> ### Why postgres can’t just reuse the dead tuple space when inserting a new tuple?
>

If postgres already know that there’s a dead tuple, why not just reuse the space, and write the new tuple over the dead tuple?

Unfortunately, on writing the new tuple postgres has no direct way to know whether a tuple can still be accessed by any other transaction or not.

Consider this, a tuple was deleted an hour ago, and now there’s a new tuple to be written. On write, we have to be sure there’s no transaction that can still read this row e.g. repeatable read transaction started before the tuple was deleted.

> ### How can postgres make sure that the dead tuple is not used by any other active transactions?
>

Without checking for all other transactions, there’s no way to be sure.

But if postgres checks for all other transactions on every writes, then it’ll be inefficient

> ### Then how to actually reuse the disk space of dead tuples?
>

Here’s where VACUUM is used. Vacuum goes through every page to find the dead tuples, then it checks whether there’s any active transaction that still can access the dead tuples or no. Then it’ll rewrite a new page without the dead tuples.

In contrast, VACUUM FULL rewrites the whole pages and excluding the dead tuples. So the new page will only have live tuples, this frees the disk space from the dead tuple.
