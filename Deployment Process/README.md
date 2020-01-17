# Deployment Process

## Overview

This deployment process is for developers and deployers to communicate with each others for deployment tasks. The process utilizes the Rocketor/goSQL tool (which developed by NexOps team) to reduce human actions and increase productivity by applying automation at some steps.

This deployment process **MUST be applied** at all times. The only exception accepted is to solve critical issues on **Production** and have prior **_Supervisor_**'s approval.

All deployments as of now will be handled in these 2 time ranges: [**10:30 AM** to **11:30 AM**] and [**03:30 PM** to **04:30 PM**]. Any requests sent outside the deployment time ranges will be handled in the next deployment time.

> **Important:** Please check your packages carefully before submit deployment request to avoid any delay that may happen because of the feedback after reviewing.

## Process Diagram

The following charts show the detail steps of deployment processes. Currently, we are having the diagram for [stored procedures & functions](#stored-procedures-functions) and [scripts](#scripts) packages. For how-to instructions for developers, please see the [tutorial](#developer-tutorial) section.

### Stored Procedures & Functions

```mermaid
sequenceDiagram
	participant Dev AS Developer
	participant Git AS Gitlab
	participant goSQL AS Rocketor/goSQL
	participant DBA AS Deployer
	participant Svr AS DB Servers

	activate Dev
	Dev ->> Git: ① Push
	activate Git
		Dev ->> goSQL: ② Create Package
		activate goSQL
			goSQL -->> Git: Get Branch Code
			Git -->> goSQL: 
			goSQL -x goSQL: New Package [SP/Func]
	deactivate Dev

	activate Dev
			activate goSQL
				Dev ->> goSQL: ③ Request
				goSQL -x DBA: Assign
				activate DBA
					DBA -->> goSQL: Get Package
					goSQL -->> DBA: 
					activate DBA
						DBA ->> DBA: Review Package
						alt Package is OK
							DBA ->> goSQL: Approve
					deactivate DBA
				deactivate DBA
							goSQL ->> Svr: Deploy
				activate Svr
							goSQL -x Dev: Notify
							opt Release
								goSQL --x Svr: Auto Deployment (Test, SIT, MNL)
							end
				deactivate Svr
			deactivate goSQL
		deactivate goSQL
						else Package is NG
		activate goSQL
			activate goSQL
				activate DBA
					activate DBA
							DBA ->> goSQL: Reject
					deactivate DBA
				deactivate DBA
							goSQL -x Dev: Feedback
			deactivate goSQL
		deactivate goSQL
						end
	deactivate Dev

	activate Dev
	Dev ->> Git: ④ Merge Request
	Git -x Git: Merge into master
	deactivate Dev
	deactivate Git
```

### Scripts

```mermaid
sequenceDiagram
	participant Dev AS Developer
	participant Git AS Gitlab
	participant goSQL AS Rocketor/goSQL
	participant DBA AS Deployer
	participant CM AS Change Management System
	participant VIP AS Approver
	participant Svr AS DB Servers

	activate Dev
	Dev ->> Git: ① Push (/scripts/ folder)
	activate Git
		Dev ->> goSQL: ② Create Package
		activate goSQL
			goSQL -->> Git: Get Branch Code
			Git -->> goSQL: 
			goSQL -x goSQL: New Package [Script]
	deactivate Dev

	activate Dev
			activate goSQL
				Dev ->> goSQL: ③ Request
				goSQL -x DBA: Assign
				activate DBA
					DBA -->> goSQL: Get Package
					goSQL -->> DBA: 
					activate DBA
						DBA ->> DBA: Review Package
						alt Package is OK
							DBA ->> CM: Create Ticket
						activate CM
							CM ->> VIP: Assign
							activate VIP
							VIP ->> VIP: Review
							alt Ticket is OK
								VIP ->> CM: Approve
							deactivate VIP
								DBA ->> Svr: Deploy
							activate Svr
								DBA ->> CM: Close
						deactivate CM
								DBA ->> goSQL: Approve
								opt HA & DR
								activate Svr
									DBA -->> Svr: Deploy HA & DR
								deactivate Svr
								end
						deactivate Svr
					deactivate DBA
				deactivate DBA
								goSQL -x Dev: Notify
			deactivate goSQL
		deactivate goSQL
							else Ticket is NG
		activate goSQL
			activate goSQL
				activate DBA
					activate DBA
						activate CM
							activate VIP
								VIP ->> DBA: Feedback
							deactivate VIP
						deactivate CM
								DBA ->> goSQL: Reject
					deactivate DBA
				deactivate DBA
								goSQL -x Dev: Feedback
			deactivate goSQL
		deactivate goSQL
							end
						else Package is NG
		activate goSQL
			activate goSQL
				activate DBA
					activate DBA
							DBA ->> goSQL: Reject
					deactivate DBA
				deactivate DBA
							goSQL -x Dev: Feedback
			deactivate goSQL
		deactivate goSQL
						end
	deactivate Dev

	activate Dev
	Dev ->> Git: ④ Merge Request
	Git -x Git: Merge into master
	deactivate Dev

	deactivate Git
```

## Tutorial

### Developer Tutorial

| Id | System | Step | Description | Video |
| :----: | :----: | :---- | :---- | :---- |
| 1 | Git | Rebase & Push | | ![](Tutorial/Video/Git-Rebase_Push.mp4) |
| 2 | goSQL | Create Package | | ![](Tutorial/Video/goSQL-NewSinglePackage.mp4) |
| 3 | goSQL | New Request | | ![](Tutorial/Video/goSQL-LocalDeploy_NewRequest.mp4) |
| 4 | goSQL | Receive Feedback | | ![](Tutorial/Video/goSQL-RequestApproved_Rejected.mp4) |
