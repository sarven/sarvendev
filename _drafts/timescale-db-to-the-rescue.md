---
title: TimescaleDB to the Rescue - Speeding Up Statistics
date: 2025-03-28
categories: [Programming, Database]
---

Some time ago, I was working on improving the performance of slow statistics. The problem was that our database contained billions of rows, making data retrieval slow, even for the last seven days. 
From a product perspective, we needed to display data for at least 30 days and in real-time. All the data was stored in MySQL without partitioning, 
so we had to find a better solution. Simply using a cache was not an option, as real-time data was required.

Let's analyze it on contrived example but similar to the original one. 

## MySQL solution
Let's say that we have a table with the following structure:

```sql
CREATE TABLE agent_stats (
    id BIGINT AUTO_INCREMENT NOT NULL,
    agent_id INT NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id)
);

CREATE INDEX idx_agent_stats_created_at_agent_id_event_type ON agent_stats (created_at, agent_id, event_type);
```

We're collecting there statistics of our AI agents. Now, we have two types of events:
- triggered
- response_generated

To test the performance of the database, we need to generate a lot of data. In MySQL there is no fast way to 
generate random data, so I used some script which generated 24 234 964 records. It was quite slow. 

And now, we want to get the number of triggered events for each agent in the last 30 days.

```sql
SELECT agent_id, event_type, COUNT(*) as count
FROM agent_stats
WHERE created_at > '2025-02-28 00:00:00'
GROUP BY agent_id, event_type
```

It's very slow, and takes 11 seconds.

## What is TimescaleDB?

TimescaleDB is an open-source time-series database built on PostgreSQL. It is designed to handle large volumes of 
time-series data efficiently. Basically it is a PostgreSQL extension that adds time-series capabilities to the 
database. It's optimized for insertions of time-series data, and it provides features like automatic partitioning, 
retention policies, and continuous aggregates. 

## TimescaleDB solution

So let's try to use TimescaleDB to speed up our statistics. Creating a similar table in TimescaleDB:

```sql
CREATE TABLE agent_stats(
   created_at TIMESTAMPTZ NOT NULL,
   agent_id BIGINT NOT NULL CHECK (agent_id > 0),
   event_type VARCHAR(255) NOT NULL
);

SELECT create_hypertable('agent_stats', 'created_at');

CREATE INDEX ON agent_stats (created_at DESC, agent_id, event_type);
```

The `create_hypertable` function creates a hypertable, which is a TimescaleDB abstraction for a standard PostgreSQL 
table, but with automatic partitioning based on time.

Generating data in TimescaleDB is much convenient and a lot faster than in MySQL.

```sql
INSERT INTO agent_stats (created_at, agent_id, event_type)
SELECT
    time,
    random() * 9 + 1, /* 1 - 10 */
    'triggered'
FROM
    generate_series(
        '2024-01-01 00:00:00',
        '2025-03-28 16:00:00',
        INTERVAL '1 second'
    ) AS time;

INSERT INTO agent_stats (created_at, agent_id, event_type)
SELECT
    time,
    random() * 9 + 1, /* 1 - 10 */
    'response_generated'
FROM
    generate_series(
        '2024-01-01 00:00:00',
        '2025-03-28 16:00:00',
        INTERVAL '1 second'
    ) AS time;
```

In this way we can generate 2 records per second in the range of around 1 year and 3 months. I repeated this a few 
times, and in a few minutes total of 184 675 516 records were generated.

Now, let's get the same data as before:

```sql
SELECT agent_id, event_type, COUNT(*) as count
FROM agent_stats
WHERE created_at > '2025-02-28 00:00:00'
GROUP BY agent_id, event_type
```

Keep in mind that we have much more data now, so the query it's also slow. Now it takes 9 seconds, but of course on 
the same data as in MySQL it would be much faster, because of the partitioning. Ok, so now we need to speed it up.

## TimescaleDB continuous aggregates
Continuous aggregates are a powerful feature of TimescaleDB that allows you to pre-compute and store the results of 
query aggregations over time. It's based on the concept of materialized views in PostgreSQL, but it can return data 
real-time. 

Let's create a new continuous aggregate for agent_stats table:
```sql
CREATE MATERIALIZED VIEW hourly_agent_stats WITH (timescaledb.continuous)
AS
SELECT
    time_bucket('1 hour', created_at) as hour,
    agent_id,
    event_type,
    COUNT(1) AS occurrences
FROM agent_stats
GROUP BY
    hour,
    agent_id,
    event_type
WITH NO DATA
;
```

We also need to create a refresh policy to keep the materialized view up to date.
```sql
SELECT add_continuous_aggregate_policy('hourly_agent_stats',
    start_offset => INTERVAL '1 day',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);
```

Now we need to populate the materialized view with the data we already have in the table.
```sql
CALL refresh_continuous_aggregate('hourly_agent_stats', '2024-01-01', '2025-03-28');
```

It takes some time, and probably on the production  it would be better to do it incrementally e.g. per day.

How it looks in the database:

| hour                | agent_id | event_type        | occurrences |
|---------------------|----------|-------------------|-------------|
| 2024-01-01 00:00:00 | 1        | triggered         | 1000        |
| 2024-01-01 00:00:00 | 1        | response_generated| 2000        |
| 2024-01-01 00:00:00 | 2        | triggered         | 1500        |
| 2024-01-01 00:00:00 | 2        | response_generated| 2500        |
| 2024-01-01 01:00:00 | 1        | triggered         | 1200        |
| 2024-01-01 01:00:00 | 1        | response_generated| 2200        |
| 2024-01-01 01:00:00 | 2        | triggered         | 1700        |
| 2024-01-01 01:00:00 | 2        | response_generated| 2700        |

Records in the view are aggregated by hour, so we have one record per hour per agent_id and event_type.

Now we can modify the query to take advantage of the newly created continuous aggregate:
```sql
SELECT agent_id, event_type, SUM(occurrences) as occurrences
FROM hourly_agent_stats
WHERE hour > '2025-02-28 00:00:00'
GROUP BY agent_id, event_type
```

This query is much faster, and it takes only 0.02 seconds.

It's possible to get real-time data from the continuous aggregate, as historical data is fetched from the view, while the last hour (since we use a 1-hour bucket) is fetched from the source table.

## TimescaleDB retention
TimescaleDB also provides a retention policy feature that allows you to automatically drop old data. Let's say that 
we need only 6 months of detailed data, and 1 year of aggregated data. We can set up the following retention policies:
```sql
SELECT add_retention_policy('agent_stats', INTERVAL '6 MONTH');
```
This will drop all data older than 6 months from the source table.

```sql
SELECT add_retention_policy('hourly_agent_stats', INTERVAL '1 YEAR');
```
This will drop all data older than 1 year from the continuous aggregate.

## Summary
We could achieve some performance improvement by using partitioning in MySQL, but it would be a slight improvement 
and additional work. Adding TimescaleDB increases the complexity of the whole system, as it's a new technology that 
needs to be maintained, but it's a great choice for time-series data and great choice from the point of view of 
application engineers, as it provides a lot of useful features. However, if you're using PostgreSQL now, using 
TimescaleDB will be easier to implement, as you don't need to learn a new technology, it's just an extension.
