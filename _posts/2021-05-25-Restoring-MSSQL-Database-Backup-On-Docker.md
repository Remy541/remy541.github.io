---
layout: post
title: Restoring an MS SQL Database Backup on a Docker Container
---

Docker is useful for so many things like setting up a testing environment or just to quickly fiddle around with some unknown software application without having to install it. The other day I needed to go through some old data, which was stored in an MS SQL database backup (.bak file) and since I am working on a MacBook Pro, I wasn't able to install MS SQL Server. Conveniently enough, Docker images exist for MS SQL Server, so I was able to run MS SQL Server (temporarily) on my laptop. The following ```docker run``` command will start a simple MS SQL Server 2019 (Express Edition) instance:

```
docker run --rm -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=some_password' -e 'MSSQL_PID=Express' mcr.microsoft.com/mssql/server:2019-latest
```

That was one less problem I had to think about. The next (and real) problem was being able to restore the backup file using some sql query. I found many questions and answers to this problem, but none used a Docker container for their MS SQL Server instance. The problem lies in the fact that the Docker container needs to have a volume mapped from the host to the container. The following ```docker run``` command will start an MS SQL Server instance with a volume mapping, and with a restriction so the Docker container is only reachable through my own machine (in this case, on port 12345):

```
docker run --rm -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=some_password' -e 'MSSQL_PID=Express' -p 127.0.0.1:12345:1433 -v /path/to/database/backup/:/tmp:rw mcr.microsoft.com/mssql/server:2019-latest
```

Now open up your favourite database management tool and open a connection to the SQL Server we just started. Enter the following query and your database is ready to use:

```
USE [master]
RESTORE DATABASE NAME_OF_DATABASE_TO_RESTORE
FROM  DISK = '/tmp/backup.bak'
WITH  FILE = 1,  NOUNLOAD,  REPLACE,  STATS = 5,
Move 'NAME_OF_DATABASE_TO_RESTORE' To '/var/opt/mssql/data/NAME_OF_DATABASE_TO_RESTORE.mdf',
Move 'NAME_OF_DATABASE_TO_RESTORE_log' To '/var/opt/mssql/data/NAME_OF_DATABASE_TO_RESTORE_log.ldf'
GO

use [NAME_OF_DATABASE_TO_RESTORE]
select *
from some_table
```

Voila!
