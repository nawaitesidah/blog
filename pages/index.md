# Welcome!

This blog is to share what I found interesting in Software Engineering.

Feel free to leave a comment [here](https://github.com/nawaitesidah/blog/discussions) 

--

## Load balancing: Kubernetes Service, Istio, and Kong

*2024-12-13*

I've always wondered what service mesh like Istio actually do. How it compared to Kubernetes' Service, and unexpectedly how it relates to Kong.

[Read more](?load-balancing-kubernetes-service-istio-and-kong)

--

## Postgres’ Dead Tuples

*2024-10-24*

When we delete a tuple, Postgres still keep the tuple in the disk, only marking it as deleted. This is called dead tuples.

Dead tuples uses extra space, so when there’s many deletes (and updates!), even without any new inserts, postgres will use more and more disk space. 

The question become

> Why is there a need for dead tuple? Why not just actually delete it from the disk?

[Read more](?postgres-dead-tuple)

--

