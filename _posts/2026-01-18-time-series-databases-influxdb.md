---
layout: post
title: "Time Series Databases: Why InfluxDB Changed How I Think About Data"
date: 2026-01-18
tags: [databases, influxdb, time-series, backend]
---

So I've been working with time series data at work lately, and honestly it took me a while to understand why we couldn't just use PostgreSQL for everything. Spoiler: we tried. It didn't go well.

## The Problem

We had sensor data coming in - thousands of data points per second. Temperature readings, pressure values, that kind of stuff. Initially we just dumped everything into a Postgres table with a timestamp column. Worked fine for a week. Then queries started taking 30+ seconds. Not great when you need real-time dashboards.

## Enter Time Series Databases

Time series databases are built specifically for this use case - data that's indexed by time and mostly append-only. You rarely update old readings, you just keep adding new ones.

The big players are:
- **InfluxDB** - what we ended up using
- **TimescaleDB** - Postgres extension, good if you want SQL
- **QuestDB** - newer, crazy fast for certain queries
- **Prometheus** - more for metrics/monitoring

## Why InfluxDB?

Few things that sold me:

**1. The query language makes sense**

```
from(bucket: "sensors")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "temperature")
  |> aggregateWindow(every: 5m, fn: mean)
```

Once you get past the initial "what is this syntax", it's actually pretty intuitive. You're basically describing a data pipeline.

**2. Automatic downsampling**

Old data? Downsample it. Keep 1-second resolution for the last day, 1-minute for the last week, 1-hour for anything older. Your storage thanks you.

**3. Built-in retention policies**

Set it and forget it. Data older than X days gets deleted automatically.

## The Gotchas

Not everything is perfect though:

- **High cardinality kills performance.** Don't use user IDs as tags if you have millions of users. Learned this the hard way.
- **Flux (the query language) has a learning curve.** InfluxQL was simpler but they're pushing Flux now.
- **Clustering is enterprise-only.** For the open source version you're stuck with a single node.

## When to Actually Use This

Don't reach for a TSDB just because it's cool. Use it when:
- You have time-stamped data (duh)
- High write throughput (thousands of points/second)
- You need time-based aggregations (averages over windows, etc)
- Data has a natural expiration

For everything else, Postgres is probably fine.

## Quick Comparison

| | InfluxDB | TimescaleDB | QuestDB |
|---|---|---|---|
| Query Language | Flux/InfluxQL | SQL | SQL |
| Learning Curve | Medium | Low (if you know SQL) | Low |
| Write Speed | Fast | Good | Very Fast |
| Maturity | High | High | Medium |

## Wrapping Up

If you're dealing with IoT data, metrics, or anything that generates continuous time-stamped readings, give InfluxDB a shot. The 2.x version with Flux is pretty solid once you get used to it.

Just remember - measure twice, choose your tags once. Cardinality will bite you if you're not careful.

---

*Got questions about time series stuff? Hit me up on [LinkedIn](https://linkedin.com/in/kinshuk-dwivedi-b447581b2/).*
