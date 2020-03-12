# Disable Terminated Licensees

## Objective

## Operation Guide

> **Important:** Please follow this instruction carefully to avoid any problems arise during the deployment.

### Configure [NDict](https://ndict.now/)

In order to prevent **_standby deployment process_** access disabled databases after finish this guide. We need to remove their configurations before proceeding the rest of this guide. The following sections show how to achieve this goal.

#### Standby Server Configuration

1. Navigate to NDict's [Administration » Standby Database](https://ndict.now/StandbyDb/Index) page.
1. Search for the configuration of `FPSLICALL` server, `FPS_DB01\MSSQLSERVERWEB (PROD)` DB server, `FPS_SUB_FPS` database.
1. Hit **Edit** button, then remove all terminated licensee databases from **Standby Databases** field of the corresponding server.
1. Choose **Submit** to save the changes.

#### Database Configuration

1. Navigate to NDict's [Administration » Database](https://ndict.now/Database/Index) page.
1. Search for the terminated databases, then hit **Delete** to remove them from NDict.

### Disable Terminated Licensees

#### Remove FPS Admin Configuration

1. Archive related configurations

	```sql
	/*
		Creator:	Billy
		Server:		DBFRAUD
	*/
	
	SET NOCOUNT ON;
	
	IF	OBJECT_ID('tempdb..#TerminatedLicensees') IS NOT NULL
		DROP TABLE	#TerminatedLicensees;
	CREATE TABLE	#TerminatedLicensees (
						SubscriberId		INT
					,	SubscriberName		NVARCHAR(50)
							PRIMARY KEY CLUSTERED
					);
	
	IF	OBJECT_ID('tempdb..#ActionSubscriber') IS NOT NULL
		DROP TABLE	#ActionSubscriber;
	CREATE TABLE	#ActionSubscriber (
						ActionID			INT
					,	SubscriberID		INT
					,	IsActive			BIT
	);
	
	INSERT INTO	#TerminatedLicensees (SubscriberName)
	SELECT	DISTINCT
			input.SubscriberName
	FROM	(
		VALUES	('')		--<========== INPUT HERE ==========<
			,	('')		--<========== INPUT HERE ==========<
	) AS input(SubscriberName);
	
	UPDATE	tmp WITH (TABLOCK)
	SET		tmp.SubscriberId = s.SubscriberID
	FROM		#TerminatedLicensees AS tmp
	INNER JOIN	FPS_ADMIN.dbo.Subscribers AS s WITH (NOLOCK)
			ON	tmp.SubscriberName = s.SubscriberName;
	
	SELECT	SubscriberId, SubscriberName
	FROM	#TerminatedLicensees AS tmp WITH (NOLOCK);
	
	
	DELETE	acs
	OUTPUT	deleted.ActionID, deleted.SubscriberID, deleted.IsActive
	INTO	#ActionSubscriber (ActionID, SubscriberID, IsActive)
	FROM		FPS_ADMIN.dbo.ActionSubscriber AS acs WITH (ROWLOCK)
	INNER JOIN	#TerminatedLicensees AS tmp WITH (NOLOCK)
			ON	tmp.SubscriberId = acs.SubscriberID;
	
	RAISERROR('~ DONE ~		ActionSubscriber		=> #ActionSubscriber', 0, 0) WITH NOWAIT;
	
	
	DELETE	uss
	OUTPUT	deleted.SubscriberID, deleted.UserID, deleted.CreatedDate, deleted.SubscriberID
	INTO	FPS_ADMIN.dbo.UserSubscribers_Terminated (SubscriberID, UserID, CreatedDate, SubId)
	FROM		FPS_ADMIN.dbo.UserSubscribers AS uss WITH (ROWLOCK)
	INNER JOIN	#TerminatedLicensees AS tmp WITH (NOLOCK)
			ON	tmp.SubscriberId = uss.SubscriberID;
			
	RAISERROR('~ DONE ~		UserSubscribers			=> UserSubscribers_Terminated', 0, 0) WITH NOWAIT;
	
	
	DELETE	sdb
	OUTPUT	deleted.SubscriberID, deleted.ServerIP, deleted.ServerName, deleted.DbName, deleted.LinkedDevice, deleted.IsOnMain
	INTO	FPS_ADMIN.dbo.SubDBServers_Terminated (SubscriberId, ServerIP, ServerName, DbName, LinkedDevice, IsOnMain)
	FROM		FPS_ADMIN.dbo.SubDBServers AS sdb WITH (ROWLOCK)
	INNER JOIN	#TerminatedLicensees AS tmp WITH (NOLOCK)
			ON	tmp.SubscriberId = sdb.SubscriberID;
			
	RAISERROR('~ DONE ~		SubDBServers			=> SubDBServers_Terminated', 0, 0) WITH NOWAIT;
	
	
	DELETE	uss
	OUTPUT	deleted.SubscriberName, deleted.SubsciberKey, deleted.IsActive, deleted.EnableSysId, deleted.EnableRuleSet, deleted.ListIPs, deleted.IsTest, deleted.IntegrationDate, deleted.IBCUserName_Prefix, deleted.SyncTicket, deleted.SubscriberID
	INTO	FPS_ADMIN.dbo.Subscribers_Terminated (SubscriberName, SubsciberKey, IsActive, EnableSysId, EnableRuleSet, ListIPs, IsTest, IntegrationDate, IBCUserName_Prefix, SyncTicket, SubscriberID)
	FROM		FPS_ADMIN.dbo.Subscribers AS uss WITH (ROWLOCK)
	INNER JOIN	#TerminatedLicensees AS tmp WITH (NOLOCK)
			ON	tmp.SubscriberId = uss.SubscriberID;
			
	RAISERROR('~ DONE ~		Subscribers				=> Subscribers_Terminated', 0, 0) WITH NOWAIT;
	```

#### Remove from Backup Process

1. Perform the last FULL backup

	```sql
	/*
		Creator:	Billy
		Server:		FPS_DB01\MSSQLSERVERWEB
	*/
	
	SET NOCOUNT ON;
	
	IF	OBJECT_ID('tempdb..#TerminatedLicensees') IS NOT NULL
		DROP TABLE	#TerminatedLicensees;
	CREATE TABLE	#TerminatedLicensees (
						SubscriberName		NVARCHAR(50)
							PRIMARY KEY CLUSTERED
					);
	
	INSERT INTO	#TerminatedLicensees (SubscriberName)
	SELECT	DISTINCT
			input.SubscriberName
	FROM	(
		VALUES	('')		--<========== INPUT HERE ==========<
			,	('')		--<========== INPUT HERE ==========<
	) AS input(SubscriberName);
	
	DECLARE	@Query		NVARCHAR(MAX)
		;
	
	SELECT	@Query = qs.str
	FROM	(
		SELECT	N'\
	IF	DB_ID(''FPS_SUB_' + input.SubscriberName + N''') IS NOT NULL
	BEGIN;
		EXECUTE	PerfmonDB.dbo.vnDBA_sp_FPS_BackupDatabases @BackupFolder = ''T:\FPSDBBackup'', @BackupDatabase = ''FPS_SUB_' + input.SubscriberName + N''', @Mode = 1;
		RAISERROR(N''~ Last FULL backup for FPS_SUB_' + input.SubscriberName + N' ~ DONE ~'', 0, 0) WITH NOWAIT;
	END;
	'
		FROM	#TerminatedLicensees AS input WITH (NOLOCK)
		FOR XML PATH ('')
		,	TYPE
	) AS xq(xml)
	CROSS APPLY	(
		SELECT	xq.xml.value('.[1]', 'NVARCHAR(MAX)')
	) AS qs(str);
	
	EXECUTE	sys.sp_executesql @Query;
	```

1. Remove Backup Configurations

	```sql
	/*
		Creator:	Billy
		Server:		DBFRAUD
	*/
	
	SET NOCOUNT ON;
	
	IF	OBJECT_ID('tempdb..#TerminatedLicensees') IS NOT NULL
		DROP TABLE	#TerminatedLicensees;
	CREATE TABLE	#TerminatedLicensees (
						SubscriberName		NVARCHAR(50)
							PRIMARY KEY CLUSTERED
					);
	
	INSERT INTO	#TerminatedLicensees (SubscriberName)
	SELECT	DISTINCT
			input.SubscriberName
	FROM	(
		VALUES	('')		--<========== INPUT HERE ==========<
			,	('')		--<========== INPUT HERE ==========<
	) AS input(SubscriberName);
	
	UPDATE	fb WITH (ROWLOCK)
	SET		Active = 0
	FROM		PerfMonDB.dbo.vnDBA_FPS_BackupDatabases AS fb
	INNER JOIN	#TerminatedLicensees AS tmp WITH (NOLOCK)
			ON	fb.DatabaseName = N'FPS_SUB_' + tmp.SubscriberName;
	```

#### Disable Terminated Licensees' Database

1. Disable related databases

	```sql
	/*
		Creator:	Billy
		Server:		FPS_DB01\MSSQLSERVERWEB
	*/
	
	SET NOCOUNT ON;
	
	IF	OBJECT_ID('tempdb..#TerminatedLicensees') IS NOT NULL
		DROP TABLE	#TerminatedLicensees;
	CREATE TABLE	#TerminatedLicensees (
						SubscriberName		NVARCHAR(50)
							PRIMARY KEY CLUSTERED
					);
	
	INSERT INTO	#TerminatedLicensees (SubscriberName)
	SELECT	DISTINCT
			input.SubscriberName
	FROM	(
		VALUES	('')		--<========== INPUT HERE ==========<
			,	('')		--<========== INPUT HERE ==========<
	) AS input(SubscriberName);
	
	DECLARE	@Query		NVARCHAR(MAX)
		;
	
	SELECT	@Query = qs.str
	FROM	(
		SELECT	N'\
	IF	DB_ID(''FPS_SUB_' + input.SubscriberName + N''') IS NOT NULL
	BEGIN;
		ALTER DATABASE	FPS_SUB_' + input.SubscriberName + N' SET OFFLINE, READ_ONLY WITH ROLLBACK IMMEDIATE;
		RAISERROR(N''~ Disable FPS_SUB_' + input.SubscriberName + N' ~ DONE ~'', 0, 0) WITH NOWAIT;
	END;
	'
		FROM	#TerminatedLicensees AS input WITH (NOLOCK)
		FOR XML PATH ('')
		,	TYPE
	) AS xq(xml)
	CROSS APPLY	(
		SELECT	xq.xml.value('.[1]', 'NVARCHAR(MAX)')
	) AS qs(str);
	
	EXECUTE	sys.sp_executesql @Query;
	```

1. Remote into **FPS_DB01** server.
1. Navigate to each terminated licensee database folder.
1. Zip all database files into `.7z` file.
1. Delete all database files, leave behind only `.7z` file for later restore if needed.