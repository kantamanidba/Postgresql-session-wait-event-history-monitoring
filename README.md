# RDS-Postgresql-session/wait-event-history-monitoring

Below is the step by step procedure for session/wait event history monitoring@AWS PostgreSQL RDS.

First we have enable to "PG_CRON" extension, Amazon RDS supports pg_cron in PostgreSQL versions 12.5 and higher.pg_cron allows you to use cron syntax to schedule PostgreSQL commands directly within your database. You can use pg_cron to schedule tasks such as periodically rolling up data for analytic reports, refreshing materialized views, and scheduling vacuum jobs to reclaim storage and also monitoring jobs.
  
Enable the pg_cron extension as follows:

Modify the parameter group associated with your PostgreSQL DB instance and add pg_cron to the shared_preload_libraries parameter value. This change requires a PostgreSQL DB instance restart to take effect.

after the PostgreSQL DB instance has restarted, run the following command using an account that has the rds_superuser permissions.

```
CREATE EXTENSION pg_cron;
```

Either use the default settings, or schedule jobs to run in other databases within your PostgreSQL DB instance. The pg_cron scheduler is set in the default PostgreSQL database named postgres. The pg_cron objects are created in this postgres database and all scheduling actions run in this database.

To schedule jobs to run in other databases within your PostgreSQL DB instance


Now connect to your required database in the instance and then create below objects.

create table, which is going to hold all session details.

```
CREATE TABLE public.wait_evnt_mntrg_tbl
( 
	db_name              varchar(20)  NULL ,
	user_name            varchar(20)  NULL ,
	proc_id              integer  NULL ,
	aplctn_name          varchar(200)  NULL ,
	clnt_adr             varchar(20)  NULL ,
	clnt_port            integer  NULL ,
	backend_strt         timestamp with time zone  NULL ,
	query_strt           timestamp with time zone  NULL ,
	wait_evnt_type       text  NULL ,
	wait_evnt            text  NULL ,
	state                text  NULL ,
	query                text  NULL 
);

```
create the procedure for capturing the session/wait event details.

```
create or replace procedure public.pr_wait_evnt_mntrg()
LANGUAGE plpgsql
AS $procedure$

declare
rec_curs RECORD;

seq_cursor cursor for select datname,usename,pid,application_name,client_addr,client_port,backend_start,query_start,wait_event_type,wait_event,state,query from pg_stat_activity where state != 'idle' and wait_event is not null ;

begin
open seq_cursor;
loop
fetch seq_cursor into rec_curs;
exit when not found;
if rec_curs.datname is not null then
insert into public.wait_evnt_mntrg_tbl (db_name,user_name,proc_id,aplctn_name,clnt_adr,clnt_port,backend_strt,query_strt,wait_evnt_type,wait_evnt,state,query) values (rec_curs.datname,rec_curs.usename,rec_curs.pid,rec_curs.application_name,rec_curs.client_addr,rec_curs.client_port,rec_curs.backend_start,rec_curs.query_start,rec_curs.wait_event_type,rec_curs.wait_event,rec_curs.state,rec_curs.query);
end if;
end loop;
close seq_cursor;
end; $procedure$;

```

Scheduling a cron job for a database:

The metadata for pg_cron is all held in the PostgreSQL default database named postgres. Because background workers are used for running the maintenance cron jobs, you can schedule a job in any of your databases within the PostgreSQL DB instance:

```
SELECT cron.schedule('Wait event Capture', '00,5,10,15,20,25,30,35,40,45,50,55 * * * *', 'call public.pr_wait_evnt_mntrg()');
```

As a user with the rds_superuser role, update the database column for the job that you just created so that it runs in another database within your PostgreSQL DB instance.

```
UPDATE cron.job SET database = 'datase_name' WHERE jobid = job_id;
```
verify job details by below query
```
select * from cron.job;
```
verify failed jobs by below query 
```
select * from cron.job_run_details where status = 'failed';
```
verify purging the wait_evnt_mntrg_tbl:

In order to maintain storage consumption low it's better purge the history data weekly/bi-weekly.
create the below procedure for purging the data.

```
create or replace procedure public.pr_wait_evnt_mntrg()
LANGUAGE plpgsql
AS $procedure$
begin
truncate table public.wait_evnt_mntrg_tbl;
end; $procedure$;
```
scheduling a cron job for purging the data on every sunday:

```
SELECT cron.schedule('purging the data', '0 0 * * 0', 'call public.pr_wait_evnt_mntrg()');
select * from cron.job; 
UPDATE cron.job SET database = 'datase_name' WHERE jobid = job_id;
```
Purging the pg_cron history table:

The cron.job_run_details table contains a history of cron jobs that can become very large over time. We recommend that you schedule a job that purges this table. For example, keeping a week's worth of entries might be sufficient for troubleshooting purposes.

The following example uses the cron.schedule function to schedule a job that runs every day at midnight to purge the cron.job_run_details table. The job keeps only the last seven days. Use your rds_superuser account to schedule the job such as the following.

```
SELECT cron.schedule('0 0 * * *', $$DELETE 
    FROM cron.job_run_details 
    WHERE end_time < now() – interval '7 days'$$);
```
