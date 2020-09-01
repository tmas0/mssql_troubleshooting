# Restore SQL Server database on RDS

## Prerequisites

* S3 bucket. For example:
```shell
aws s3api create-bucket --bucket backups --acl private
```
* Configure the required policy.
```
aws s3api put-bucket-policy --bucket backups --policy file://policy.json

policy.json:
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation" 
            ],
            "Resource": "arn:aws:s3:::backups" 
        },
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListMultipartUploadParts",
                "s3:AbortMultipartUpload" 
            ],
            "Resource": "arn:aws:s3:::backups/*" 
        }
    ]
}
```
* Upload all bak and trn files in previous S3 bucket.
```shell
aws s3 cp backupfile.bak s3://backups/backupfile.bak
```
* SQL Server instance configured with a SQLSERVER_BACKUP_RESTORE option.
** Add the SQLSERVER_BACKUP_RESTORE option to the option group.
```shell
aws rds add-option-to-option-group \
	--apply-immediately \
	--option-group-name mybackupgroup \
	--options "OptionName=SQLSERVER_BACKUP_RESTORE, \
	  OptionSettings=[{Name=IAM_ROLE_ARN,Value=arn:aws:iam::account-id:role/role-name}]"
```
** Apply the option group to the DB instance.
```shell
aws rds modify-db-instance \
	--db-instance-identifier mydbinstance \
	--option-group-name mybackupgroup \
	--apply-immediately
```

## Restore procedure

To restore your database, call the rds_restore_database stored procedure.

Restore all required bak files. One full and maybe one differential backup. Use the following SQL command.
```sql
exec msdb.dbo.rds_restore_database 
	@restore_db_name='database_name', 
	@s3_arn_to_restore_from='arn:aws:s3:::bucket_name/file_name.extension',
	@with_norecovery=0|1,
	[@kms_master_key_arn='arn:aws:kms:region:account-id:key/key-id'],
	[@type='DIFFERENTIAL|FULL'];
```

The following parameters are required:
* @restore_db_name – The name of the database to restore.
* @s3_arn_to_restore_from – The ARN indicating the Amazon S3 prefix and names of the backup files used to restore the database.
** For a single-file backup, provide the entire file name.
** For a multifile backup, provide the prefix that the files have in common, then suffix that with an asterisk (*).
** If @s3_arn_to_restore_from is empty, the following error message is returned: S3 ARN prefix cannot be empty.

The following parameter is required for differential restores, but optional for full restores:
* @with_norecovery – The recovery clause to use for the restore operation.
** Set it to 0 to restore with RECOVERY. In this case, the database is online after the restore.
** Set it to 1 to restore with NORECOVERY. In this case, the database remains in the RESTORING state after restore task completion. With this approach, you can do later differential restores.
** For DIFFERENTIAL restores, specify 0 or 1.
** For FULL restores, this value defaults to 0.

The following parameters are optional:
* @kms_master_key_arn – If you encrypted the backup file, the KMS key to use to decrypt the file.
When you specify a KMS key, client-side encryption is used.
* @type – The type of restore. Valid types are DIFFERENTIAL and FULL. The default value is FULL.

Examples of Full and differential backup:
```sql
-- FULL BACKUP
exec msdb.dbo.rds_restore_database 
    @restore_db_name='your_database', 
    @s3_arn_to_restore_from='arn:aws:s3:::backups/full_backupfile.bak',
    @with_norecovery=1,
    @type='FULL';
GO
```
```sql
-- INCREMENTAL BACKUP
exec msdb.dbo.rds_restore_database 
    @restore_db_name='your_database', 
    @s3_arn_to_restore_from='arn:aws:s3:::backups/differential_backupfile.bak',
    @with_norecovery=1,
    @type='DIFFERENTIAL';
GO
```

### PITR

If your restoration is PITR, you should restore all required transactional logs for your time recovery.
The following function helps you to restore your logs:
```sql
exec msdb.dbo.rds_restore_log 
	@restore_db_name='database_name', 
	@s3_arn_to_restore_from='arn:aws:s3:::bucket_name/log_file_name.extension',
	[@kms_master_key_arn='arn:aws:kms:region:account-id:key/key-id'],
	[@with_norecovery=0|1],
	[@stopat='datetime'];
```

The following parameters are required:
* @restore_db_name – The name of the database whose log to restore.
* @s3_arn_to_restore_from – The ARN indicating the Amazon S3 prefix and name of the log file used to restore the log. The file can have any extension, but .trn is usually used.
** If @s3_arn_to_restore_from is empty, the following error message is returned: S3 ARN prefix cannot be empty.

The following parameters are optional:
* @kms_master_key_arn – If you encrypted the log, the KMS key to use to decrypt the log.
* @with_norecovery – The recovery clause to use for the restore operation. This value defaults to 1.
** Set it to 0 to restore with RECOVERY. In this case, the database is online after the restore. You can't restore further log backups while the database is online.
** Set it to 1 to restore with NORECOVERY. In this case, the database remains in the RESTORING state after restore task completion. With this approach, you can do later log restores.
* @stopat – A value that specifies that the database is restored to its state at the date and time specified (in datetime format). Only transaction log records written before the specified date and time are applied to the database.
** If this parameter isn't specified (it is NULL), the complete log is restored.

For example:
```sql
-- TRANSACTIONAL LOG
exec msdb.dbo.rds_restore_log 
    @restore_db_name='your_database', 
    @s3_arn_to_restore_from='arn:aws:s3:::backups/tranlog_filename_1.trn',
    @with_norecovery=1;
GO

exec msdb.dbo.rds_restore_log 
    @restore_db_name='your_database', 
    @s3_arn_to_restore_from='arn:aws:s3:::backups/tranlog_filename_n.trn',
    @with_norecovery=0,
    @stopat='2020-07-04T16:47:41';
GO
```


## Status of tasks

```sql
-- STATUS
exec msdb.dbo.rds_task_status
	[@db_name='database_name'],
	[@task_id=ID_number];
GO

-- All database tasks
exec msdb.dbo.rds_task_status @db_name='my_database';
```

## Canceling a task

```sql
exec msdb.dbo.rds_cancel_task @task_id=ID_number;
```

## Drop database

```sql
exec msdb.dbo.rds_drop_database @db_name='database_name';
```

# Sources

* https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/SQLServer.Procedural.Importing.html#SQLServer.Procedural.Importing.Native.Using.Restore