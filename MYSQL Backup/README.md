## Import required python libraries

```
import os
import time
import datetime
import pipes

```

## MySQL database details to which backup to be done. Make sure below user having enough privileges to take databases backup.

```
DB_HOST = 'localhost' 
DB_USER = 'root'
DB_USER_PASSWORD = '_mysql_user_password_'
DB_NAME = 'db_name_to_backup'
BACKUP_PATH = '/backup/dbbackup'
```

## Getting current DateTime to create the separate backup folder like "20180817-123433".

```
DATETIME = time.strftime('%Y%m%d-%H%M%S')
TODAYBACKUPPATH = BACKUP_PATH + '/' + DATETIME
```
## Checking if backup folder already exists or not. If not exists will create it.

```
try:
    os.stat(TODAYBACKUPPATH)
except:
    os.mkdir(TODAYBACKUPPATH)
```

## Starting actual database backup process.

```
db = DB_NAME
dumpcmd = "mysqldump -h " + DB_HOST + " -u " + DB_USER + " -p" + DB_USER_PASSWORD + " " + db + " > " + pipes.quote(TODAYBACKUPPATH) + "/" + db + ".sql"
os.system(dumpcmd)
gzipcmd = "gzip " + pipes.quote(TODAYBACKUPPATH) + "/" + db + ".sql"
os.system(gzipcmd)

print ("Backup script completed")
print ("Your backups have been created in '" + TODAYBACKUPPATH + "' directory")
```

```
import os
import time
import datetime
import pipes

DB_HOST = 'localhost' 
DB_USER = 'root'
DB_USER_PASSWORD = '_mysql_user_password_'
DB_NAME = 'db_name_to_backup'
BACKUP_PATH = '/backup/dbbackup'


DATETIME = time.strftime('%Y%m%d-%H%M%S')
TODAYBACKUPPATH = BACKUP_PATH + '/' + DATETIME

try:
    os.stat(TODAYBACKUPPATH)
except:
    os.mkdir(TODAYBACKUPPATH)

print ("Starting backup of database " + DB_NAME)
 
db = DB_NAME
dumpcmd = "mysqldump -h " + DB_HOST + " -u " + DB_USER + " -p" + DB_USER_PASSWORD + " " + db + " > " + pipes.quote(TODAYBACKUPPATH) + "/" + db + ".sql"
os.system(dumpcmd)
gzipcmd = "gzip " + pipes.quote(TODAYBACKUPPATH) + "/" + db + ".sql"
os.system(gzipcmd)

print ("Backup script completed")
print ("Your backups have been created in '" + TODAYBACKUPPATH + "' directory")
```
