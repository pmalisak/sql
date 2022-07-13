# PostgreSQL indexes

```postgresql
CREATE TABLE foo_v2 (
    id UUID PRIMARY KEY,
    name VARCHAR(32) NOT NULL,
    status VARCHAR(15) NOT NULL,
    created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL
);

CREATE INDEX foo_v2_status ON foo_v2 (status);
CREATE INDEX foo_v2_created_at ON foo_v2 (created_at);

INSERT INTO foo_v2(id, name, status, created_at)
SELECT
    gen_random_uuid(),
    LEFT(md5(random()::text), 20),
    (array['waiting', 'processing', 'pending', 'rejected', 'failed', 'completed'])[floor(random() * 6 + 1)],
    (SELECT NOW() + (random() * (interval '90 days')) + '30 days')
FROM generate_series(1, 1000000);

INSERT INTO foo_v2(id, name, status, created_at)
SELECT
    gen_random_uuid(),
    LEFT(md5(random()::text), 20),
    'test',
    (SELECT NOW() + (random() * (interval '90 days')) + '30 days')
FROM generate_series(1, 100);
```

There is only 100 records with status 'test'.

```postgresql
explain analyze select id from foo_v2 where status = 'test' limit 20;
```

```
Index Scan using foo_v2_status on foo_v2  (cost=0.43..8.35 rows=1 width=16) (actual time=0.324..0.333 rows=10 loops=1)
  Index Cond: ((status)::text = 'test'::text)
Planning Time: 1.090 ms
Execution Time: 0.534 ms
```

There is about 1.5 million records with status 'completed'.

```postgresql
explain analyze select id from foo_v2 where status = 'completed' limit 20;
```

```
Limit  (cost=0.00..2.68 rows=20 width=16) (actual time=0.239..0.260 rows=20 loops=1)
  ->  Seq Scan on foo_v2  (cost=0.00..228679.12 rows=1704668 width=16) (actual time=0.232..0.242 rows=20 loops=1)
        Filter: ((status)::text = 'completed'::text)
        Rows Removed by Filter: 50
Planning Time: 0.917 ms
Execution Time: 0.399 ms

```
