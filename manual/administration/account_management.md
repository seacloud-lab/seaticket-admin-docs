# Account management


## First team administrator

The standard Docker deployment creates the first team administrator from `INIT_SEATICKET_TEAM_ADMIN_EMAIL` and `INIT_SEATICKET_TEAM_ADMIN_PASSWORD` on first startup. Use this account for normal SeaTicket work and team administration.

## Add a system administrator

Ensure the container is running, then enter this command:

```bash
docker exec -it seaqa-web /scripts/reset-admin.sh
```

Enter the email address and password according to the prompts. This creates a system administrator, not a team administrator.

System administrators do not belong to a team and can only access the system administration interface. Most deployments do not need one for daily use.

## Forgot a system administrator password?

Create a new system administrator as described above, then use it to reset the password for the old system administrator account.
