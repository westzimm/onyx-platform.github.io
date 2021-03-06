---
layout: user_guide_page
title: Messaging
categories: [user-guide-page]
---

## Messaging

### Background

The messaging layer takes care of the direct peer to peer transfer of segment
batches, acks, segment completion and segment retries to the relevant virtual
peers. The messaging layer design document can be found at the
[Messaging section]({{ "/messaging.html" | prepend: page.dir | prepend: site.baseurl }}).

### Messaging Implementations

The Onyx messaging implementation is pluggable and alternative implementations
can be selected via the `:onyx.messaging/impl` [peer-config]({{ "/peer-config.html" | prepend: page.dir | prepend: site.baseurl }}).

#### Aeron Messaging

Owing to [Aeron's](https://github.com/real-logic/Aeron) high throughput and low
latency, Aeron is the default Onyx messaging implementation. There are a few
relevant considerations when using the Aeron implementation.

##### Subscription (Connection) Multiplexing

One issue when scaling Onyx to a many node cluster is that every virtual peer
may require a communications channel to any other virtual peer. As a result, a
naive implementation will require up to m<sup>2</sup> connections over the
cluster, where m is the number of virtual peers. By sharing Aeron subscribers
between virtual peers on a node, this can be reduced to n<sup>2</sup>
connections, where n is the number of nodes. This reduces the amount of
overhead required to maintain connections between peers, allowing the
cluster to scale better as the number of nodes to increase.

It is worth noting that Aeron subscribers (receivers) must also generally
perform deserialization.  Therefore, subscribers may become CPU bound by the
amount of deserializaton work that needs to be performed. In order to reduce
this effect, multiple subscribers can be instantiated per node.  This can be
tuned via `:onyx.messaging.aeron/subscriber-count` in
[peer-config]({{ "/peer-config.html" | prepend: page.dir | prepend: site.baseurl }}). As increasing
the number of subscribers may lead back to an undesirable growth in the number
of connections between nodes, each node will only choose one subscription to
communicate through. The choice of subscriber is calculated via a hash of the combined IPs of the
communicating nodes, in order to consistently spread the use of subscribers over the cluster.

Clusters which perform a large proportion of the time serializing should
consider increasing the subscriber count. As a general guide, `# cores = #
virtual peers + # subscribers`.

##### Connection Short Circuiting

When virtual peers are co-located on the same node, messaging will bypass the
use of Aeron and directly communicate the message without any use of the
network and without any serialization. Therefore, performance benchmarks
performed on a single node can be very misleading.

The [peer-config]({{ "/peer-config.html" | prepend: page.dir | prepend: site.baseurl }}) option, `:onyx.messaging/allow-short-circuit?`
is provided for the purposes of more realistic performance testing on a single node.

##### Port Use

The Aeron messaging implementation will use the port configured via
`:onyx.messaging/peer-port`. This *UDP port* must be unfirewalled.

##### Media Driver

Aeron requires a media driver to be used on each node. Onyx provides an
embedded media driver for local testing, however use of the embedded driver is
not recommended in production. The embedded driver can be configured via the
`:onyx.messaging.aeron/embedded-driver?` [peer-config]({{ "/peer-config.html" | prepend: page.dir | prepend: site.baseurl }}) option.

When using Aeron messaging in production, a media driver should be created in
another java process. You can do this via the following code snippet, or by using the [Aeron distribution](https://github.com/real-logic/Aeron#media-driver-packaging).

```clojure
(ns your-app.aeron-media-driver
  (:require [clojure.core.async :refer [chan <!!]])
  (:import [uk.co.real_logic.aeron Aeron$Context]
           [uk.co.real_logic.aeron.driver MediaDriver MediaDriver$Context ThreadingMode]))

(defn -main [& args]
  (let [ctx (doto (MediaDriver$Context.))
        media-driver (MediaDriver/launch ctx)]
    (println "Launched the Media Driver. Blocking forever...")
    (<!! (chan))))
```

##### Configuration Options

Aeron is independently configurable via Java properties (e.g.  `JAVA_OPTS="-Daeron.mtu.length=16384"`).
Configuration of these may cause different performance characteristics, and
certain options may need to be configured in order to communicate large segments between peers.

Documentation for these configuration options can be found in
[Aeron's documentation](https://github.com/real-logic/Aeron/wiki/Configuration-Options).
