# Welcome!

This blog is to share what I found interesting in Software Engineering.

--

## Postgres’ Dead Tuples

*2024-10-24*

When we delete a tuple, Postgres still keep the tuple in the disk, only marking it as deleted. This is called dead tuples.

Dead tuples uses extra space, so when there’s many deletes (and updates!), even without any new inserts, postgres will use more and more disk space. 

Then the question become

> Why is there a need for dead tuple? Why not just actually delete it from the disk?

[Read more](?postgres-dead-tuple)

--

