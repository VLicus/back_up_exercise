# Backups exercise

## Backups types

1. Full: The most basic and comprehensive backup method, where all data is sent to another location.

2. Incremental: Backups all files that have changed since the last backup ocurred. Incremental backups are made possible by enabling the server's binary log, which the server uses to record data changes.

3. Differential: Backups only copies of all files that have changed since the last full backup.

## CLI tools to perform backups (full and incremental) and restores for the databases listed below:

### MySQL

Most backup strategies start with a complete (full) backup of the MySQL server, from which you can restore all databases and tables. After you have created a full backup, you might perform incremental backups (which are smaller and faster) for the next several backup tasks. You then make a full backup periodically to begin the cycle again.

#### Full MySQL backup

To perform a full backup of a MySQL database, it can be used `mysqldump` command, which is a MySQL utility for creating logical backups. It will be use as an example a MySQL database named "example"

```
mysqldump -u your_username -p your_password example > example_backup.sql
```

Where:
`your_username` must be your MySQL username.
`your_password` must be your MySQL password (empty if there is no password configured).
`example` is the name of your MySQL database.
`example_backup.sql` would be the name for your backup file.

Once you run this command, it will prompt you for the MySQL user password and after the password has been provided, it will create a SQL dump file `example_backup.sql` containing the complete structure of the specified MySQL database (example).

If you are running this command on the same machine where MySQL is installed, you might not need to provide `-u`and `-p` options and the command would look like this:

```
mysqldump example > example_backup.sql
```

This assumes that your MySQL user has the necessary privileges to access and dump the specified database (example).

A good practice will include the use of different flags that will be mentioned below:

`--single-transaction`: This option ensures that a consisten snapshot of the database is taken, using a single transaction. This is particulary useful for InnoDB tables, as it avoids locking the tables during the backup.

If binary loggin is enabled:

`--flush-logs`: This option causes the server to flush the logs after the backup, ensuring that the binary log files contain the events from the backup.

`--master-data=2`: This option includes the binary log coordinates as a comment at the end of the dump file. The value "2" specifies that it should incluse the file name and position. This information is useful for point-in-time recovery. `--master-data` seems that is deprecated and it will be removed in future versions, instead, it can be used `--source-data=2` in our example.

The command then would look like this:

```
mysqldump -u your_username -p your_password --single-transaction --flush-logs --master-data=2 example > example_backup.sql
```

Other graphical user interfaces and third-party tools such as phpMyAdmin or MySQL Workbench can be used but in this repository we will only focus on CLI tools.

#### Incremental MySQL backup

As mentioned before, incremental MySQL backups can NOT be done if we have not performed any other backup previously and it needs the use of binary logging records.

In order to enable binary logging, we need to modify the MySQL default configuration file, often located at `/etc/mysql/my.cnf or /etc/my.cnf`.

With your favourite text editor we will modify or add the following lines, in this case we will use `nano` text editor:

```
log_bin = /var/log/mysql/mysql-bin.log
```

> Replace `/var/log/mysql/` with the desired path for your binary log.

Then we should restart MySQL to apply the changes with the following command:

```
service mysql restart
```

Now we can perform an incremental backup.

First we need to find the `--start-position` of the binlog position from the last backup. In order to do so, we need to run this command:

```
SHOW MASTER STATUS;
```

It will give us information about the current binlog file name and its position.

Secondly we need to run the following command:

```
mysqlbinlog --start-position=xxx /var/log/mysql/mysql-bin.00000x >> example_backup.sql
```

Replacing "xxx" with the actual position of the binlog and "x" in "00000x" with the sequence number of the same file that you want to work with. An example would be (position:2 / sequence number:2):

```
mysqlbinlog --start-position=2 /var/log/mysql/mysql-bin.000002 >> example_backup.sql
```

> `>>` Is also for output redirection, but it appends the output to the  end of an existing file or creates a new file that does not exists.

In the case scenario that the binary logging file has been enabled before the full backup performance and the `mysqldump` command has been ran with the flag `--flush-logs` to close the current logs (mysql-bin.000001) and create a new one (mysql-bin.000002), we can just simply take an incremental backup by flushing the binary logs and saving the binary logs created from the last full backup by running the following command:

```
mysqladmin -uroot -p flush-logs
```

This will close the `mysql-bin.000002` and create a new one (`mysql-bin.000003`).

It can be checked by running `ls -l /var/log/mysql/`


### MongoDB

MongoDB backups create a BSON files inside the backup directory that represent the collections and documents of the MongoDB database, having `.bson` extension.

#### Full MongoDB backup

In order to perform a full backup in a Mongo database we will need to run the following command:

```
mongodump --out /path/to/fullbackup/directory
```

Replace */path/to/fullbackup/directory* with the actual path where you want to store the backup.

`mongodump` is an utility that creates a binary export of the content of the database.

`--out` specify the path where the BSON files will be written for the dumped databases. By default, mongodump saves output files in a directory named `dump` in the current working directory.

This command will create a backup for all databases, if you want to full backup a specific database, you can specify the database name:

```
mongodump --db example --out /path/to/fullbackup/directory
```

Being *example* the actual name of the specific database.

If MongoDB server requires authentication, it can be used *-u* for username and *-p* for password flags:

```
mongodump -u your_username -p your_password --out /path/to/fullbackup/directory
```

MongoDB also allow us to compress the backup files in order to save space by adding `--gzip` flag:

```
mongodump --gzip --out /path/to/fullbackup/directory
```

We can also include a timestamp in the backup directory name like this:

```
mongodump --out /path/to/backup/directory/example_$(date +"%Y%m%d_%H%M%S")
```

This will looks like example_*20240225_174530* for *February 25, 2024, at 17:45:30*

#### Incremental 

To create an incremental backup we just need to add -- oplog flag at the end of the comand

```
mongodump -- out /path/to/incremental/backup/directory -- oplog
```

#### Restoring DB

1. MySQL

Restoring from a full backup: 
mysql -u username -p dbname < full_backup.sql

Restoring from an incremental backup:

We can start doing incremental backup recoveries once a full backup recovery has been performed (see Restoring from a full backup). Then, with the use of the following command, we will start recovering the database from the *--start-position=xxx* and its source file */path/to/mysql-bin.00000x*:

```
mysqlbinlog --start-position=xxx /path/to/mysql-bin.00000x | mysql -u username -p dbname
```

If the incremental backup spawns multiple binary logs, we need to apply them in sequence:

```
mysqlbinlog --start-position=yyy /path/to/mysql-bin.0000yy | mysql -u username -p dbname
```

Setting up *--start-position* and the source path *0000yy* as the first binary log file right after the full backup file.

2. MongoDB

Restoring with MongoDB we just need the use of the following command:

```
mongorestore --oplogReplay /path/to/incremental/backup
```

`--oplogReplay` This option indicates that the oplog should be replayed during the restoration process. The oplog is a capped collection in MongoDB that records all write operations. By replaying the oplog, you can apply changes to the database to reach a specific point in time.

## Automate the backups from section 2 using bash scripting (either MySQL or MongoDB).

### MySQL

The bash script for automating full back up would look like:

```
#!/bin/bash

# Set the backup directory with the current date as the subfolder name
DIR=$(date +%Y%m%d_%H%M%S)
DEST=~/db_backups/$DIR
mkdir -p $DEST

# Replace the placeholders with your MySQL server details
MYSQL_HOST="localhost"
MYSQL_USER="your_username"
MYSQL_PASSWORD="your_password"
DATABASE_NAME="example"

# Use mysqldump to create a SQL backup file for the specified database
mysqldump -h $MYSQL_HOST -u $MYSQL_USER -p$MYSQL_PASSWORD $DATABASE_NAME > $DEST/${DATABASE_NAME}_full_backup.sql

# Optionally compress the backup
gzip $DEST/${DATABASE_NAME}_full_backup.sql

echo "Full backup of $DATABASE_NAME completed. Backup stored in: $DEST"
```

In order to perform an incremental backup script, we need to keep in mind that it will need to start from a certain position in the binary log, so the script will look like the following:

```
#!/bin/bash

# Set the backup directory with the current date as the subfolder name
DIR=$(date +%Y%m%d_%H%M%S)
DEST=~/db_backups/$DIR
mkdir -p $DEST

# Replace the placeholders with your MySQL server details
MYSQL_HOST="localhost"
MYSQL_USER="your_username"
MYSQL_PASSWORD="your_password"
DATABASE_NAME="example"

# Get the latest binary log file and position
LATEST_STATUS=$(mysql -h $MYSQL_HOST -u $MYSQL_USER -p$MYSQL_PASSWORD -e "SHOW MASTER STATUS;" 2>/dev/null)
LATEST_FILE=$(echo "$LATEST_STATUS" | awk 'NR==2 {print $1}')
LATEST_POSITION=$(echo "$LATEST_STATUS" | awk 'NR==2 {print $2}')

# Use mysqlbinlog to capture changes since the last backup
mysqlbinlog --start-position=$LATEST_POSITION /var/log/mysql/$LATEST_FILE > $DEST/${DATABASE_NAME}_incremental_backup.sql

# Optionally compress the incremental backup
gzip $DEST/${DATABASE_NAME}_incremental_backup.sql

echo "Incremental backup of $DATABASE_NAME completed. Backup stored in: $DEST"

```

Make sure that both scripts have executable permissions, if not, give them the permission by using the following command:

```
sudo chmod +x full_backup_scrip.sh
sudo chmod +x incremental_backup_script.sh
```

## Schedule the scripts from section 3 using cron job to perform a full backup once a week and an incremental backup every day.

In order to perform this backups as detailed, we need to modify the crontab file with `crontab -e` just by adding the following lines at the end of the crontab file:

```
# Schedule Full Backup every Sunday at 2:00 AM
0 2 * * 0 /path/to/full_backup_script.sh

# Schedule Incremental Backup every day at 3:00 AM
0 3 * * * /path/to/incremental_backup_script.sh
```

In this example:

> The full backup script will run every Sunday at 2:00 AM.
> The incremental backup script will run every day at 3:00 AM.