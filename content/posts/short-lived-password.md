+++
date = '2025-09-11T01:21:25+03:30'
title = 'Short-lived password: a self interview'
showToC = true
[cover]
image = "https://substackcdn.com/image/fetch/$s_!Qh7q!,w_1040,h_545,c_fill,f_webp,q_auto:good,fl_progressive:steep,g_auto/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F936023c5-a3e3-4e25-8ca2-e8085d5c3def_1100x788.png"
+++

At my workplace, I was given a task: use short-lived kubeconfig for accessing Kubernetes. This was already implemented in other services in different ways, and I was asked to simply choose one of them and copy-paste it into the service I was working on. That was my plan — but when I read the code, lots of questions came to my mind about the decisions developers had made for implementing the feature.

After a while, I noticed that not only was I not copying their code into the codebase, but I was also analyzing their solutions, listing each one’s pros and cons, and even coming up with my own approach. It felt like a realistic system design interview: I was trying to identify problems, find single points of failures, and provide improvements — even if they introduced more complexity.

This blog aims to present the problem and all the solutions that came to my mind for solving it, along with their benefits and drawbacks.

---

## Problem

Imagine we have various services (like an API service or workers) that need to access a resource in order to perform their tasks. I’m using generic terms here to generalize the problem: the services can be APIs, workers, cron jobs, or anything else, and the resource could be a database, a Kubernetes cluster, or whatever.

The important point is: accessing the resource requires authentication.

Originally, this authentication was handled with a username/password or a token. Now, we want to make it short-lived. By short-lived, I mean it expires quickly, and you need to provide a new password or token to access the resource again.

To support this, we have a system called the short-lived password provider. You can call it to get a fresh password for the resource, valid for a default TTL. (It’s not central to the question, but worth mentioning: this provider itself also requires authentication, usually via public/private keys, mTLS, or another secure mechanism.)

The provider exposes an endpoint that returns a short-lived password for accessing the resource.

![problem](https://substackcdn.com/image/fetch/$s_!qLA4!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F62d89ef8-bf79-4492-b093-53bd662247ce_1506x740.png)
Now let’s walk through different solutions, step by step, and see how we can improve them.

---

## 1st solution: the simple API call

![1st solution: the simple API call](https://substackcdn.com/image/fetch/$s_!f0Dm!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F6c4e6761-95f8-43b0-830f-eb0c18d2ef32_1184x482.png)

The easiest and most straightforward solution is to call the short-lived password provider every time we need to access the resource. This works well if our services do not use the resource frequently.

In a horizontally scaled system (with multiple nodes handling the same responsibility), each node would request its own password on each usage. Think of it like a deployment with multiple pods — each pod gets its own password whenever it needs one.

| Pros                    | Cons                                                                                                                                                                                     |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| + simple implementation | - does not take advantage of the TTL; calls the provider every time<br> - each node calls the provider, creating high load <br>- blocking I/O due to waiting for the provider’s response |

---

## 2nd solution: add local cache

![2nd solution: add local cache](https://substackcdn.com/image/fetch/$s_!kO22!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F7d5f4806-593d-4811-95af-51816fea2394_1622x444.png)

We can improve the first design by taking advantage of the TTL. Instead of calling the provider every time, we store the password in memory (or on the node’s local storage) for the ttl duration. When it expires, we fetch a new one.

| Pros                                                        | Cons                                                                                      |
| ----------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| + fewer calls due to caching<br>+ less load on the provider | - each node still calls the provider separately, so the provider may still face high load |

---

## 3rd solution: add shared cache

![3rd solution: add shared cache](https://substackcdn.com/image/fetch/$s_!qN25!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fa9b3b177-a595-48c6-b7b1-23f97b4695e3_1454x494.png)

Instead of local cache, we can use a shared cache accessible from all nodes. This could be a shared PVC mounted into pods, or something like Redis.

The idea is similar to the 2nd solution, but now only one node needs to fetch the password from the provider — the rest can use the shared cache.

| Pros                                                                                            | Cons                                                                                                                                            |
| ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| + fewer calls to the provider<br> + provider called only once per TTL<br> + shared across nodes | - I/O overhead of calling shared cache each time (still faster than provider, but slower than local memory)<br> - high load on the shared cache |

A question may cross your mind: since multiple nodes are now reading and writing to shared cache, do we need to handle concurrency or apply locks? In most cases, no — if the password provider is idempotent (always returns a valid password that doesn’t expire prematurely), then all passwords are valid and no locking is required.

---

## 4th solution: shared and local cache together

![4th solution: shared and local cache together](https://substackcdn.com/image/fetch/$s_!o2M5!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fd901cb49-2ac4-4ccf-9b42-eaa2ba9ac749_1766x780.png)

We can combine the 2nd and 3rd solutions. Each node first checks its local cache; if it misses, it checks the shared cache; if that also misses, it calls the provider.

This reduces load on the shared cache, but introduces a new issue: cache invalidation. Now we must also store the expiration time alongside the password, so that nodes know how long they can safely reuse it.

This approach also allows fallback: if the shared cache fails, nodes can still go directly to the provider.

| Pros                                                                                                                         | Cons                                            |
| ---------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| + efficient use of caches<br> + fewer calls to provider<br> + less load on shared cache<br> + fallback if shared cache fails | - complex cache eviction and invalidation logic |

---

## 5th solution: rely on permission denied responses

![5th solution: rely on permission denied responses](https://substackcdn.com/image/fetch/$s_!sTCt!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F1632295e-5dc4-494b-890d-36fa4ce34f27_1184x694.png)

Another alternative is to skip TTL handling entirely in local cache. Instead, we keep using the cached password until the resource itself denies access (e.g., HTTP 401 Unauthorized). When that happens, the node fetches a new password (first from shared cache, and if not available, from the provider).

| Pros                                                                                                                 | Cons                                                                                                            |
| -------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| + no cache invalidation complexity<br> + fewer provider calls thanks to caching<br> + fallback if shared cache fails | - complexity of exception handling<br> - initial request fails before refresh<br> - potentially slower recovery |

---

## 6th solution: auto-refresh

So far, all solutions refresh the password on demand. The drawback is that the first request after expiration may be delayed.

![6th solution: auto-refresh](https://substackcdn.com/image/fetch/$s_!3-V9!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F825ef3fd-47d7-4581-b9f1-b8056445561a_1300x462.png)

To fix this, we can introduce an auto-refresh mechanism: a separate thread (or cron job) checks for expiring passwords and refreshes them proactively, both in local and shared cache.

| Pros                                                                                            | Cons                                                                                                                                                   |
| ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| + requests always have a valid password in cache<br> + fewer provider calls due to shared cache | - added complexity of scheduling and eviction <br> - multithreading overhead<br> - if the auto-refresh fails, nodes may be left with expired passwords |

---

## 7th solution: auto-refresher node

![7th solution: auto-refresher node](https://substackcdn.com/image/fetch/$s_!Iofq!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F44586483-06ad-4df4-b4c3-9f74493a8d28_1300x560.png)
What if instead of guaranteeing that the local cache is always valid, we guarantee that shared cache is always valid, so each time the local cache expires it only gets from the shared cache and it doesn't handle all the logic for updating the shared cache from password provider? In this case we can add another node which it's responsibility is to periodically check shared cache and if the password is about to expire, refresh it. the local cache eviction can be either on-demand cache eviction (solution 4) or on-demand permission denied handling (solution 5).

| Pros                                                                | Cons                                                                                                                                   |
| ------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| + fewer calls to the provider<br> + nodes rely only on shared cache | - single point of failure: if refresher node fails, no new passwords are fetched<br> - added complexity of monitoring refresher health |

However this solution has a big problem. the auto-refresher node is a single point of failure and if it fails no other node can refresh the password. so lets bring a fallback mechanism for handling auto-refresher's failure.

---

## 8th solution: auto-refresher node with fallback

![8th solution: auto-refresher node with fallback](https://substackcdn.com/image/fetch/$s_!cXZt!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8befbc10-f4b4-448e-a538-915ce92df856_1300x710.png)

To solve the single point of failure, we can give all nodes the ability to fetch from the provider if needed. The auto-refresher node still takes the main responsibility for updating shared cache, but if it fails, nodes can fall back to solutions 4 or 5.

![9th solution: auto-refresher node with fallback](https://substackcdn.com/image/fetch/$s_!gWz1!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fe163b24a-f0ec-4725-8d0a-ef9fbca62dc6_1360x728.png)

| Pros                                                                                                        | Cons                                                                                                |
| ----------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| + efficient and resilient<br> + refresher node reduces provider load<br> + fallback if refresher node fails | - added complexity of monitoring refresher health<br> - added complexity of scheduling and eviction |

---

## Conclusion

Designing a system around short-lived credentials is all about balancing simplicity, performance, and resilience.

- If usage is rare → simple API calls may be enough.
- If usage is frequent but simple → local cache works.
- If you need efficiency across multiple nodes → shared cache (with or without local cache) is a good fit.
- If you want robustness → auto-refresh mechanisms (with fallbacks) provide better guarantees at the cost of added complexity.

In practice, there’s no single “correct” approach. It depends on your system’s requirements, scale, and tolerance for failure.

For me, what started as a “copy-paste task” turned into a mini system design exploration. And that’s the fun part of engineering: sometimes the simplest problems open the door to deeper thinking about reliability, scalability, and tradeoffs.
