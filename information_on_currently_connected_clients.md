# Information on currently connected clients

At the SQL Server level, if you want to find information about currently connected clients, you should execute the following query:

```sql
SELECT DB_NAME(es.database_id) AS [DB] ,
       es.login_name ,
       es.nt_domain ,
       es.nt_user_name ,
       es.status ,
       es.host_name ,
       es.program_name ,
       es.last_successful_logon ,
       es.unsuccessful_logons ,
       ec.client_net_address ,
       ec.local_net_address ,
       ec.connect_time
FROM sys.dm_exec_sessions es
LEFT JOIN sys.dm_exec_connections ec ON ec.session_id = es.session_id
WHERE es.database_id > 0
GROUP BY es.database_id,
         es.login_name ,
         es.nt_domain ,
         es.nt_user_name ,
         es.status ,
         es.host_name ,
         es.program_name ,
         es.last_successful_logon ,
         es.unsuccessful_logons ,
         ec.client_net_address ,
         ec.local_net_address ,
         ec.connect_time
```