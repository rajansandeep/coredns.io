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
a Kubernetes cluster was set up with one replica of CoreDNS running in the Master node. The memory limit of the CoreDNS container was set
to 1700Mi to help capture the memory consumption in case CoreDNS crashes due to it being `OOMkilled`.

To provide a burst of incoming queries, [a simple tool for stressing CoreDNS](https://github.com/mikkeloscar/go-dnsperf) was used.
The tool spins up DNS client replicas which queries a set of DNS names to resolve.

Each DNS replica queries at a rate of 100 qps(queries per second), with each querying for both A and AAAA records.
Hence, a single replica would request a total of (100 qps x2 records = ) 200 qps.
To provide an high incoming query burst, the test had 150 DNS client replicas spun up
which in total would result in a total incoming request of (200 x 150 = ) 30k qps to CoreDNS.

The Configuration used to test is the default as of Kubernetes v1.14 with the exception that the [`erratic` plugin](https://coredns.io/plugins/erratic/) was used 
instead of the `forward` plugin to overcome any Network shortcomings.

The CoreDNS Configuration used is as follows: 
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
The metrics captured for the memory comsumption and the number of `goroutines`: 

![no-throttle-memory](https://user-images.githubusercontent.com/30265084/55086273-7923c700-507e-11e9-8735-ff18d407c40c.png)


### Memory consumption of CoreDNS using the `throttle` plugin

The same scenario was tested with the `throttle` plugin enabled. 
Example usage of `throttle` in the CoreDNS Corefile configuration:

~~~ corefile
. {
    throttle <max-inflight-queries>
}
~~~

The test was done for three different values for the `max-inflight-queries`, where it was set to 100, 500 and 1000
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
