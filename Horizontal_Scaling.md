# Horizontal Scaling
Goal is to scale database horizontal using Partitions feature on PostgreSQL. Finally we’ll have a distributed Postgres system handling read and write queries. And user can query to any of the server to get the result.

We can also use logical replication. But I think replication and partition will work together in complete solution.

Steps to use partitioning for horizontal scaling in PostgreSQL:


1. Partition the Table
2. Distribute Partitions Across Servers
3. **Implement a Routing Layer**
4. Query Optimization
5. Partition Management

First I have tried partitioning on same server.


    CREATE TABLE sales (
        order_id INT,
        order_date DATE,
        order_amount NUMERIC
    )
    PARTITION BY RANGE (order_date);
    
    CREATE TABLE sales_2023 PARTITION OF sales
        FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');
    
    CREATE TABLE sales_2024 PARTITION OF sales
        FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
    
    -- Insert data into the sales_2023 partition
    INSERT INTO sales_2023 (order_id, order_date, order_amount)
    VALUES
        (1, '2023-03-15', 500.00),
        (2, '2023-06-01', 750.00),
        (3, '2023-09-20', 300.00),
        (4, '2023-11-30', 1000.00);
    
    -- Insert data into the sales_2024 partition
    INSERT INTO sales_2024 (order_id, order_date, order_amount)
    VALUES
        (5, '2024-02-10', 400.00),
        (6, '2024-05-05', 600.00),
        (7, '2024-08-15', 850.00),
        (8, '2024-11-22', 700.00);


I think important step is to decide how to **Implement a Routing Layer**
Here I’m going with pgpool-II, It is very useful Postgres extension. We are mainly interested in Load balancing.
Before setting up load balancing we need to setup replication between primary and secondary server running replica of primary cluster.

1. Create primary cluster
    initdb
2. Update configuration file in database
    listen address to *
    port 5433
3. Start data cluster
4. Create user for replication
    ./psql -U davindersingh postgres
      create user rep_user replication;
5. Update permission in pg_hba.conf to allow rep_user to connect.
    host    all             rep_user             127.0.0.1/32            trust


    -- indicates connected to primary database
    postgres=# SELECT pg_is_in_recovery();
     pg_is_in_recovery 
    -------------------
     f
    (1 row)


6. Create replica
        ./pg_basebackup -h localhost -U rep_user
         --checkpoint=fast -D replica_db -R --slot=logical_slot -C --port=5433



8. Change port on replica to avoid conflict as these are on same system
        e.g. port 5434
9. Start the replica 
    % ./pg_ctl -D replica_db start
    waiting for server to start....2024-06-19 09:43:38.506 IST [18260] LOG:  starting PostgreSQL 16.2 on x86_64-apple-darwin23.2.0, compiled by Apple clang version 15.0.0 (clang-1500.1.0.2.5), 64-bit
    2024-06-19 09:43:38.506 IST [18260] LOG:  listening on IPv6 address "::", port 5434
    2024-06-19 09:43:38.506 IST [18260] LOG:  listening on IPv4 address "0.0.0.0", port 5434
    2024-06-19 09:43:38.507 IST [18260] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5434"
    2024-06-19 09:43:38.515 IST [18263] LOG:  database system was interrupted; last known up at 2024-06-18 21:36:28 IST
    2024-06-19 09:43:38.664 IST [18263] LOG:  entering standby mode
    2024-06-19 09:43:38.665 IST [18263] LOG:  starting backup recovery with redo LSN 0/2000028, checkpoint LSN 0/2000060, on timeline ID 1
    2024-06-19 09:43:38.669 IST [18263] LOG:  redo starts at 0/2000028
    2024-06-19 09:43:38.669 IST [18263] LOG:  completed backup recovery with redo LSN 0/2000028 and end LSN 0/2000100
    2024-06-19 09:43:38.669 IST [18263] LOG:  consistent recovery state reached at 0/2000100
    2024-06-19 09:43:38.669 IST [18260] LOG:  database system is ready to accept read-only connections

 
 At this point we have setup logical replication, i.e. every update is being replicated in replication.
 We can check that by connecting directly to the replica.

    postgres=# SELECT pg_is_in_recovery();
     pg_is_in_recovery 
    -------------------
     t
    (1 row)

 
How do it check if logical replication is sending changes or something else ???

Now we can start with pgpool. 
1. Install pgpool

     brew install pgpool-ii

2. change configuration usr/local/etc

    cp pgpool.conf.sample to pgpool.conf

Following are some important configuration settings.

    # - pgpool Connection Settings -
    listen_addresses = '*'
    #port = 9999
    
    # - Backend Connection Settings -
    backend_hostname0 = 'localhost'
                                       # Host name or IP address to connect to for backend 0
    backend_port0 = 5433
                                       # Port number for backend 0
    backend_weight0 = 0
                                       # Weight for backend 0 (only in load balancing mode)
    backend_data_directory0 = '/Users/davindersingh/mywork/postgres/build/bin/primary_db'
                                       # Data directory for backend 0
    backend_hostname1 = 'localhost'
    backend_port1 = 5434
    backend_weight1 = 1
    backend_data_directory1 = '/Users/davindersingh/mywork/postgres/build/bin/replica_db'
    
    log_statement = on                                   # Log all statements
    log_per_node_statement = on
    # - Streaming -
    sr_check_user = 'rep_user'



3. Start pgpool
    pgpool -n > pgpool.log 2>&1 &


    c % pgpool -n &
    [1] 21601
    (base) 10:09:02:/usr/local/etc % 2024-06-19 10:09:03.957: main pid 21601: LOG:  Backend status file /usr/local/var/log/pgpool_status does not exist
    2024-06-19 10:09:03.957: main pid 21601: LOG:  health_check_stats_shared_memory_size: requested size: 12288
    2024-06-19 10:09:03.957: main pid 21601: LOG:  memory cache initialized
    2024-06-19 10:09:03.957: main pid 21601: DETAIL:  memcache blocks :1
    2024-06-19 10:09:03.957: main pid 21601: LOG:  allocating (3878152) bytes of shared memory segment
    2024-06-19 10:09:03.957: main pid 21601: LOG:  allocating shared memory segment of size: 3878152 
    2024-06-19 10:09:03.959: main pid 21601: LOG:  health_check_stats_shared_memory_size: requested size: 12288
    2024-06-19 10:09:03.959: main pid 21601: LOG:  health_check_stats_shared_memory_size: requested size: 12288
    2024-06-19 10:09:03.959: main pid 21601: LOG:  memory cache initialized
    2024-06-19 10:09:03.959: main pid 21601: DETAIL:  memcache blocks :1
    2024-06-19 10:09:03.978: main pid 21601: LOG:  pool_discard_oid_maps: discarded memqcache oid maps
    2024-06-19 10:09:03.979: main pid 21601: LOG:  create socket files[0]: /tmp/.s.PGSQL.9999
    2024-06-19 10:09:03.979: main pid 21601: LOG:  listen address[0]: *
    2024-06-19 10:09:03.979: main pid 21601: LOG:  Setting up socket for :::9999
    2024-06-19 10:09:03.980: main pid 21601: LOG:  Setting up socket for 0.0.0.0:9999
    2024-06-19 10:09:03.991: main pid 21601: LOG:  find_primary_node_repeatedly: waiting for finding a primary node
    2024-06-19 10:09:04.013: main pid 21601: LOG:  find_primary_node: primary node is 0
    2024-06-19 10:09:04.013: main pid 21601: LOG:  find_primary_node: standby node is 1
    2024-06-19 10:09:04.013: main pid 21601: LOG:  create socket files[0]: /tmp/.s.PGSQL.9898
    2024-06-19 10:09:04.014: main pid 21601: LOG:  listen address[0]: localhost
    2024-06-19 10:09:04.014: main pid 21601: LOG:  Setting up socket for ::1:9898
    2024-06-19 10:09:04.014: main pid 21601: LOG:  Setting up socket for 127.0.0.1:9898
    2024-06-19 10:09:04.016: pcp_main pid 21638: LOG:  PCP process: 21638 started
    2024-06-19 10:09:04.016: main pid 21601: LOG:  pgpool-II successfully started. version 4.5.2 (hotooriboshi)
    2024-06-19 10:09:04.017: main pid 21601: LOG:  node status[0]: 1
    2024-06-19 10:09:04.017: main pid 21601: LOG:  node status[1]: 2
    2024-06-19 10:09:04.016: sr_check_worker pid 21639: LOG:  process started
    2024-06-19 10:09:04.016: health_check pid 21640: LOG:  process started
    2024-06-19 10:09:04.016: health_check pid 21641: LOG:  process started
    2024-06-19 10:11:04.443: psql pid 21630: LOG:  statement: create table t2 (a int);
    2024-06-19 10:11:04.445: psql pid 21630: LOG:  DB node id: 0 backend pid: 21728 statement: SELECT pg_catalog.version()
    2024-06-19 10:11:04.449: psql pid 21630: LOG:  DB node id: 0 backend pid: 21728 statement: create table t2 (a int);
    2024-06-19 10:11:25.390: psql pid 21630: LOG:  statement: insert into t2 values (1);
    2024-06-19 10:11:25.391: psql pid 21630: LOG:  DB node id: 0 backend pid: 21728 statement: insert into t2 values (1);
    2024-06-19 10:11:43.751: psql pid 21630: LOG:  DB node id: 1 backend pid: 21729 statement: select * from t2;





