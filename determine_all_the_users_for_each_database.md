# Determine all the users for each database

If you want to find all the users for each database, you can execute the following query:

```sql
declare @sql nvarchar(max)
declare @database table (name sysname)
declare @dbname sysname

insert into @database
select name
from sys.databases
where state_desc = 'ONLINE'
    and name not in ('master', 'tempdb', 'model', 'msdb')

while exists(select top 1 1 from @database) begin
    select @dbname = name from @database
    print @dbname
    set @sql = 'use '+quotename(@dbname)+'
        SELECT '''+quotename(@dbname)+''';
        SELECT name as username
            from sys.database_principals
            where type not in (''A'', ''G'', ''R'', ''X'')
                  and sid is not null
                  and name != ''guest''
                  and authentication_type != 3
            order by username;'
    delete @database where name = @dbname
    exec sp_executesql @sql
end
```