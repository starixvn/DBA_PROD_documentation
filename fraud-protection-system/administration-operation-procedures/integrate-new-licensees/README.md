# Integrate New Licensees

## Objective

## Operation Guide

> **Important:** Please follow this instruction carefully to avoid any problems arise during the deployment.

### Collect Licensees' Information

Before proceeding with this guide, please confirm with related parties about the following information:

1. The correct name of the licensees
1. The type of each licensee
1. The location where each licensee's database will be deployed to:
	- `D:\MSSQL\DATA` on **FPS_DB01**
	- `E:\MSSQL\DATA` on **FPS_DB01**

### Create Database Folders

1. Remote into **FPS_DB01** server.
1. Input licensee names:
	1. Open file `T:\vnDBA\NewLicenseeScript\NewDatabases.txt`
	1. Enter new licensees' name into this file. One name on each line.
1. Execute command to create folders:
	1. Open **_Windows PowerShell_**.
	1. Execute this command if you want to deploy to `$D`

		```powershell
		T:\vnDBA\NewLicenseeScript\CreateFolders.ps1 -DataLocation "D:\MSSQL\DATA" -BackupLocation "T:\FPSDBBackup"
		```

	1. Execute this command if you want to deploy to `$E`

		```powershell
		T:\vnDBA\NewLicenseeScript\CreateFolders.ps1 -DataLocation "E:\MSSQL\DATA" -BackupLocation "T:\FPSDBBackup"
		```

> **Note:** If you want to deploy to both `$D` and `$E`, repeat the above step 2 (two) times.

### Generate & Deploy Databases

1. Open **SSMS** and connect to `FPS_DB01\MSSQLSERVERWEB` instance.
1. Choose an active licensee database, then **_Right Click_ » Task » Generate Scripts...** to open the wizard that help export all database objects.
1. Follow the wizard to script entire database and all database objects, then export the script to **_New query windows_**.
1. Check the generated script carefully, make sure the database file size is the initial value and point to the correct folder.
1. For each new licensee, replace the generated licensee name with the new one, then **Execute** the script to deploy the database.

### Configure New Licensees

#### Backup Process

1. Insert backup configuration

	```sql
	/*
		Creator:	Billy
		Server:		DBFRAUD
	*/
	
	SET NOCOUNT ON;
	
	IF	OBJECT_ID('tempdb..#NewLicensees') IS NOT NULL
		DROP TABLE	#NewLicensees;
	CREATE TABLE	#NewLicensees (
						SubscriberName		NVARCHAR(50)
							PRIMARY KEY CLUSTERED
					);
	
	INSERT INTO	#NewLicensees (SubscriberName)
	SELECT	DISTINCT
			input.SubscriberName
	FROM	(
		VALUES	('')		--<========== INPUT HERE ==========<
			,	('')		--<========== INPUT HERE ==========<
	) AS input(SubscriberName);
	
	INSERT INTO PerfMonDB.dbo.vnDBA_FPS_BackupDatabases (ServerName, DatabaseName, Threshold, Active, NoFullFiles, NoDiffFiles)
	SELECT	[ServerName]	= N'FPS_DB01\MSSQLSERVERWEB'
		,	[DatabaseName]	= N'FPS_SUB_' + input.SubscriberName
		,	[Threshold]		= NULL
		,	[Active]		= 1
		,	[NoFullFiles]	= 1
		,	[NoDiffFiles]	= 1
	FROM	#NewLicensees AS input WITH (NOLOCK);
	```

1. Perform the first FULL backup

	```sql
	/*
		Creator:	Billy
		Server:		FPS_DB01\MSSQLSERVERWEB
	*/
	
	SET NOCOUNT ON;
	
	IF	OBJECT_ID('tempdb..#NewLicensees') IS NOT NULL
		DROP TABLE	#NewLicensees;
	CREATE TABLE	#NewLicensees (
						SubscriberName		NVARCHAR(50)
							PRIMARY KEY CLUSTERED
					);
	
	INSERT INTO	#NewLicensees (SubscriberName)
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
		FROM	#NewLicensees AS input WITH (NOLOCK)
		FOR XML PATH ('')
		,	TYPE
	) AS xq(xml)
	CROSS APPLY	(
		SELECT	xq.xml.value('.[1]', 'NVARCHAR(MAX)')
	) AS qs(str);
	
	EXECUTE	sys.sp_executesql @Query;
	```

#### Configure FPS Admin

1. Insert metadata and initialize configurations

	```sql
	/*
		Creator:	Billy
		Server:		DBFRAUD
	*/
	
	SET NOCOUNT ON;
	
	USE	FPS_ADMIN;
	
	IF	OBJECT_ID('tempdb..#NewLicensees') IS NOT NULL
		DROP TABLE	#NewLicensees;
	CREATE TABLE	#NewLicensees (
						SubscriberName		NVARCHAR(50)
							PRIMARY KEY CLUSTERED
					);
	
	IF	OBJECT_ID('tempdb..#UserSubs') IS NOT NULL
		DROP TABLE	#UserSubs;
	CREATE TABLE	#UserSubs (
						UserID				INT
					,	UserName			NVARCHAR(50)
					,	SubscriberID		INT
					,	SiteName			NVARCHAR(50)
					,	PRIMARY KEY CLUSTERED (UserID, SubscriberID)
					)
	
	
	INSERT INTO	#NewLicensees (SubscriberName)
	SELECT	DISTINCT
			input.SubscriberName
	FROM	(
		VALUES	('')		--<========== INPUT HERE ==========<
			,	('')		--<========== INPUT HERE ==========<
	) AS input(SubscriberName);
	
	-- STEP 1: Insert new subscribers
	INSERT INTO	dbo.Subscribers
		(	      SubscriberName,    SubsciberKey,    IsActive,    EnableSysId,    EnableRuleSet,    ListIPs,    IsTest,    IntegrationDate,    IBCUserName_Prefix)
	SELECT	input.SubscriberName, dt.SubsciberKey, dt.IsActive, dt.EnableSysId, dt.EnableRuleSet, dt.ListIPs, dt.IsTest, dt.IntegrationDate, dt.IBCUserName_Prefix
	FROM		#NewLicensees AS input WITH (NOLOCK)
	CROSS APPLY	(
		SELECT	[SubsciberKey]			= LOWER(input.SubscriberName)
			,	[IsActive]				= 1
			,	[EnableSysId]			= 1
			,	[EnableRuleSet]			= 0
			,	[ListIPs]				= ''
			,	[IsTest]				= 0
			,	[IntegrationDate]		= CAST(CAST(GETDATE() AS DATE) AS SMALLDATETIME) 
			,	[IBCUserName_Prefix]	= (input.SubscriberName + '$')
	) AS dt
	WHERE	NOT EXIST (SELECT 1 FROM dbo.Subscribers AS _s WITH (NOLOCK) WHERE input.SubscriberName = _s.SubscriberName);

	-- STEP 2: Insert DB's config
	INSERT INTO	dbo.SubDBServers
		(	SubscriberId, ServerIP, ServerName, DbName, LinkedDevice, IsOnMain)
	SELECT	SubscriberID		= s.SubscriberID
		,	ServerIP			= '10.40.40.107'						-- server IP to locate
		,	ServerName			= 'FPSLIC107WEB'						-- linked server name to locate
		,	DbName				= 'FPS_SUB_' + input.SubscriberName		-- database name to create
		,	LinkedDevice		= 'FPSDEV'								-- linked server is hosting FPS_DEV
		,	IsOnMain			= 0
	FROM		#NewLicensees AS input WITH (NOLOCK)
	LEFT JOIN	dbo.Subscribers AS s WITH (NOLOCK)
			ON	input.SubscriberName = s.SubscriberName
	WHERE	NOT EXIST (SELECT 1 FROM dbo.SubDBServers AS _db WITH (NOLOCK) WHERE s.SubscriberID = _db.SubscriberID);
	GO

	-- STEP 3: Grant permission on subscriber
	INSERT INTO	#UserSubs (UserID, UserName, SubscriberID, SiteName)
	SELECT	u.UserID, u.UserName, t.SubscriberID, u.SiteName
	FROM		dbo.Users u WITH (NOLOCK)
	INNER JOIN	dbo.UserGroups ug WITH (NOLOCK)
			ON	u.GroupId = ug.GroupID
	CROSS APPLY	(
		SELECT	s.SubscriberID
		FROM		dbo.Subscribers s WITH (NOLOCK)
		INNER JOIN	#NewLicensees AS input WITH (NOLOCK)
				ON	s.SubscriberName = input.SubscriberName
	) AS t
	WHERE	u.StatusID = 1
		AND	(	ug.GroupKey = 'sa'
			OR	(	ug.GroupKey = 'administrators' AND ug.SiteName ='FPSADMIN')
			--OR	(	u.UserName IN ('admin', 'ronny', 'phuong', 'pgmrazor') AND ug.SiteName = 'FPS')
			);

	MERGE INTO	dbo.UserSubscribers AS t
	USING	#UserSubs AS s
		ON	t.UserId = s.UserId
		AND t.SubscriberID = s.SubscriberID
	WHEN NOT MATCHED BY TARGET THEN
		INSERT	(  UserId,   SubscriberID)
		VALUES	(s.UserId, s.SubscriberID)
	;

	-- grant action login  by default
	INSERT INTO	dbo.ActionSubscriber
		(	  SubscriberID, ActionID, IsActive)
	SELECT	s.SubscriberID, 2       , 1
	FROM		dbo.Subscribers s WITH (NOLOCK)
	INNER JOIN	#NewLicensees AS input WITH (NOLOCK)
			ON	s.SubscriberName = t.SubscriberName
	INNER JOIN	dbo.SubDBServers d WITH (NOLOCK)
			ON	s.SubscriberID = d.SubscriberId;
	```

### Configure [NDict](https://ndict.now/)

In order to synchronize the schema between licensee databases, we need to configure them into **_standby deployment process_**. The following sections show how to achieve this goal.

#### Database Configuration

1. Navigate to NDict's [Administration » Database](https://ndict.now/Database/Index) page.
1. Hit **+ Create Database** button.
1. Input the database's name and description (if needed) then **Submit**.

#### Standby Server Configuration

1. Navigate to NDict's [Administration » Standby Database](https://ndict.now/StandbyDb/Index) page.
1. Search for the configuration of `FPSLICALL` server, `FPS_DB01\MSSQLSERVERWEB (PROD)` DB server, `FPS_SUB_FPS` database.
1. Hit **Edit** button, then add all new licensee databases to **Standby Databases** field of the corresponding server.
1. Choose **Submit** to save the changes.