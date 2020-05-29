# How to handle connections for multiprocessing

I am working on a case which needs to crawl the machines in our lab - there should be 1000+ systems in the lab - check the health status of the systems, and record the current status of the machine into the DB. For each crawl, once I get the status of the system, I have to compare the current status with previous status record in the DB.

Some interaction with the systems are needed, now, I use the ssh to run some command remotely to get the status of the system, which is time consuming (via the network). And the query to the DB to get previous status from the DB is also time consuming. 

The communication with different systems are independent with each other, so, I tried to use the multiprocessing module provided by python, and now, I have a question that, how to sync the connection usage between so many processes.

At the very beginning, I tried with multiprocessing with the mysql connection pool provided by the mysql.connector.pooling, at that time, I thought, it would be some kind of MxN combination. M is the number of processes, and N is the number of connections in the pool. M processes can communicate with different systems in parallel, while the N connections (pre-established) can be shared by those M processes.

`Example 1:`

```python
with Pool(240) as p:
    p.map(func, lines)
p.close()
p.join()

```

```python
CON_POOL = mcp(32, "mypool", **config)
def func(array):
    conn = CON_POOL.get_connection()
    cursor = conn.cursor()
    cursor.execute(query)
    ...
    conn.close()
```

Once the script is triggered, I noticed that, the actual number of connections to the MySQL DB is larger than the number of connections in the pool. This will cause the actual connection number to reach the limitation of the MySQL DB, and hit the 'Too Many Connections' exception. The function CON_POOL is inherited by the subprocess, that is why the connection number keep increasing. Allocation Pool doesn't mean the connection established immediately for those 32 connections.

`Example 2:`

```python
  with Pool(initializer=init, processes=120) as p:
      p.map(func, lines)
  p.close()
  p.join()
```

```python
cnx = None

def init():
    global cnx
    #conn_pool = mcp(32, "mypool", **config)
    cnx = mysql.connector.connect(**config)

def func(array):
    cursor = cnx.cursor()
    ...
    cnx.close()

```

For example 2:

120 processes created at once and 120 connections are created at once in the init function. But, as the  script runs, the connection number dropped to < 10, why? And even hit the error as below finally. Can we say each connection used in the func is one of those 120 established connections?

```python
multiprocessing.pool.RemoteTraceback:
"""
Traceback (most recent call last):
  File "/usr/lib64/python3.6/multiprocessing/pool.py", line 119, in worker
    result = (True, func(*args, **kwds))
  File "/usr/lib64/python3.6/multiprocessing/pool.py", line 44, in mapstar
    return list(map(*args))
  File "./refresh_db_mt_common.py", line 162, in func
    cursor = cnx.cursor()
  File "/usr/lib64/python3.6/site-packages/mysql/connector/connection_cext.py", line 541, in cursor
    raise errors.OperationalError("MySQL Connection not available.")
mysql.connector.errors.OperationalError: MySQL Connection not available.
"""

```

This error is caused by the *cnx.close()* at the end of the func. 120 db connections are created by adding the connection creation login in the init() function. And later on, as one of the worker finished the assigned job, it goes to the end of the func(), and it close the connection. But since the worker will be used to handle other task in the backlog, but the question is, the per process connection is closed by previous task operation, which cause the current process has no connection at all, that is why it hit the connection not variable issue.

> When the init function is called?
> It is called only once when each worker process is created.

`Example 3`

```python
  with Pool(initializer=init, processes=120) as p:
      p.map(func, lines)
  p.close()
  p.join()
```

```python
conn_pool = None
def init():
    global conn_pool
    conn_pool = mcp(1, "mypool", **config)

def func(array):
    cnx = conn_pool.get_connection()
    ...
    cnx.close()

```

This is the best option.

Create the pool with 1 connection inside in the init function, that means 120 connection pool is created, and even we close the connection at the end of the func, the connection pool is still there, and can be used by the subprocess to handle the following up tasks with the same connection from the same pool.

So, the question is, what is the best way to handle my use case, how to parallelize the DB connection from multiple processes with efficiency?

I found [this](https://stackoverflow.com/questions/28638939/python3-x-how-to-share-a-database-connection-between-processes).

which can probably give me insight and correct my bad thought:

1. DB connection Pool is only helpful for multi-threads within a single process.
2. Don't pass the DB connection Pool around the processes.
3. Each process should use its own connection.

So, I changed my mind, and created a dedicate connection for each process. Due to limitation of the number that can be supported by the DB, I limit the number of processes as well.

No, the strange behavior due to the out-of-sync of the DB operation is gone.

Another problem that I have hit is regarding the *initializer* of the *Pool*.  I used to put the connection establishment logic in the initializer of the constructor of the Pool, but, found that the connection created in the initializer is not per process, all the processes share the same connection. I found this by checking the connection numbers in the MySQL DB.
