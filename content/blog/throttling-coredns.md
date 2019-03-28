+++
title = "Throttling CoreDNS"
description = "A guide to limit impact on memory consumotion due to burst of queries"
tags = ["Kubernetes", "Memory", "DNS", "Throttle", "Documentation"]
date = "2019-03-26T20:23:43-00:00"
author = "sandeep"
+++

## CoreDNS Memory Consumption

There have been [some incidents reported](https://github.com/coredns/coredns/issues/2593)
regarding CoreDNS taking up too much memory when there is a high rate of DNS queries.

In the context of Kubernetes, this results in CoreDNS crashing as it gets `OOMkilled`.
To understand this better, we need to understand the resource requests and limits of a pod and container.
A deep dive on how Kubernetes manages its resources for containers is well documented [here](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/).
To give a high level summary, when CoreDNS exceeds its resource limits and requests the following may occur:
 - If the CoreDNS pod exceeds its memory limit, it might be terminated. If it is restartable, the kubelet will restart it.
 - If CoreDNS exceeds its memory request, it is likely that its Pod will be evicted whenever the node runs out of memory.
 
Below is the CoreDNS limits set in Kubernetes:

~~~ yaml
resources:
  limits:
    memory: 170Mi
  requests:
    cpu: 100m
    memory: 70Mi
~~~

This means that if CoreDNS uses greater than 170Mi due to a burst of incoming DNS queries, CoreDNS will be `OOMKilled` due to it taking up more
memory than the limit set. This may be problematic if the CoreDNS pods are crashing since it may lead to an outage.

To prevent the memory spike from happening, the `throttle` plugin has been proposed.
The proposed `throttle` plugin helps to limit the maximum number of simultaneous inflight queries which helps in preventing CoreDNS
being `OOMkilled` in case of a burst of incoming queries. 

## Testing CoreDNS memory usage

To test the `throttle` plugin and show the difference in memory consumption of CoreDNS with and without the `throttle` plugin,
I set up a Kubernetes cluster with one replica of CoreDNS running in the Master node. I set the memory limit of the CoreDNS container
to 1700Mi to help capture the memory consumption in case CoreDNS crashes due to it being `OOMkilled`.

To provide a burst of incoming queries, I used a [a simple tool for stressing CoreDNS](https://github.com/mikkeloscar/go-dnsperf).
The tool spins up DNS client replicas which queries a set of DNS names to resolve. One of the reasons I used this tool is because it appends search domains from /etc/resolv.conf. 
With the Kubernetes "search-path/ndots:5" situation, this multiplies the actual number of queries being sent by a large amount (X4-5).

For this test, I had set up each DNS replica queries at a rate of 100 qps(queries per second), with each querying for both A and AAAA records.
Hence, a single replica would request a total of (100 qps x2 records = ) 200 qps.
To provide an high incoming query burst, I set up the test to spin up 150 DNS client replicas
which in total resulted in a total incoming request of (200 x 150 = ) 30k qps to CoreDNS.

The Configuration I used to test is the default as of Kubernetes v1.14 with the exception that I used the [`erratic` plugin](https://coredns.io/plugins/erratic/)
instead of the `forward` plugin to overcome any Network shortcomings.

The CoreDNS Configuration I used is as follows: 
```corefile
 .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          upstream
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        throttle 100
        erratic {
          drop 0
        }
        cache 30
        loop
        reload 2s
        loadbalance
    }
```

### Memory consumption of CoreDNS without the `throttle` plugin

Without the `throttle` plugin, CoreDNS was crashing repeatedly due to it being `OOMKilled`. The maximum consumed memory of CoreDNS 
that was captured before the crash is 1.655 GiB.

~~~logs
      State:          Running
      Started:      Tue, 26 Mar 2019 12:20:48 -0400
      Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      Tue, 26 Mar 2019 12:20:07 -0400
      Finished:     Tue, 26 Mar 2019 12:20:26 -0400
      Ready:          True
      Restart Count:  2
~~~ 

It is worth noting that the amount of memory consumed by CoreDNS is proportional to the number of `goroutines`. 
The metrics captured for the memory consumption and the number of `goroutines`: 

![no-throttle-memory](https://user-images.githubusercontent.com/30265084/55086273-7923c700-507e-11e9-8735-ff18d407c40c.png)


### Memory consumption of CoreDNS using the `throttle` plugin

I tested the same scenario with the `throttle` plugin enabled. 
Example usage of `throttle` in the CoreDNS Corefile configuration:

~~~ corefile
. {
    throttle <max-inflight-queries>
}
~~~

I had tested for three different values for the `max-inflight-queries`, where it was set to 100, 500 and 1000
to see the memory consumption in each of the cases.

In this test, the base memory used by CoreDNS when there are no incoming queries is 20MiB. 
The table below shows the throttled value of the `max-inflight-queries` to the corresponding difference of memory from the
base memory CoreDNS was consuming with no incoming queries and memory when there were queries from the 150 DNS client replicas.

| Throttle             | Memory Usage (Δ) |  
|:--------------------:|:----------------:|
|  100                 |   12MiB          |
|  500                 |   20MiB          |
| 1000                 |   25MiB          |

As seen above, the memory used by CoreDNS using the `throttle` plugin is way less and hence it does not result in OOMkills.

The `throttle` plugin also includes useful metrics to show the total `incoming`, `served` and `dropped` requests.
Below is a snippet of the metrics of CoreDNS memory and metrics from the `throttle` plugin.


#### Throttle `max-inflight-queries` at 100

![throttle-100](https://user-images.githubusercontent.com/30265084/55090003-c145e800-5084-11e9-8ab8-1d1f95339d18.png)

#### Throttle `max-inflight-queries` at 500

![throttle-500](https://user-images.githubusercontent.com/30265084/55090033-cacf5000-5084-11e9-890d-748dffd72966.png)

#### Throttle `max-inflight-queries` at 1000 

![throttle-1k](https://user-images.githubusercontent.com/30265084/55090063-d753a880-5084-11e9-9bf4-9129f839f49f.png)


## Conclusion

When there is a sudden burst of incoming queries, CoreDNS spins up a large amount of concurrent `goroutines` to process all the incoming queries.
This results is a massive increase in memory consumed by CoreDNS.
As a result, from a Kubernetes perspective, this can result in CoreDNS to be `OOMKilled` due to crossing the memory limits set in the deployment.

To overcome this issue, as I have demonstrated above, the `throttle` plugin can be used to limit the impact of the burst of incoming queries to CoreDNS
by throttling the number of simultaneous inflight queries. This results in significantly less memory consumed by CoreDNS and ensure that CoreDNS does not crash
and continues to perform as expected.
