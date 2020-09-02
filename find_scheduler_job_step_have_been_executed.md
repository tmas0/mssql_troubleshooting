# Find all Scheduler Job and steps have been executed

To find information about all scheduler jobs and the corresponding steps that have been executed, you should execute the following query:

```sql
WITH step_config AS (
    SELECT 
        J.job_id
        ,JobName = J.name
        ,JS.step_id, JS.step_name, JS.command
        ,StartIndex = 
            CASE 
                WHEN JS.command LIKE '/DTS%' OR JS.command LIKE '/SQL%' OR JS.command LIKE '/ISSERVER%' THEN CHARINDEX('\',JS.command, CHARINDEX('\',JS.command) + 1) --'
                WHEN JS.command LIKE '/SERVER%' THEN CHARINDEX('"', JS.Command, CHARINDEX(' ',command, CHARINDEX(' ',command) + 1) + 1) + 1
                ELSE 0
            END
        ,EndIndex = 
            CASE 
                WHEN JS.command LIKE '/DTS%' OR JS.command LIKE '/SQL%'  OR JS.command LIKE '/ISSERVER%' 
                    THEN  CHARINDEX('"',JS.command, CHARINDEX('\',JS.command, CHARINDEX('\',JS.command) + 1)) --'
                        - CHARINDEX('\',JS.command, CHARINDEX('\',JS.command) + 1) - 1 --'
                WHEN JS.command LIKE '/SERVER%' 
                    THEN  CHARINDEX('"',command, CHARINDEX('"', JS.Command, CHARINDEX(' ',command, CHARINDEX(' ',command) + 1) + 1) + 1)
                        - CHARINDEX('"', JS.Command, CHARINDEX(' ',command, CHARINDEX(' ',command) + 1) + 1) - 1
                ELSE 0
            END
    FROM msdb.dbo.sysjobsteps JS
    INNER JOIN msdb.dbo.sysjobs J
        ON JS.job_id = J.job_id
    WHERE JS.subsystem = 'SSIS'
)
SELECT 
CONVERT(DATETIME, RTRIM(h.run_date) + ' ' + STUFF(STUFF(REPLACE(STR(RTRIM(h.run_time),6,0), ' ','0'),3,0,':'),6,0,':')) as 'Run Date'
,j.name as 'Job Name'
, h.step_name as 'Step Name'
, Package = 
        CASE 
            WHEN sp.command LIKE '/DTS%' OR sp.command LIKE '/ISSERVER%' THEN SUBSTRING(sp.command, sp.StartIndex, sp.EndIndex)
            WHEN sp.command LIKE '/SQL%' THEN '\MSDB' + SUBSTRING(sp.command, sp.StartIndex, sp.EndIndex)
            WHEN sp.command LIKE '/SERVER%' THEN '\MSDB\' + SUBSTRING(sp.command, sp.StartIndex, sp.EndIndex)
            ELSE NULL
        END
, case h.run_status
when 1 then 'SUCCEEDED'
else 'FAILED'
end as 'Status'
, case h.run_status
when 0 then
h.message
else ''
end  as 'Step error message'
FROM [msdb].[dbo].[sysjobhistory] h 
inner join [msdb].[dbo].sysjobs j
ON j.job_id = h.job_id

    LEFT JOIN (
                SELECT 
                    [job_id]
                    , [run_date]
                    , [run_time]
                    , [run_status]
                    , [run_duration]
                    , [message]
                    , ROW_NUMBER() OVER (
                                            PARTITION BY [job_id] 
                                            ORDER BY [run_date] DESC, [run_time] DESC
                      ) AS RowNumber
                FROM [msdb].[dbo].[sysjobhistory] 
                WHERE [step_id] = 0
            ) AS [sJOBH]
            ON j.[job_id] = [sJOBH].[job_id]
            AND [sJOBH].[RowNumber] = 1
inner join step_config sp
ON sp.job_id = h.job_id and sp.step_id = h.step_id
WHERE  h.run_date =  format(getdate(), 'yyyyMMdd') 
and h.step_name <> '(Job outcome)' 
and j.enabled = 1
```