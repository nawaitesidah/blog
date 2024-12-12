# Load balancing: Kubernetes Service, Istio, and Kong

*2024-12-13*

I've always wondered what service mesh like Istio actually do. How it compared to Kubernetes' Service, and unexpectedly how it relates to Kong.

From the documentation:

> **Istio is an open source service mesh that layers transparently onto existing distributed applications. Istio’s powerful features provide a uniform and more efficient way to secure, connect, and monitor services. Istio is the path to load balancing, service-to-service authentication, and monitoring – with few or no service code changes**

We can see that it adds a lots of stuff, but lets focus on the load balancing. In kubernetes environment, we already have Service resource for this, where connection to the service is load balanced to the pod matching the service selector.

## So, if we already have kubernetes service, why would we still need istio for load balancing?

The answer lies in which layer the load balance happen.

For kubernetes service, it happens on Layer 4. Which means it happen when establishing TCP connection. So it'll load balance the traffic when a TCP connection is established. 

### Nice, it load balances my traffic, end of story right…?

The thing is, some protocol can reuse TCP connection, e.g. HTTP/1.1 with the keep-alive connections, HTTP/2, gRPC, WebSocket.

So, once a connection is established, it will be reused to handle multiple subsequent traffic. This is to prevent the overhead of establishing new connection.

But it can also causes some **unexpected issue** for load balancing. Consider that if we reuse a set of TCP connections created to a kubernetes deployments with 2 replicas. What will happen if we spawn a new replica?

For example, a deployment in Kubernetes are exposed via kong gateway, by using the kubernetes Service e.g. `example-service.namespace.svc.cluster.local` as the upstream target. And initially there's only two replicas of this deployment, then sometime later the deployment is increased to 3 pods

When kong receives traffic, it'll first open a TCP connection to any of the two pods and forward the request through it. On subsequent request, kong will reuse this connection again. Kong will reuse this connection for 10000 times, which is the default value of `upstream_keepalive_max_request`.

This means kong will have to go through these 10000 requests to the old pods first before creating a new connection. 

*(Or if there's a new incoming request and all of the existing connections are busy, then kong will spawn the new connection to the service, or until the connection is idle for 60 seconds, which is the default `upstream_keepalive_idle_timeout`.)*

Let's consider an extreme case, where the API traffic is 6000 RPM and moderate amount of concurrent requests, lets say it's a stable 30 concurrent client requests at all times, this means at the start kong will have at least 15 connections established to two existing pods, each with a quota of 10000 requests.

With these values, it'll exhaust the connections after 50 minutes! (10000 request quota x 30 established connections / 6000 rpm).

Then after exhausting the connection, it'll establish a new connection to **one random pod (yes, RANDOM)**.

## Kubernestes Service

The default Service in kubernetes (Cluster IP Service), is actually backed by iptables. And the ip tables are randomly routing between the available pods, not round robin, and certainly not least used.

Here's the config to be exact:

```
Chain KUBE-SVC-XXXXXXXXXXXXXXX (1 references)
 pkts bytes target                prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ         tcp  --  *      *      !10.128.0.0/16       10.128.50.100        /* example-service.namespace.svc.cluster.local cluster IP */ tcp dpt:8080
    0     0 KUBE-SEP-AAAAAAAAAAAAA tcp  --  *      *       0.0.0.0/0           0.0.0.0/0            statistic mode random probability 0.33333333333
    0     0 KUBE-SEP-BBBBBBBBBBBBB tcp  --  *      *       0.0.0.0/0           0.0.0.0/0            statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-CCCCCCCCCCCCC tcp  --  *      *       0.0.0.0/0           0.0.0.0/0            

Chain KUBE-SEP-AAAAAAAAAAAAA (1 references)
 pkts bytes target                prot opt in     out     source               destination         
    0     0 DNAT                  tcp  --  *      *       0.0.0.0/0           10.128.60.10         /* pod-1 for example-service */ tcp to:10.128.60.10:8080

Chain KUBE-SEP-BBBBBBBBBBBBB (1 references)
 pkts bytes target                prot opt in     out     source               destination         
    0     0 DNAT                  tcp  --  *      *       0.0.0.0/0           10.128.60.11         /* pod-2 for example-service */ tcp to:10.128.60.11:8080

Chain KUBE-SEP-CCCCCCCCCCCCC (1 references)
 pkts bytes target                prot opt in     out     source               destination         
    0     0 DNAT                  tcp  --  *      *       0.0.0.0/0           10.128.60.12         /* pod-3 for example-service */ tcp to:10.128.60.12:8080
```

So when kong establish connection to `example-service.namespace.svc.cluster.local`, it'll go through these steps:

1. Roll a 33% chance to be resolved as the first pod’s IP.
2. Otherwise roll a 50% chance to resolve as the second pod's IP.
3. Otherwise it'll be resolved to the third pod.

And once connected, it'll be reused for 10000 times again. So Kubernetes Service may not the best tool to load balance traffic, especially long lived connections.

## Enter istio

Istio load balancer works on Layer 7. This means that it's aware of protocols like HTTP/1.1, HTTP/2, or gRPC.

So on **each individual request**, istio will route it to the corresponding pods, unlike Kubernetes service that only load balance on establishing new TCP connections. This will lead to each pods having more balanced loads.

If you need to load balance long lived connections, you'll need L7 load balancer like istio.

*Note: Alternatively we can reduce the `upstream_keepalive_max_request` value. So that kong will be more frequently establish new connections, and hopefully will randomly land to a less used pods.*

[Home](?index)
