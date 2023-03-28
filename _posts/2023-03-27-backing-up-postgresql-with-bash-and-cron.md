---
layout: post
title: Backing up PostgreSQL with bash and cron
date: 2023-03-25 00:15 -0400
categories: homelab self-hosting databases scripts postgresql bash
tags: homelab blog bash postgresql cron backup script automation postgres database
---
## Summary

Backing up your PostgreSQL database is an important task that should be performed regularly to protect against data loss. One way to automate this process is by using a Bash script and cronjob. A Bash script is a simple way to automate tasks on a server, and it can be used to create backups of your PostgreSQL database.

## Steps

Here are the steps I took to create a Bash script to backup my PostgreSQL database:

1. Create a backup directory somewhere on your server: I created a `pg_backup` directory in my home directory.

2. Create a bash script to backup your database: A couple of great scripts can be found on the postgesql wiki: [pg_backup_rotated.sh](https://wiki.postgresql.org/wiki/Automated_Backup_on_Linux) and [pg_backup.sh](https://wiki.postgresql.org/wiki/Automated_Backup_on_Linux). I used the `pg_backup_rotated.sh` script as a starting point and modified it slightly to suit my needs.

The first thing I did was create the configuration file `pg_backup.config` in the same directory as the script. This file contains the configuration variables that are used by the script. The variables looked something like this:

```bash
# Optional system user to run backups as.  If the user the script is running as doesn't match this
# the script terminates.  Leave blank to skip check.
BACKUP_USER=

# Optional hostname to adhere to pg_hba policies.  Will default to "localhost" if none specified.
HOSTNAME=192.168.86.123

# Optional username to connect to database as.  Will default to "postgres" if none specified.
USERNAME=tbryant
PASSWORD=password123

# This dir will be created if it doesn't exist.  This must be writable by the user the script is
# running as.
BACKUP_DIR=/home/tbryant/pg_backup/backups/

# List of strings to match against in database name, separated by space or comma, for which we only
# wish to keep a backup of the schema, not the data. Any database names which contain any of these
# values will be considered candidates. (e.g. "system_log" will match "dev_system_log_2010-01")
SCHEMA_ONLY_LIST=""

# Will produce a custom-format backup if set to "yes"
ENABLE_CUSTOM_BACKUPS=no

# Will produce a gzipped plain-format backup if set to "yes"
ENABLE_PLAIN_BACKUPS=yes

# Will produce gzipped sql file containing the cluster globals, like users and passwords, if set to "yes"
ENABLE_GLOBALS_BACKUPS=yes

#### SETTINGS FOR ROTATED BACKUPS ####

# Which day to take the weekly backup from (1-7 = Monday-Sunday)
DAY_OF_WEEK_TO_KEEP=2

# Number of days to keep daily backups
DAYS_TO_KEEP=3

# How many weeks to keep weekly backups
WEEKS_TO_KEEP=5
```

I did not use all variables but left them for reference. For my purposes, I chose to create daily backups and keep them for 3 days. The one additional change I made to the config was I added a variable for `PASSWORD` so that I didn't have to enter my password every time I ran the script.

Once the config file was created, I modified the backup script slightly to make use of the new `PASSWORD` variable. I also added a few lines to the script to print out the start time of the backup process for the logs. The modified script looked like this:

```bash

#!/bin/bash

# Set current time for logs
now=$(TZ=America/New_York date +"%T")
echo "\n\nScript start time: $now"
echo -e "--------------------------------------------\n"

###########################
####### LOAD CONFIG #######
###########################

while [ $# -gt 0 ]; do
        case $1 in
                -c)
                        CONFIG_FILE_PATH="$2"
                        shift 2
                        ;;
                *)
                        ${ECHO} "Unknown Option \"$1\"" 1>&2
                        exit 2
                        ;;
        esac
done

if [ -z $CONFIG_FILE_PATH ] ; then
        SCRIPTPATH=$(cd ${0%/*} && pwd -P)
        CONFIG_FILE_PATH="${SCRIPTPATH}/pg_backup.config"
fi

if [ ! -r ${CONFIG_FILE_PATH} ] ; then
        echo "Could not load config file from ${CONFIG_FILE_PATH}" 1>&2
        exit 1
fi

source "${CONFIG_FILE_PATH}"

###########################
#### PRE-BACKUP CHECKS ####
###########################

# Make sure we're running as the required backup user
if [ "$BACKUP_USER" != "" -a "$(id -un)" != "$BACKUP_USER" ] ; then
        echo "This script must be run as $BACKUP_USER. Exiting." 1>&2
        exit 1
fi


###########################
### INITIALISE DEFAULTS ###
###########################

if [ ! $HOSTNAME ]; then
        HOSTNAME="localhost"
fi;

if [ ! $USERNAME ]; then
        USERNAME="postgres"
fi;


###########################
#### START THE BACKUPS ####
###########################

function perform_backups()
{
        SUFFIX=$1
        FINAL_BACKUP_DIR=$BACKUP_DIR"`date +\%Y-\%m-\%d`$SUFFIX/"

        echo "Making backup directory in $FINAL_BACKUP_DIR"

        if ! mkdir -p $FINAL_BACKUP_DIR; then
                echo "Cannot create backup directory in $FINAL_BACKUP_DIR. Go and fix it!" 1>&2
                exit 1;
        fi;

        #######################
        ### GLOBALS BACKUPS ###
        #######################

        echo -e "\n\nPerforming globals backup"
        echo -e "--------------------------------------------\n"

        if [ $ENABLE_GLOBALS_BACKUPS = "yes" ]
        then
                    echo "Globals backup"

                    set -o pipefail
                    if ! PGPASSWORD="$PASSWORD" pg_dumpall -g -h "$HOSTNAME" -U "$USERNAME" | gzip > $FINAL_BACKUP_DIR"globals".sql.gz.in_progress; then
                            echo "[!!ERROR!!] Failed to produce globals backup" 1>&2
                    else
                            mv $FINAL_BACKUP_DIR"globals".sql.gz.in_progress $FINAL_BACKUP_DIR"globals".sql.gz
                    fi
                    set +o pipefail
        else
                echo "None"
        fi


        ###########################
        ### SCHEMA-ONLY BACKUPS ###
        ###########################

        for SCHEMA_ONLY_DB in ${SCHEMA_ONLY_LIST//,/ }
        do
                SCHEMA_ONLY_CLAUSE="$SCHEMA_ONLY_CLAUSE or datname ~ '$SCHEMA_ONLY_DB'"
        done

        SCHEMA_ONLY_QUERY="select datname from pg_database where false $SCHEMA_ONLY_CLAUSE order by datname;"

        echo -e "\n\nPerforming schema-only backups"
        echo -e "--------------------------------------------\n"

        SCHEMA_ONLY_DB_LIST=`PGPASSWORD="$PASSWORD" psql -h "$HOSTNAME" -U "$USERNAME" -At -c "$SCHEMA_ONLY_QUERY" postgres`

        echo -e "The following databases were matched for schema-only backup:\n${SCHEMA_ONLY_DB_LIST}\n"

        for DATABASE in $SCHEMA_ONLY_DB_LIST
        do
                echo "Schema-only backup of $DATABASE"
                set -o pipefail
                if ! PGPASSWORD="$PASSWORD" pg_dump -Fp -s -h "$HOSTNAME" -U "$USERNAME" "$DATABASE" | gzip > $FINAL_BACKUP_DIR"$DATABASE"_SCHEMA.sql.gz.in_progress; then
                        echo "[!!ERROR!!] Failed to backup database schema of $DATABASE" 1>&2
                else
                        mv $FINAL_BACKUP_DIR"$DATABASE"_SCHEMA.sql.gz.in_progress $FINAL_BACKUP_DIR"$DATABASE"_SCHEMA.sql.gz
                fi
                set +o pipefail
        done


        ###########################
        ###### FULL BACKUPS #######
        ###########################

        for SCHEMA_ONLY_DB in ${SCHEMA_ONLY_LIST//,/ }
        do
                EXCLUDE_SCHEMA_ONLY_CLAUSE="$EXCLUDE_SCHEMA_ONLY_CLAUSE and datname !~ '$SCHEMA_ONLY_DB'"
        done

        FULL_BACKUP_QUERY="select datname from pg_database where not datistemplate and datallowconn $EXCLUDE_SCHEMA_ONLY_CLAUSE order by datname;"

        echo -e "\n\nPerforming full backups"
        echo -e "--------------------------------------------\n"

        for DATABASE in `PGPASSWORD="$PASSWORD" psql -h "$HOSTNAME" -U "$USERNAME" -At -c "$FULL_BACKUP_QUERY" postgres`
        do
                if [ $ENABLE_PLAIN_BACKUPS = "yes" ]
                then
                        echo "Plain backup of $DATABASE"

                        set -o pipefail
                        if ! PGPASSWORD="$PASSWORD" pg_dump -Fp -h "$HOSTNAME" -U "$USERNAME" "$DATABASE" | gzip > $FINAL_BACKUP_DIR"$DATABASE".sql.gz.in_progress; then
                                echo "[!!ERROR!!] Failed to produce plain backup database $DATABASE" 1>&2
                        else
                                mv $FINAL_BACKUP_DIR"$DATABASE".sql.gz.in_progress $FINAL_BACKUP_DIR"$DATABASE".sql.gz
                        fi
                        set +o pipefail

                fi

                if [ $ENABLE_CUSTOM_BACKUPS = "yes" ]
                then
                        echo "Custom backup of $DATABASE"

                        if ! PGPASSWORD="$PASSWORD" pg_dump -Fc -h "$HOSTNAME" -U "$USERNAME" "$DATABASE" -f $FINAL_BACKUP_DIR"$DATABASE".custom.in_progress; then
                                echo "[!!ERROR!!] Failed to produce custom backup database $DATABASE"
                        else
                                mv $FINAL_BACKUP_DIR"$DATABASE".custom.in_progress $FINAL_BACKUP_DIR"$DATABASE".custom
                        fi
                fi

        done

        echo -e "\nAll database backups complete!"
}

# DAILY BACKUPS

# Delete daily backups 3 days old or more
find $BACKUP_DIR -maxdepth 1 -mtime +$DAYS_TO_KEEP -name "*-daily" -exec rm -rf '{}' ';'

perform_backups "-daily"
curl -fsS -m 10 --retry 5 -o /dev/null https://hc-ping.com/c6c4321-987a-423ss-23abdef-1234567890
```

You'll notice the addition of `PGPASSWORD="$PASSWORD"` to all the `pg_dump` commands. This is needed because the `pg_dump` command will prompt for a password if it is not provided. In other words, I could run the script manually and provide the password when prompted, but I want to run the script automatically so I need to provide the password in the script. This likely isn't best practice from a security perspective, but it works for me in my little homelab environment.

The final addition I made is a one liner at the bottom `curl -fsS -m 10 --retry 5 -o /dev/null https://hc-ping.com/c6c4321-987a-423ss-23abdef-1234567890` which is a ping to a service called [Healthchecks](https://healthchecks.io/). This service allows you to monitor the status of your script and will send you a notification if the script fails to run. This is useful if you are running the script on a schedule and want to be notified if the script fails.

3. Make the script executable:

```bash
chmod +x pg_backup_rotated.sh
```

4. Verify that the script works:

```bash
./pg_backup_rotated.sh
```

If all goes well, you should see output similar to the following:

```bash
Script start time: 11:00:01
--------------------------------------------

Making backup directory in /home/tbryant/pg_backup/backups/2023-03-27-daily/

Performing globals backup
--------------------------------------------

Globals backup

Performing schema-only backups
--------------------------------------------

The following databases were matched for schema-only backup:

Performing full backups
--------------------------------------------

Plain backup of atuin
Plain backup of gitea
Plain backup of postgres
Plain backup of rx_resume
Plain backup of semaphore
Plain backup of zabbix_proxy
Plain backup of zzabbix

All database backups complete!
```

5. Finally, create a cron job to run the script at regular intervals:

```bash
# Edit the crontab file (I use nano here but you can use any text editor you like)
EDITOR=nano crontab -e

# Add the following line to the crontab file
0 15 * * * bash /home/tbryant/pg_backup/pg_backup_rotated.sh >> /home/tbryant/pg_backup/logs/backup.log 2>&1

# verify that the cron job was created
crontab -l
```

This will run the script every day at 3pm and log the output to a file called `backup.log` in the `logs` directory.

## Conclusion

There are many ways to backup your PostgreSQL database, and you can use the method that works best for you. Many cloud providers offer managed PostgreSQL services that include automated backups, so you may not need to worry about backing up your database. However, if you are running your own PostgreSQL database, for example in a homelab environment like I am, then you should consider backing up your database regularly to protect against data loss. A bash script is a simple way to automate the process and cron can be used to schedule the script to run at regular intervals ensuring that your database is backed up regularly.
