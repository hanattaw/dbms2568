```sql
-- account table
DROP TABLE account CASCADE;
CREATE TABLE account(
    account_id serial PRIMARY KEY,
    name text NOT NULL,
    dob date
);
```

```sql
-- thread table
DROP TABLE thread CASCADE;
CREATE TABLE thread(
    thread_id serial PRIMARY KEY,
    account_id integer NOT NULL REFERENCES account(account_id),
    title text NOT NULL
);
```

```sql
-- post table
DROP TABLE post CASCADE;
CREATE TABLE post(
    post_id serial PRIMARY KEY,
    thread_id integer NOT NULL REFERENCES thread(thread_id),
    account_id integer NOT NULL REFERENCES account(account_id),
    created timestamp with time zone NOT NULL DEFAULT now(),
    visible boolean NOT NULL DEFAULT TRUE,
    comment text NOT NULL
);
```


```sql
-- word table create with word in linux file
DROP TABLE words;
CREATE TABLE words (word TEXT) ;
\copy words (word) FROM '/data/words';
```

```sql
-- create account data
INSERT INTO account (name, dob)
SELECT
    substring('AEIOU', (random()*4)::int + 1, 1) ||
    substring('ctdrdwftmkndnfnjnknsntnyprpsrdrgrkrmrnzslstwl', (random()*22*2 + 1)::int, 2) ||
    substring('aeiou', (random()*4 + 1)::int, 1) || 
    substring('ctdrdwftmkndnfnjnknsntnyprpsrdrgrkrmrnzslstwl', (random()*22*2 + 1)::int, 2) ||
    substring('aeiou', (random()*4 + 1):: int, 1),
    Now() + ('1 days':: interval * random() * 365)
FROM generate_series (1, 100)
;
```

```sql
-- create thread data 
INSERT INTO thread (account_id, title)
WITH random_titles AS (
    -- 1. สร้างชื่อ Title สุ่มเตรียมไว้ 1,000 ชุด (หรือเท่ากับจำนวนที่ต้องการ insert)
    -- วิธีนี้จะทำการสุ่มคำเพียงครั้งเดียวต่อหนึ่ง title
    SELECT 
        row_number() OVER () as id,
        initcap(sentence) as title
    FROM (
        SELECT (SELECT string_agg(word, ' ') FROM (SELECT word FROM words ORDER BY random() LIMIT 5) AS w) as sentence
        FROM generate_series(1, 1000)
    ) s
)
SELECT
    (RANDOM() * 99 + 1)::int,
    rt.title
FROM generate_series(1, 1000) AS s(n)
JOIN random_titles rt ON rt.id = s.n
;
```

```sql
-- create post data
INSERT INTO post (thread_id, account_id, created, visible, comment)
WITH random_comments AS (
    SELECT row_number() OVER () as id, sentence
    FROM (
        SELECT (SELECT string_agg(word, ' ') FROM (SELECT word FROM words ORDER BY random() LIMIT 20) AS w) as sentence
        FROM generate_series(1, 1000)
    ) s
),
source_data AS (
    -- สร้างโครงข้อมูล 100,000 แถว พร้อมสุ่ม ID สำหรับเลือก comment
    SELECT 
        (RANDOM() * 999 + 1)::int AS t_id,
        (RANDOM() * 99 + 1)::int AS a_id,
        NOW() - ('1 days'::interval * random() * 1000) AS c_date,
        (RANDOM() > 0.1) AS vis,
        floor(random() * 1000 + 1)::int AS comment_id
    FROM generate_series(1, 100000)
)
SELECT 
    sd.t_id, 
    sd.a_id, 
    sd.c_date, 
    sd.vis, 
    rc.sentence
FROM source_data sd
JOIN random_comments rc ON sd.comment_id = rc.id -- ใช้ JOIN เพื่อการันตีว่าข้อมูลต้องมีค่า
;
```

## Step 1: Baseline Analysis
```sql
SELECT pg_size_pretty(pg_relation_size('post')) AS initial_size;
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'post';

-- Output


```

## Step 2: Create Massive Bloat
```sql
-- Run this UPDATE 5 times to create 500,000 dead tuples
UPDATE post SET comment = comment || ' [bloat]';
UPDATE post SET comment = comment || ' [bloat]';
UPDATE post SET comment = comment || ' [bloat]';
UPDATE post SET comment = comment || ' [bloat]';
UPDATE post SET comment = comment || ' [bloat]';


SELECT pg_size_pretty(pg_relation_size('post')) AS initial_size;
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'post';


-- Output


```

## Step 3: Verify the Performance Hit
```sql
EXPLAIN ANALYZE SELECT count(*) FROM post;
-- Note the increased Execution Time due to scanning dead rows.

-- Output 



```

## Step 4: Run Standard VACUUM
```sql
VACUUM (VERBOSE, ANALYZE) post;
-- Check size: It won't shrink, but n_dead_tup will go to 0.

-- Output


```

```sql

SELECT pg_size_pretty(pg_relation_size('post')) AS initial_size;
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'post';


-- Output
```
## Step 5: Run VACUUM FULL
```sql
VACUUM FULL post;
-- Check final size: The file will finally shrink on disk.
SELECT pg_size_pretty(pg_relation_size('post')) AS initial_size;
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'post';
 

```


