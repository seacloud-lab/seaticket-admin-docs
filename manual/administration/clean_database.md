# Clean Database

The SQL examples below use the default database name `seaticket_db`. If you have customized `MYSQL_DB_NAME`, use your configured database name instead.

## Session

Use the following command to clear expired session records in the SeaTicket database:

```
cd /opt/seaticket/seaticket
python3 manage.py clearsessions
```

!!! tip
    Enter the SeaTicket container, then go to `/opt/seaticket/seaticket`.

## SeaTicket Indexer periodic cleanup

SeaTicket Indexer runs periodic cleanup tasks (default: every 24 hours). The tasks remove records in the following databases.

### SeaTicket database (project connection metadata)

* Deleted connections in `project_connection`
* Deleted projects in `deleted_projects` (after cleanup, related connections for the project are removed)

### SeaDB (connection data tables)

* For recently updated connections (default: updated within the last 7 days)
  * Delete rows that are marked `deleted=true` and have been deleted for more than 14 days
* For deleted connections
  * Drop the SeaDB tables that belong to the connection


## Login

Use the following command to clean the login records:

```
use seaticket_db;
DELETE FROM sysadmin_extra_userloginlog WHERE to_days(now()) - to_days(login_date) > 90;
```

## Clean chat sessions

SeaTicket Web provides a management command to clean old chat data in batches. By default, it keeps the last 90 days.

!!! tip
    Enter the SeaTicket container, then go to `/opt/seaticket/seaticket`.

```
python3 manage.py clean_chat_sessions
```

You can change the retention period by passing `--days`:

```
python3 manage.py clean_chat_sessions --days=90
```

This command deletes:

* `chat_sessions` older than the cutoff date
* `chat_messages` linked to expired sessions
* `chat_message_thought_process` linked to expired sessions
* Orphaned `chat_messages` and `chat_message_thought_process` rows with no corresponding session


## Clean notifications

SeaTicket Events runs a periodic notification cleanup task (default: every 24 hours). By default it keeps the last 30 days, and the interval/retention can be configured in `NOTIFICATION_CLEANER`.

This task deletes old records in batches from:

* `user_notifications`
* `project_notification`

You can also clean these tables manually as follows.

```
use seaticket_db;
DELETE FROM user_notifications WHERE `timestamp` < '<cutoff_date>';

DELETE FROM project_notification WHERE `timestamp` < '<cutoff_date>';
```


## Clean portal external invitations

Portal external invitation tokens that expire after a fixed time window (default: 72 hours). Expired invitations should be cleaned to reduce stale records.

You can also clean these tables manually as follows.

```
DELETE FROM portal_external_invitations WHERE expire_time < '<cutoff_date>';
```


## Clean AI usage statistics

AI usage statistics are stored in `ai_usage_statistics` and should be cleaned by date. The filter field is `date`.

You can also clean these tables manually as follows.

```
DELETE FROM ai_usage_statistics WHERE `date` < '<cutoff_date>';
```
