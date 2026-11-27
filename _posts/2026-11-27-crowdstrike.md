---
title: I decompiled 3.3 million lines of CrowdStrike
date: 2026-11-26
draft: false
---


<p align="center"><img src="https://upload.wikimedia.org/wikipedia/commons/4/4f/CrowdStrike_logo.svg?utm_source=de.wikipedia.org&amp;utm_campaign=index&amp;utm_content=original" alt="CrowdStrike" width="320"></p>

As many of you might know, I work on [VulcanSIEM](https://vulcansiem.com) which is my custom ultra high-performance security analytics engine and SIEM. I started working on it around 2 years ago and have now reached a point where it is getting serious. Recent benchmarks prove VulcanSIEM to ingest and query data, orders of magnitude faster than competitors.

Naturally I did some basic competitor analysis and found [CrowdStrike](https://www.crowdstrike.com/) to be my toughest competitor. Their metrics already claim very high numbers.

Today I took some time to do some decompilation and reverse engineering of CrowdStrike LogScale for fun and so I can understand their core internals and ingestion logic at a much better niveau.

This was done via the official Docker image (`humio/humio-single-node-demo`, documented [here](https://library.humio.com/deployment/installation-containers-humio.html)). I pulled the rootfs with [skopeo](https://github.com/containers/skopeo), then let [jadx](https://github.com/skylot/jadx) loose on the JVM classes. The image called itself LogScale **1.258.4**.

Luckily for me (bad for CrowdStrike), they shipped JVM classes. Decompilation is easy in theory: JVM bytecode gets you close to the original source. In practice, Scala makes jadx work for it, but you still end up with something you can read.

I expected a couple of jars. I got **736**. 410 of them contain `com/humio` classes. Decompiling all of them gave me:

```
29,527 Java files
3,339,116 lines of code
129 com.humio packages
```

![decompiled CrowdStrike LogScale line count](https://pbs.twimg.com/media/HX-lyztX0AAjTwn?format=jpg&name=4096x4096)

That's 3.3 million lines of decompiled CrowdStrike. Here is what stood out.

**Kafka is the spine.** The biggest surprise: Kafka is not an optional add-on. Even a single node runs it, and the cluster's control plane is three topics: `global-events` (every entity change, plus the global snapshot), `chatter` (running queries and stats), and `humio-ingest` (the ingest queue). Nodes elect a leader, heartbeat through global state, and treat a Kafka epoch change as fatal rather than risk split-brain.

That's a bold design. Kafka is not just a queue here; it is the source of truth and the coordination layer.

**Ingestion is a pipeline with a lot of seatbelts.** The protocol surface is huge: native JSON, Elastic bulk, Splunk HEC, OTLP, raw, Logplex, plus TCP/UDP listeners for syslog, GELF and NetFlow. Beats and Fluent Bit come in through the Elastic path.

The flow is roughly: token → dataspace → parser → tags → Kafka. AutoSharding kicks in when a dataspace gets too many datasources. A digest leader per partition consumes the queue and writes segments. Offsets are not committed to Kafka; they are rebuilt from segment metadata, so a crash replays the last window. At-least-once, by design.

What impressed me is how much code exists just to stop one bad parser or one slow tenant from taking down ingestion. Timeouts, limiter jobs, drop ratios, backpressure. Very operational.

**Storage is immutable segments.** Mini-segments are LZ4 blocks with an index block and a bloom-filter sidecar. Normal segments are recompressed with Zstd and a trained dictionary. Merges happen in the background; uploads go to S3/Azure/GCS; replication is weighted rendezvous hashing. The upload rule is beautifully simple: only the first current owner uploads. No distributed lock.

Retention keeps segments that still have a Kafka offset so replay stays possible. Deletes are query-driven segment rewrites.

**Queries are coordinated per query.** A query picks a coordinator, then a state machine (live, historical time slicer, table, multi-pass, federated). Historical queries split into up to five cache-aligned slices. Each segment is assigned to exactly one host, so replicas are for failover, not parallel reads. The coordinator polls and merges partial reducer states. Bloom filters let it skip whole blocks.

It is pragmatic. Not academically pretty, but very much built for real workloads.

**Multi-tenancy is everywhere.** Organization → View/repository → Dataspace. Every entity is org-tagged. Permissions have five tiers, roles, groups and query prefixes. The interesting code is at the boundaries: global requesters, internal requesters, and org impersonation. That is where a SIEM either holds up or doesn't.

## The tax: JVM + Scala + Pekko + Kafka

Everything above buys durability and coordination. It also costs.

**Kafka is on the critical path, even for one node.** Every event goes HTTP → parser → Kafka produce → Kafka consume → segment. That is an extra network hop, an extra serialization and an extra disk write compared with writing the segment directly. The producer uses `acks=all` and `linger.ms=3`; production adds RF 3 and cross-AZ replication. The demo sets RF to 1, but then you have paid for Kafka without the durability.

**The control plane is one partition.** All entity writes serialize through `global-events`, and every node replays it at boot. It is simple and correct, but it is a single-lane road for cluster metadata. Token churn, config updates and segment metadata all queue behind each other. Retention is capped at 20 GB, and the transactions are JSON.

**Query coordination is poll-based.** Coordinators tick their state machines on the order of 700 ms to 1.5 s (first tick 1.5 s, then 700 ms for live queries). Historical queries go through time slices and a state cache. For an interactive SIEM, that polling cadence is latency you feel.

**The JVM and Scala tax is real.** The demo runs two JVMs (LogScale and Kafka), 736 jars, a bundled JDK, and a Scala codebase full of companions, `Option`/`Seq` allocations, boxed primitives and anonymous classes. Heap is sized to 75% of RAM and direct memory gets a quarter GB per core. That is a lot of surface for GC pauses, JIT warmup and classloading before you have indexed a single event.

**Pekko/Akka is powerful and opaque.** Actors, dispatchers, mailboxes and FSMs make the distributed coordination possible, but backpressure is explicit queue limits rather than a property of the model. Debugging a query that is stuck across a coordinator FSM, polling actors and segment fetchers is not fun.

**The operational weight is the real cost.** Kafka is mandatory; a Kafka epoch change is fatal by design; ingest offsets are rebuilt from segments; partition ownership is a compare-and-swap; consumer groups are avoided with manual assignment. All of it is reasonable. All of it is machinery you have to run to store logs.

Kafka buys replay, durability and a clean control plane. If you do not need a shared log, you are paying for one. For a single node or a latency-sensitive local pipeline, this architecture is heavier than it needs to be. I did not benchmark it; these are costs visible in the code and the defaults.

Two big pieces are missing from the Java dump.

**The collector (and how I got it).** The LogScale Collector is a separate closed-source Go binary (Go 1.27.3 in the 1.12.2 build). The server only implements the fleet-management control plane: install tokens, enroll/poll/pull/push/refresh, metrics, update checks, and a YAML config schema with 12 source types (file, journald, syslog, cmd, wineventlog, unifiedlog, Postgres, MySQL, Oracle, MSSQL, ...) and 4 sink types (LogScale, HEC, loopback).

Getting the agent was easier than expected. The docs point at a public Helm repo:

```
https://registry.crowdstrike.com/log-collector-us1-prod/index.yaml
```

Chart 1.6.2 pins the image in `values.yaml`:

```
registry.crowdstrike.com/log-collector/us-1/release/logscale-collector:1.12.2
```

That registry allows anonymous pulls. One command gets the container:

```bash
skopeo copy --override-os linux --override-arch arm64 \
  docker://registry.crowdstrike.com/log-collector/us-1/release/logscale-collector:1.12.2 \
  dir:logscale-collector
```

The layer contains `/usr/bin/humio-log-collector`: a 29 MB arm64 / 32 MB amd64 stripped Go binary. The image is Ubuntu-based; the entrypoint is `tini` plus `run-humio-log-collector.sh`, and the config lives at `/etc/humio-log-collector/config.yaml`.

The other routes are gated. The server's update manifest (`https://lc-update.humio.com/manifest.json`) wants the LogScale license as a bearer token and returns 401 anonymously. The `.deb`/`.rpm`/`.msi`/`.pkg` downloads are proxied by a running LogScale instance with an API token. The public OCI image needs neither.

The agent is still closed source: this gets you the binary, not the Go source.

The **frontend** ships in three pure-asset jars that contain no `com/humio` bytecode, which is why the class decompile missed them: `ui.jar` (compiled Elm), `webcomponents.jar` (seven esbuild components), `api-explorer.jar` (Monaco). The web components have source maps, but no `sourcesContent`. The Elm bundle has no source map at all. No free frontend source.

**Decompiling is not magic.** JVM bytecode gets you close to the original structure, but not byte-level accuracy. Scala companions, `$$anonfun` classes and erased generics are everywhere; some files are decompiler artifacts. Treat line numbers as approximate and verify behavior against the running binary.

And the things that are not shipped — the Go agent, the Elm UI, the TypeScript — are not recoverable from the jars. You can read the bundles; you cannot get the source back.

If I had to compress it into a few takeaways:

- Kafka-first is a real commitment. Simple durability and coordination; a mandatory latency and blast-radius component.
- Immutable segments + bloom filters + late merges fit append-heavy security data.
- The query design is pragmatic: one owner per segment, merge partial states, reuse cached slices.
- The hard part of a SIEM is not storing logs. It is multi-tenancy and the long tail of ingestion formats. LogScale has both.

## What's next

I want to spend more time on the segment format and query planner, and to analyze the collector binary. The frontend is a dead end without source maps, though the module graph is interesting. I will also at some point need to benchmark logscale against my own high-variance dataset to compare its performance metrics against VulcanSIEM. 

That's it. 3.3 million lines of Scala later, I still think LogScale is one of the most impressive log platforms out there.

P.S: @crowdstrike if ur reading this, please dont sue me. I love you guys <3
