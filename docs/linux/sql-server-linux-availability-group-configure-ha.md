---
# required metadata

title: Configure availability group for SQL Server on Linux | Microsoft Docs
description: 
author: MikeRayMSFT 
ms.author: mikeray 
manager: jhubbard
ms.date: 03/17/2017
ms.topic: article
ms.prod: sql-linux
ms.technology: database-engine
ms.assetid: 

# optional metadata
# keywords: ""
# ROBOTS: ""
# audience: ""
# ms.devlang: ""
# ms.reviewer: ""
# ms.suite: ""
# ms.tgt_pltfrm: ""
# ms.custom: ""

---

# Configure Always On Availability Group for SQL Server on Linux

You can configure an Always On availability group for SQL Server on Linux. There are two architectures for availability groups. A *high availability* (HA) architecture uses a cluster manager to provide business continuity. This architecture can also include read-scale replicas. This document explains how to create the availability group HA architecture.

You can also create a *read-scale* availability group without a cluster manager. This architecture only provides read-only scalability. It does not provide HA. To create a read-scale availability group, see [Configure read-scale availability group for SQL Server on Linux](sql-server-linux-availability-group-configure-rs.md).

[!INCLUDE [Create Prereq](../includes/ss-linux-cluster-availability-group-create-prereq.md)]

## Create the availability group

Create the availability group. In order to create the availability group for HA on Linux, use the [CREATE AVAILABILITY GROUP](https://docs.microsoft.com/en-us/sql/t-sql/statements/create-availability-group-transact-sql) Transact-SQL DDL with `CLUSTER_TYPE = EXTERNAL`. 

The `EXTERNAL` value for `CLUSTER_TYPE` option specifies that the an external cluster entity manages the availability group. Pacemaker is an example of an external cluster entity. When the availability group `CLUSTER_TYPE = EXTERNAL`, set each replica `FAILOVER_MODE = EXTERNAL`. After you create the availability group, configure the cluster resource for the availability group using the cluster management tools - for example with Pacemaker use `pcs` (on RHEL, Ubuntu) or `crm` (on SLES). See the Linux distribution specific cluster configuration section for an end-to-end example.

The following Transact-SQL script creates an availability group for HA named `ag1`. The script configures the availability group replicas with `SEEDING_MODE = AUTOMATIC`. This setting causes SQL Server to automatically create the database on each secondary server. Update the following script for your environment. Replace the  `**<node1>**` and `**<node2>**` values with the names of the SQL Server instances that will host the replicas. Replace the `**<5022>**` with the port you set for the data mirroring endpoint. Run the following Transact-SQL on the SQL Server instance that will host the primary replica to create the availability group.

You can also configure an availability group with CLUSTER_TYPE=EXTERNAL using SQL Server Management Studio or PowerShell. 

```Transact-SQL
CREATE AVAILABILITY GROUP [ag1]
    WITH (DB_FAILOVER = ON, CLUSTER_TYPE = EXTERNAL)
    FOR REPLICA ON
        N'**<node1>**' 
		WITH (
		    ENDPOINT_URL = N'tcp://**<node1>:**<5022>**',
		    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
		    FAILOVER_MODE = EXTERNAL,
		    SEEDING_MODE = AUTOMATIC
		    ),
        N'**<node2>**' 
		WITH ( 
		    ENDPOINT_URL = N'tcp://**<node2>**:**<5022>**', 
		    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
		    FAILOVER_MODE = EXTERNAL,
		    SEEDING_MODE = AUTOMATIC
		    );
		
ALTER AVAILABILITY GROUP [ag1] GRANT CREATE ANY DATABASE;
```

### Join secondary replicas to the availability group

The following Transact-SQL script joins a SQL Server instance to an availability group named `ag1`. Update the script for your environment. On each SQL Server instance that will host a secondary replica, run the following Transact-SQL to join the availability group.

```Transact-SQL
ALTER AVAILABILITY GROUP [ag1] JOIN WITH (CLUSTER_TYPE = EXTERNAL);
		 
ALTER AVAILABILITY GROUP [ag1] GRANT CREATE ANY DATABASE;
```


[!INCLUDE [Create Post](../includes/ss-linux-cluster-availability-group-create-post.md)]

>[!IMPORTANT]
>After you create the availability group, you must configure integration with a cluster technology like Pacemaker for HA. In the case of a read-only scale-out architecture using availability groups, starting with SQL Server 2017, setting up a cluster is not required.
If you followed the steps in this document, you have an availability group that is not yet clustered. The next step is to add the cluster. While this is a valid configuration in read-scale/load balancing scenarios, it is not valid for HADR. To achieve HADR, you need to add the availability group as a cluster resource. See [Next steps](#next-steps) for instructions. 

## Notes

>[!IMPORTANT]
>After you configure the cluster and add the availability group as a cluster resource, you cannot use Transact-SQL to fail over the availability group resources. SQL Server cluster resources on Linux are not coupled as tightly with the operating system as they are on a Windows Server Failover Cluster (WSFC). SQL Server service is not aware of the presence of the cluster. All orchestration is done through the cluster management tools. In RHEL or Ubuntu use `pcs`. In SLES use `crm`. 

>[!IMPORTANT]
>If the availability group is a cluster resource, there is a known issue in current release where forced failover with data loss to an asynchronous replica does not work. This will be fixed in the upcoming release. Manual or automatic failover to a synchronous replica will succeed. 


## Next steps

[Configure Red Hat Enterprise Linux Cluster for SQL Server Availability Group Cluster Resources](sql-server-linux-availability-group-cluster-rhel.md)

[Configure SUSE Linux Enterprise Server Cluster for SQL Server Availability Group Cluster Resources](sql-server-linux-availability-group-cluster-sles.md)

[Configure Ubuntu Cluster for SQL Server Availability Group Cluster Resources](sql-server-linux-availability-group-cluster-ubuntu.md)