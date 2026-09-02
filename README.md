# PostgreSQL_Vacuum

# PostgreSQL Learning Notes

Personal notes written while studying PostgreSQL, alongside deliberate expansion into networking, Linux, and DevOps. This repo is a running log of topics I dig into, written in my own words and refined through practice.

## Contents

- [`postgresql-vacuum.md`](./postgresql-vacuum.md) — What `VACUUM` does, the three types (Plain, `VACUUM FULL`, `AUTOVACUUM`), why it matters, how to monitor it, and common issues with fixes.

## How these notes are built

Each topic follows the same approach:

1. Write out my own understanding first.
2. Fill gaps and correct misconceptions through research and practice.
3. Test the concepts hands-on against a real PostgreSQL instance (Docker or local install).
4. Document it here in a clear, self-contained file.

## Practicing VACUUM hands-on

To reproduce the dead-tuple scenario described in `postgresql-vacuum.md`:

```bash
docker run --name pg-practice -e POSTGRES_PASSWORD=test123 -p 5432:5432 -d postgres

```
<img width="1292" height="469" alt="image" src="https://github.com/user-attachments/assets/9459337b-a5d5-4e8d-8b4b-237e6d325989" />

```bash
docker exec -it pg-practice psql -U postgres
```
<img width="985" height="309" alt="image" src="https://github.com/user-attachments/assets/c70978ca-ac19-4a93-a754-27f8abda3b3c" />

Then inside `psql`:

```sql
CREATE TABLE test_vacuum (id SERIAL PRIMARY KEY, data TEXT);
INSERT INTO test_vacuum (data) SELECT 'row ' || generate_series(1,10000);
UPDATE test_vacuum SET data = 'updated' WHERE id < 5000;


CREATE TABLE makes a simple table with an auto-incrementing id and a text column
generate_series(1,10000) generates 10,000 numbers, so the INSERT creates 10,000 rows in one shot
The UPDATE rewrites 5,000 rows, which under MVCC creates 5,000 dead tuples instead of editing in place

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

SELECT relname, n_dead_tup, n_live_tup FROM pg_stat_user_tables WHERE relname = 'test_vacuum';
VACUUM test_vacuum;
SELECT relname, n_dead_tup, n_live_tup FROM pg_stat_user_tables WHERE relname = 'test_vacuum';
```

Compare `n_dead_tup` before and after the `VACUUM` to see the cleanup in action.

<img width="1051" height="290" alt="image" src="https://github.com/user-attachments/assets/be1caee3-541f-42c8-aaaa-2c4805d4cdb8" />


