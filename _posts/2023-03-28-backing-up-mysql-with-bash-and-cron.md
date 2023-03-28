---
layout: post
title: Backing up MySQL with bash and cron
date: 2023-03-28 00:15 -0400
categories: homelab self-hosting databases scripts mysql bash
tags: homelab blog bash mysql cron backup script automation database
---
## Summary

Backing up your MySQL database regularly is important to protect against data loss. I use this Bash script to automate the backup process and make it easier to keep my database backed up regularly. The script creates a backup of all databases and compresses it into a zip file. It also deletes old backups to save space. Finally it pings [healthcheck.io](https://healthcheck.io) to let me know that the backup was successful. The script can be scheduled to run automatically using cron.

### Script:

```bash
# Set current time for logs
now=$(TZ=America/New_York date +"%T")
echo -e "\nScript start time: $now"
echo -e "--------------------------------------------\n"

# Set user directory
path=/home/tbryant

# Backup storage directory
backupfolder=$path/mysql_backup/backups

# MySQL user
user=dbbackup

# Number of days to store the backup
keep_day=3

sqlfile=$backupfolder/all-database-$(date +%d-%m-%Y_%H-%M-%S).sql
zipfile=$backupfolder/all-database-$(date +%d-%m-%Y_%H-%M-%S).zip

# Create a backup
mysqldump  --defaults-file=$path/.my.cnf -u $user --all-databases > $sqlfile

if [ $? == 0 ]; then
  echo -e 'Sql dump created'
  echo -e "--------------------------------------------\n"
else
  echo 'mysqldump return non-zero code'
  exit
fi

# Compress backup
echo -e 'Compressing Sql file'
echo -e "--------------------------------------------\n"
zip -q $zipfile $sqlfile

if [ $? == 0 ]; then
  echo -e  'The backup was successfully compressed'
  echo -e "--------------------------------------------\n"
else
  echo 'Error compressing backup'
  exit
fi

rm $sqlfile
echo -e "$(basename ${zipfile}) was created successfully"
echo -e "--------------------------------------------\n"

# Delete old backups
find $backupfolder -mtime +$keep_day -delete

# Ping healthcheck.io
curl -fsS -m 10 --retry 5 -o /dev/null https://hc-ping.com/123456890-abcd-123456-sdef-1237653
```

## Steps

To use the script, follow these instructions:

1. Set the environment variables in the file: You will need to set the environment variables for user directory, backup storage directory, MySQL user, and the number of days to store the backups for. You can adjust these variables to fit your needs.

2. Create `.my.cnf` file: You will need to create a `.my.cnf` file in your users home directory. This file will contain the password for the MySQL user you specified in the script. The file should look like this:

```ini
[mysqldump]
user = dbbackup
password = password123
host = localhost
```

3. Save the script: Save the script to your server in a location where you can easily access it. I called mine `mysql_backup.sh`.

4. Make the script executable: To make the script executable, you will need to run the command `chmod +x mysql_backup.sh`.

5. Test the script: To test the script, run the command `./mysql_backup.sh` in the directory where the script is located. If the script runs successfully, it will create a backup of your MySQL database in the backup storage directory you specified. You should also see a log of output in the terminal similar to the following:

```shell
Script start time: 13:00:01
--------------------------------------------

Sql dump created
--------------------------------------------

Compressing Sql file
--------------------------------------------

The backup was successfully compressed
--------------------------------------------

all-database-28-03-2023_17-00-01.zip was created successfully
--------------------------------------------
```

6. Schedule the script: You can schedule the script to run automatically using a cron job.

```
# Edit the crontab file using a text editor of your choice
EDITOR=nano crontab -e

# Add the following line to the crontab file
0 9 * * * /path/to/scriptname.sh >/dev/null 2>&1
```

This will run the script every day at 9:00 AM. You can adjust the time and frequency to fit your needs. The last part redirects the output to `/dev/null` to prevent email notifications.

## Conclusion

There are many ways to back up your MySQL database. You can use a GUI tool such as [MySQL Workbench](https://www.mysql.com/products/workbench/) or a scrip that uses the command line tool [mysqldump](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html) like we've done above. No matter how you choose to back up your database, it is important to have a back up plan in place to protect against data loss. I also want to recommend that you not store your backups on the same server as your database. Consider storing them in a cloud storage service such as [Google Drive](https://www.google.com/drive/) or [Amazon S3](https://aws.amazon.com/s3/), or on a NAS/external hard drive.
