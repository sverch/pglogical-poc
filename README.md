# pglogical-poc

> Replicate from PostgreSQL 11.5 to 11.10 using pglogical 2.2.2

- [Manually setting up built-in logical replication](MANUAL.md)
- [Docker Compose Network checks](NETWORK.md)
- [pglogical-docs](PGLOGICAL.md)

## Setup

In this PoC we logically replicate from a PostgreSQL 11.5 to a PostgreSQL 11.10. Both instances running in Docker containers and communicating with each other. Both have pglogical 2.2.2 installed.

```bash
# start containers
docker-compose up -d

# in case we need to rebuilt the images use
docker-compose up -d --build

# running services
docker-compose ps

# stop containers
docker-compose down --rmi all
```

Now run replication queries:

```bash
# first for the provider:
#   - pglogical.create_node
#   - pglogical.create_replication_set
#   - pglogical.replication_set_add_table
docker exec -it pglogical-poc_pgprovider_1 \
  psql -U postgres -d pg_logical_replication -f /replication.sql

# second for the subscriber:
#   - pglogical.create_node
#   - pglogical.create_subscription
docker exec -it pglogical-poc_pgsubscriber_1 \
  psql -U postgres -d pg_logical_replication_results -f /replication.sql
```

And finally, check if the correct number of posts was replicated based on the arbitrary row filter `user_id = 1`:

```bash
# get number of posts having `user_id = 1`
docker exec -it pglogical-poc_pgprovider_1 \
  psql -U postgres -d pg_logical_replication \
    -c 'SELECT COUNT(*) FROM posts WHERE user_id = 1;'
 count
-------
    19
(1 row)

# get number of replicated posts
docker exec -it pglogical-poc_pgsubscriber_1 \
  psql -U postgres -d pg_logical_replication_results \
    -c 'SELECT COUNT(*) FROM posts;'
 count
-------
    19
(1 row)
```

_The actual number of posts can differ between runs, as the initial data is generated randomly. The important thing is that the two numbers are indeed equal._

Try to add more posts to the provider instance and check if the replication worked.

```bash
docker exec -it pglogical-poc_pgprovider_1 \
  psql -U postgres -d pg_logical_replication \
    -c 'INSERT INTO posts (SELECT generate_series(1001, 2000), FLOOR(random()*50)+1);'
INSERT 0 1000
```

SQL queries as plain text for convenience:

```sql
-- pgprovider
-- create node
SELECT pglogical.create_node(
  node_name := 'provider',
  dsn := 'host=pgprovider port=5432 dbname=pg_logical_replication user=postgres password=s3cr3t'
);

-- create replication set
SELECT pglogical.create_replication_set(
  set_name := 'posts',
  replicate_insert := TRUE,
  replicate_update := FALSE,
  replicate_delete := FALSE,
  replicate_truncate := FALSE
);

-- add table to replication set
SELECT pglogical.replication_set_add_table(
  set_name := 'posts',
  relation := 'posts',
  row_filter := 'user_id = 1',
  synchronize_data := TRUE
);

-- pgsubscriber
-- create subscriber node
SELECT pglogical.create_node(
  node_name := 'subscriber',
  dsn := 'host=pgsubscriber port=5432 dbname=pg_logical_replication_results user=postgres password=s3cr3t'
);

-- create subscription
SELECT pglogical.create_subscription(
  subscription_name := 'subscription',
  replication_sets := array['posts'],
  provider_dsn := 'host=pgprovider port=5432 dbname=pg_logical_replication user=postgres password=s3cr3t'
);

-- show subscription status
SELECT * FROM pglogical.show_subscription_status(
  subscription_name := 'subscription'
);

-- show subscription table
SELECT * FROM pglogical.show_subscription_table(
  subscription_name := 'subscription',
  relation := 'posts'
);

-- show `pglogical` relations
\dt pglogical.

-- describe `pglogical.local_sync_status`
\d+ pglogical.local_sync_status

-- show local sync status
SELECT sync_status
  FROM pglogical.local_sync_status
  WHERE sync_nspname = 'public' AND sync_relname = 'posts';
```

The sync states are defined [here](https://github.com/2ndQuadrant/pglogical/blob/REL2_x_STABLE/pglogical_sync.h#L43-L51) and mean the following:

- `\0`: `SYNC_STATUS_NONE` (No sync)
- `i`: `SYNC_STATUS_INIT` (Ask for sync)
- `s`: `SYNC_STATUS_STRUCTURE` (Sync structure)
- `d`: `SYNC_STATUS_DATA` (Sync data)
- `c`: `SYNC_STATUS_CONSTAINTS` (Sync constraints)
- `w`: `SYNC_STATUS_SYNCWAIT` (Table sync is waiting to get OK from main thread)
- `u`: `SYNC_STATUS_CATCHUP` (Catching up)
- `y`: `SYNC_STATUS_SYNCDONE` (Sync finished at lsn)
- `r`: `SYNC_STATUS_READY` (Sync done)

## Resources

- [PostgreSQL and the logical replication](https://blog.raveland.org/post/postgresql_lr_en/)
- [PostgreSQL replication with Docker](https://medium.com/swlh/postgresql-replication-with-docker-c6a904becf77)
- [Dockerfile](https://gist.github.com/asaaki/b07dccfd6ff6eed4c7b4ef279ade7b0c)
- [docker-pglogical](https://github.com/reediculous456/docker-pglogical/blob/master/Dockerfile)
- [Demystifying pglogical](http://thedumbtechguy.blogspot.com/2017/04/demystifying-pglogical-tutorial.html)
- [Short tutorial to setup replication using pglogical](https://gist.github.com/ratnakri/c22a7389d9fab788d7b8b12e2a6c337a)
- [How to configure pglogical](https://www.tutorialdba.com/2018/01/how-to-configure-pglogical-streaming.html)
- [PostgreSQL – logical replication with pglogical](https://blog.dbi-services.com/postgresql-logical-replication-with-pglogical/)
- [PG Phriday: Perfectly Logical](http://bonesmoses.org/2016/10/14/pg-phriday-perfectly-logical/)
