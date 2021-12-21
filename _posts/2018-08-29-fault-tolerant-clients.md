---
layout: post
title: Best practices for fault-tolerant web clients
twitter_link: https://twitter.com/clare_liguori/status/1034829325306978304
---

[This Twitter thread](https://twitter.com/colmmacc/status/1034502453385822208)
by Colm MacCarthaigh about shuffle sharding reminded me of how important it is 
that web clients participate in fault tolerance, and how frustrated I get when
a client library *doesn't* do this by default in my application. Let's talk 
about some best practices!

There are three important behaviors for fault-tolerant clients:

1. Retry
2. Timeout
3. Backoff

Good client libraries have knobs for each one, so you can tune for your application's needs.

**Retries** are a must-have! They'll most likely get your request directed to a 
healthy node if one is having issues, and will help you weather any transient
network issues. Great clients have logic that retries only *some* kinds of 
failures, like connection errors and HTTP 500s, and doesn't retry on errors 
that are likely non-transient like HTTP 400s.

**Timeouts** are important so that 1) you get the opportunity to retry! and 2)
slow requests don't hog all your available threads waiting on a response. There
are usually different kinds of timeouts you can set in good client libraries, 
with sane defaults: connection timeout, socket timeout, read timeout, write 
timeout, individual request timeout, overall timeout including retries, etc.

My favorite timeout setting (read: the one that has bitten me many, many times)
is probably socket timeouts i.e. the amount of time a request's connection can 
sit there idle before the request gives up and fails. On many systems, a dead 
socket won't timeout by default for ~2 hours (yes, that's HOURS).  As in, get a
stuck request, go off to dinner, come back, and your request will still be 
sitting there idle hogging a thread!

You can ratchet socket timeouts down a bit at the system level by configuring
keep-alives (see [this Redshift guidance](https://docs.aws.amazon.com/redshift/latest/mgmt/connecting-firewall-guidance.html#connecting-firewall-guidance.change-tcpip-settings)),
but in general you'll want to configure timeouts at the application and client
level based on your needs. Many systems and client libraries will use a socket 
read timeout of *infinity* by default. As in, FOREVERRRR ... at least, until
the next application restart. I am not waiting around forever, no thanks, my 
application has better things to do!

Moving on: **Backoffs!** Backoffs can be as simple as a `sleep(1)` in your retry
loop, but exponential backoffs will give you the most bang for your buck. 
Requests during short issues will get retried quickly and then succeed, and
longer issues won't require a ton of retries.

Spending some time to **tune your web client settings** has a high payoff of
immediately better resiliency in your application. The AWS SDKs are great 
examples of fault-tolerant client libraries, with configurable retries, 
exponential backoff, and timeouts. The defaults are a good start, but remember
to monitor and tune for best performance. For example,
[this post](https://aws.amazon.com/blogs/developer/tuning-the-aws-sdk-for-java-to-improve-resiliency/)
discusses tuning the AWS SDK for Java to improve resiliency. Many other common
client libraries now have fault-tolerant options by default too. Apache 
HttpClient 4.x for example has `DefaultHttpRequestRetryHandler` and 
`DefaultBackoffStrategy`, but set the timeouts explicitly (otherwise, you could
be waiting for hours to timeout on a dead socket ... HOURS).

Two very common clients that repeatedly trip me up with respect to fault 
tolerance are **ssh and curl!** Both require non-default configuration to be 
used in a reliable application. Think about your build scripts, automated 
operations tools, and monitoring canaries that you want to withstand failures,
and likely many of them are using some combination of ssh and curl.

For resilient tooling, add this to your SSH config:
```
  Host *
    ConnectTimeout 10
    ConnectionAttempts 10
```
And use these curl options:
```
curl --retry 3 --connect-timeout 10 --max-timeout 20 --retry-max-time 30
```
(Tune the exact numbers for your application's needs)

In the containers world, I'm excited about using **sidecar proxies** like Envoy
to help applications set sane retries and timeouts, regardless of the 
application's client libraries. Notice in
[this example using an Envoy proxy](https://blog.christianposta.com/microservices/02-microservices-patterns-with-envoy-proxy-part-ii-timeouts-and-retries/): 
no special curl flags are required to enable retries!
