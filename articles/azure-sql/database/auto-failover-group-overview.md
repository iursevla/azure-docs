---
title: Auto-failover groups
titleSuffix: Azure SQL Database & SQL Managed Instance
description: Auto-failover groups let you manage geo-replication and automatic / coordinated failover of a group of databases on a server, or all databases on a managed instance.
services: sql-database
ms.service: sql-db-mi
ms.subservice: high-availability
ms.custom: sqldbrb=2
ms.topic: conceptual
author: emlisa
ms.author: emlisa
ms.reviewer: kendralittle, mathoma
ms.date: 2/2/2022
---

# Use auto-failover groups to enable transparent and coordinated geo-failover of multiple databases
[!INCLUDE[appliesto-sqldb-sqlmi](../includes/appliesto-sqldb-sqlmi.md)]

The auto-failover groups feature allows you to manage the replication and failover of a group of databases on a server or all databases in a managed instance to another region. It is a declarative abstraction on top of the existing [active geo-replication](active-geo-replication-overview.md) feature, designed to simplify deployment and management of geo-replicated databases at scale. You can initiate a geo-failover manually or you can delegate it to the Azure service based on a user-defined policy. The latter option allows you to automatically recover multiple related databases in a secondary region after a catastrophic failure or other unplanned event that results in full or partial loss of the SQL Database or SQL Managed Instance availability in the primary region. A failover group can include one or multiple databases, typically used by the same application. Additionally, you can use the readable secondary databases to offload read-only query workloads.

> [!NOTE]
> Auto-failover groups support geo-replication of all databases in the group to only one secondary server or instance in a different region. If you need to create multiple Azure SQL Database geo-secondary replicas (in the same or different regions) for the same primary replica, use [active geo-replication](active-geo-replication-overview.md).
> 

When you are using auto-failover groups with automatic failover policy, an outage that impacts one or several of the databases in the group will result in an automatic geo-failover. Typically, these are outages that cannot be automatically mitigated by the built-in high availability infrastructure. Examples of geo-failover triggers include an incident caused by a SQL Database tenant ring or control ring being down due to an OS kernel memory leak on compute nodes, or an incident caused by one or more tenant rings being down because a wrong network cable was accidentally cut during routine hardware decommissioning. For more information, see [SQL Database High Availability](high-availability-sla.md).

In addition, auto-failover groups provide read-write and read-only listener end-points that remain unchanged during geo-failovers. Whether you use manual or automatic failover activation, a geo-failover switches all secondary databases in the group to the primary role. After the geo-failover is completed, the DNS record is automatically updated to redirect the endpoints to the new region. For geo-failover RPO and RTO, see [Overview of Business Continuity](business-continuity-high-availability-disaster-recover-hadr-overview.md).

When you are using auto-failover groups with automatic failover policy, an outage that impacts databases on a server or managed instance results in an automatic geo-failover. 

You can manage auto-failover group using:

- [Azure portal](geo-distributed-application-configure-tutorial.md)
- [Azure CLI: Failover Group](scripts/add-database-to-failover-group-cli.md)
- [PowerShell: Failover Group](scripts/add-database-to-failover-group-powershell.md)
- [REST API: Failover group](/rest/api/sql/failovergroups)

When configuring a failover group, ensure that authentication and network access on the secondary is set up to function correctly after geo-failover, when the geo-secondary becomes the new primary. For details, see [SQL Database security after disaster recovery](active-geo-replication-security-configure.md).

To achieve full business continuity, adding regional database redundancy is only part of the solution. Recovering an application (service) end-to-end after a catastrophic failure requires recovery of all components that constitute the service and any dependent services. Examples of these components include the client software (for example, a browser with a custom JavaScript), web front ends, storage, and DNS. It is critical that all components are resilient to the same failures and become available within the recovery time objective (RTO) of your application. Therefore, you need to identify all dependent services and understand the guarantees and capabilities they provide. Then, you must take adequate steps to ensure that your service functions during the failover of the services on which it depends. For more information about designing solutions for disaster recovery, see [Designing Cloud Solutions for Disaster Recovery Using active geo-replication](designing-cloud-solutions-for-disaster-recovery.md).

## <a name="terminology-and-capabilities"></a> Failover group terminology and capabilities

- **Failover group (FOG)**

  A failover group is a named group of databases managed by a single server or within a managed instance that can fail over as a unit to another region in case all or some primary databases become unavailable due to an outage in the primary region. When it's created for SQL Managed Instance, a failover group contains all user databases in the instance and therefore only one failover group can be configured on an instance.
  
  > [!IMPORTANT]
  > The name of the failover group must be globally unique within the `.database.windows.net` domain.

- **Servers**

  Some or all of the user databases on a logical server can be placed in a failover group. Also, a server supports multiple failover groups on a single server.

- **Primary**

  The server or managed instance that hosts the primary databases in the failover group.

- **Secondary**

  The server or managed instance that hosts the secondary databases in the failover group. The secondary cannot be in the same region as the primary.

- **Adding single databases to failover group**

  You can put several single databases on the same server into the same failover group. If you add a single database to the failover group, it automatically creates a secondary database using the same edition and compute size on secondary server. You specified that server when the failover group was created. If you add a database that already has a secondary database in the secondary server, that geo-replication link is inherited by the group. When you add a database that already has a secondary database in a server that is not part of the failover group, a new secondary is created in the secondary server.

  > [!IMPORTANT]
  > Make sure that the secondary server doesn't have a database with the same name unless it is an existing secondary database. In failover groups for SQL Managed Instance, all user databases are replicated. You cannot pick a subset of user databases for replication in the failover group.

- **Adding databases in elastic pool to failover group**

  You can put all or several databases within an elastic pool into the same failover group. If the primary database is in an elastic pool, the secondary is automatically created in the elastic pool with the same name (secondary pool). You must ensure that the secondary server contains an elastic pool with the same exact name and enough free capacity to host the secondary databases that will be created by the failover group. If you add a database in the pool that already has a secondary database in the secondary pool, that geo-replication link is inherited by the group. When you add a database that already has a secondary database in a server that is not part of the failover group, a new secondary is created in the secondary pool.
  
- **Initial Seeding**

  When adding databases, elastic pools, or managed instances to a failover group, there is an initial seeding phase before data replication starts. The initial seeding phase is the longest and most expensive operation. Once initial seeding completes, data is synchronized, and then only subsequent data changes are replicated. The time it takes for the initial seeding to complete depends on the size of your data, number of replicated databases, the load on primary databases, and the speed of the link between the primary and secondary. Under normal circumstances, possible seeding speed is up to 500 GB an hour for SQL Database, and up to 360 GB an hour for SQL Managed Instance. Seeding is performed for all databases in parallel.

  For SQL Managed Instance, consider the speed of the Express Route link between the two instances when estimating the time of the initial seeding phase. If the speed of the link between the two instances is slower than what is necessary, the time to seed is likely to be noticeably impacted. You can use the stated seeding speed, number of databases, total size of data, and the link speed to estimate how long the initial seeding phase will take before data replication starts. For example, for a single 100 GB database, the initial seed phase would take about 1.2 hours if the link is capable of pushing 84 GB per hour, and if there are no other databases being seeded. If the link can only transfer 10 GB per hour, then seeding a 100 GB database will take about 10 hours. If there are multiple databases to replicate, seeding will be executed in parallel, and, when combined with a slow link speed, the initial seeding phase may take considerably longer, especially if the parallel seeding of data from all databases exceeds the available link bandwidth. If the network bandwidth between two instances is limited and you are adding multiple managed instances to a failover group, consider adding multiple managed instances to the failover group sequentially, one by one. Given an appropriately sized gateway SKU between the two managed instances, and if corporate network bandwidth allows it, it's possible to achieve speeds as high as 360 GB an hour.  

- **DNS zone**

  A unique ID that is automatically generated when a new SQL Managed Instance is created. A multi-domain (SAN) certificate for this instance is provisioned to authenticate the client connections to any instance in the same DNS zone. The two managed instances in the same failover group must share the DNS zone.
  
  > [!NOTE]
  > A DNS zone ID is not required or used for failover groups created for SQL Database.

- **Failover group read-write listener**

  A DNS CNAME record that points to the current primary. It is created automatically when the failover group is created and allows the read-write workload to transparently reconnect to the primary when the primary changes after failover. When the failover group is created on a server, the DNS CNAME record for the listener URL is formed as `<fog-name>.database.windows.net`. When the failover group is created on a SQL Managed Instance, the DNS CNAME record for the listener URL is formed as `<fog-name>.<zone_id>.database.windows.net`.

- **Failover group read-only listener**

  A DNS CNAME record that points to the current secondary. It is created automatically when the failover group is created and allows the read-only SQL workload to transparently connect to the secondary when the secondary changes after failover. When the failover group is created on a server, the DNS CNAME record for the listener URL is formed as `<fog-name>.secondary.database.windows.net`. When the failover group is created on a SQL Managed Instance, the DNS CNAME record for the listener URL is formed as `<fog-name>.secondary.<zone_id>.database.windows.net`.

- **Automatic failover policy**

  By default, a failover group is configured with an automatic failover policy. The system triggers a geo-failover after the failure is detected and the grace period has expired. The system must verify that the outage cannot be mitigated by the built-in [high availability infrastructure](high-availability-sla.md), for example due to the scale of the impact. If you want to control the geo-failover workflow from the application or manually, you can turn off automatic failover policy.
  
  > [!NOTE]
  > Because verification of the scale of the outage and how quickly it can be mitigated involves human actions, the grace period cannot be set below one hour. This limitation applies to all databases in the failover group regardless of their data synchronization state.

- **Read-only failover policy**

  By default, the failover of the read-only listener is disabled. It ensures that the performance of the primary is not impacted when the secondary is offline. However, it also means the read-only sessions will not be able to connect until the secondary is recovered. If you cannot tolerate downtime for the read-only sessions and can use the primary for both read-only and read-write traffic at the expense of the potential performance degradation of the primary, you can enable failover for the read-only listener by configuring the `AllowReadOnlyFailoverToPrimary` property. In that case, the read-only traffic will be automatically redirected to the primary if the secondary is not available.

  > [!NOTE]
  > The `AllowReadOnlyFailoverToPrimary` property only has effect if automatic failover policy is enabled and an automatic geo-failover has been triggered. In that case, if the property is set to True, the new primary will serve both read-write and read-only sessions.

- **Planned failover**

  Planned failover performs full data synchronization between primary and secondary databases before the secondary switches to the primary role. This guarantees no data loss. Planned failover is used in the following scenarios:

  - Perform disaster recovery (DR) drills in production when data loss is not acceptable
  - Relocate the databases to a different region
  - Return the databases to the primary region after the outage has been mitigated (failback)

- **Unplanned failover**

  Unplanned or forced failover immediately switches the secondary to the primary role without waiting for recent changes to propagate from the primary. This operation may result in data loss. Unplanned failover is used as a recovery method during outages when the primary is not accessible. When the outage is mitigated, the old primary will automatically reconnect and become a new secondary. A planned failover may be executed to fail back, returning the replicas to their original primary and secondary roles.

- **Manual failover**

  You can initiate a geo-failover manually at any time regardless of the automatic failover configuration. During an outage that impacts the primary, if automatic failover policy is not configured, a manual failover is required to promote the secondary to the primary role. You can initiate a forced (unplanned) or friendly (planned) failover. A friendly failover is only possible when the old primary is accessible, and can be used to relocate the primary to the secondary region without data loss. When a failover is completed, the DNS records are automatically updated to ensure connectivity to the new primary.

- **Grace period with data loss**

  Because the secondary databases are synchronized using asynchronous replication, an automatic geo-failover may result in data loss. You can customize the automatic failover policy to reflect your application’s tolerance to data loss. By configuring `GracePeriodWithDataLossHours`, you can control how long the system waits before initiating a forced failover, which may result in data loss.

- **Multiple failover groups**

  You can configure multiple failover groups for the same pair of servers to control the scope of geo-failovers. Each group fails over independently. If your tenant-per-database application is deployed in multiple regions and uses elastic pools, you can use this capability to mix primary and secondary databases in each pool. This way you may be able to reduce the impact of an outage to only some tenant databases.

  > [!NOTE]
  > SQL Managed Instance does not support multiple failover groups.
  
## Permissions

Permissions for a failover group are managed via [Azure role-based access control (Azure RBAC)](../../role-based-access-control/overview.md). The [SQL Server Contributor](../../role-based-access-control/built-in-roles.md#sql-server-contributor) role has all the necessary permissions to manage failover groups.

### <a name="create-failover-group"></a> Create a failover group

To create a failover group, you need Azure RBAC write access to both the primary and secondary servers, and to all databases in the failover group. For a SQL Managed Instance, you need Azure RBAC write access to both the primary and secondary SQL Managed Instance, but permissions on individual databases are not relevant, because individual SQL Managed Instance databases cannot be added to or removed from a failover group.

### Update a failover group

To update a failover group, you need Azure RBAC write access to the failover group, and all databases on the current primary server or managed instance.  

### Fail over a failover group

To fail over a failover group, you need Azure RBAC write access to the failover group on the new primary server or managed instance.

## <a name="best-practices-for-sql-database"></a> Failover group best practices for SQL Database

The auto-failover group must be configured on the primary server and will connect it to the secondary server in a different Azure region. The groups can include all or some databases in these servers. The following diagram illustrates a typical configuration of a geo-redundant cloud application using multiple databases and auto-failover group.

![Diagram shows a typical configuration of a geo-redundant cloud application using multiple databases and auto-failover group.](./media/auto-failover-group-overview/auto-failover-group.png)

> [!NOTE]
> See [Add SQL Database to a failover group](failover-group-add-single-database-tutorial.md) for a detailed step-by-step tutorial adding a database in SQL Database to a failover group.

When designing a service with business continuity in mind, follow these general guidelines:

### <a name="using-one-or-several-failover-groups-to-manage-failover-of-multiple-databases"></a> Use one or several failover groups to manage failover of multiple databases

One or many failover groups can be created between two servers in different regions (primary and secondary servers). Each group can include one or several databases that are recovered as a unit in case all or some primary databases become unavailable due to an outage in the primary region. Creating a failover group creates geo-secondary databases with the same service objective as the primary. If you add an existing geo-replication relationship to a failover group, make sure the geo-secondary is configured with the same service tier and compute size as the primary.
  
### <a name="using-read-write-listener-for-oltp-workload"></a> Use the read-write listener to connect to primary

For read-write workloads, use `<fog-name>.database.windows.net` as the server name in the connection string. Connections will be automatically directed to the primary. This name does not change after failover. Note the failover involves updating the DNS record so the client connections are redirected to the new primary only after the client DNS cache is refreshed. The time to live (TTL) of the primary and secondary listener DNS record is 30 seconds.

### <a name="using-read-only-listener-for-read-only-workload"></a> Use the read-only listener to connect to geo-secondary

If you have logically isolated read-only workloads that are tolerant to data latency, you can run them on the geo-secondary. For read-only sessions, use `<fog-name>.secondary.database.windows.net` as the server name in the connection string. Connections will be automatically directed to the geo-secondary. It is also recommended that you indicate read intent in the connection string by using `ApplicationIntent=ReadOnly`.

> [!NOTE]
> In Premium, Business Critical, and Hyperscale service tiers, SQL Database supports the use of [read-only replicas](read-scale-out.md) to offload read-only query workloads, using the `ApplicationIntent=ReadOnly` parameter in the connection string. When you have configured a geo-secondary, you can use this capability to connect to either a read-only replica in the primary location or in the geo-replicated location.
>
> - To connect to a read-only replica in the primary location, use `ApplicationIntent=ReadOnly` and `<fog-name>.database.windows.net`.
> - To connect to a read-only replica in the secondary location, use `ApplicationIntent=ReadOnly` and `<fog-name>.secondary.database.windows.net`.

### <a name="preparing-for-performance-degradation"></a> Potential performance degradation after geo-failover

A typical Azure application uses multiple Azure services and consists of multiple components. The automatic geo-failover of the failover group is triggered based on the state the Azure SQL components alone. Other Azure services in the primary region may not be affected by the outage and their components may still be available in that region. Once the primary databases switch to the secondary (DR) region, the latency between the dependent components may increase. To avoid the impact of higher latency on the application's performance, ensure the redundancy of all the application's components in the DR region, follow these [network security guidelines](#failover-groups-and-network-security), and orchestrate the geo-failover of relevant application components together with the database.

### <a name="preparing-for-data-loss"></a> Potential data loss after geo-failover

If an outage occurs in the primary region, recent transactions may not be able to replicate to the geo-secondary. If the automatic failover policy is configured, the system waits for the period you specified by `GracePeriodWithDataLossHours` before initiating an automatic geo-failover. The default value is 1 hour. This favors database availability over no data loss. Setting `GracePeriodWithDataLossHours` to a larger number, such as 24 hours, or disabling automatic geo-failover lets you reduce the likelihood of data loss at the expense of database availability.

> [!IMPORTANT]
> Elastic pools with 800 or fewer DTUs or 8 or fewer vCores, and more than 250 databases may encounter issues including longer planned geo-failovers and degraded performance. These issues are more likely to occur for write intensive workloads, when geo-replicas are widely separated by geography, or when multiple secondary geo-replicas are used for each database. A symptom of these issues is an increase in geo-replication lag over time, potentially leading to a more extensive data loss in an outage. This lag can be monitored using [sys.dm_geo_replication_link_status](/sql/relational-databases/system-dynamic-management-views/sys-dm-geo-replication-link-status-azure-sql-database). If these issues occur, then mitigation includes scaling up the pool to have more DTUs or vCores, or reducing the number of geo-replicated databases in the pool.

### <a name="changing-secondary-region-of-the-failover-group"></a> Change the secondary region of a failover group

To illustrate the change sequence, we will assume that server A is the primary server, server B is the existing secondary server, and server C is the new secondary in the third region. To make the transition, follow these steps:

1. Create additional secondaries of each database on server A to server C using [active geo-replication](active-geo-replication-overview.md). Each database on server A will have two secondaries, one on server B and one on server C. This will guarantee that the primary databases remain protected during the transition.
2. Delete the failover group. At this point login attempts using failover group endpoints will be failing.
3. Re-create the failover group with the same name between servers A and C.
4. Add all primary databases on server A to the new failover group. At this point the login attempts will stop failing.
5. Delete server B. All databases on B will be deleted automatically.

### <a name="changing-primary-region-of-the-failover-group"></a> Change the primary region of a failover group

To illustrate the change sequence, we will assume server A is the primary server, server B is the existing secondary server, and server C is the new primary in the third region. To make the transition, follow these steps:

1. Perform a planned geo-failover to switch the primary server to B. Server A will become the new secondary server. The failover may result in several minutes of downtime. The actual time will depend on the size of failover group.
2. Create additional secondaries of each database on server B to server C using [active geo-replication](active-geo-replication-overview.md). Each database on server B will have two secondaries, one on server A and one on server C. This will guarantee that the primary databases remain protected during the transition.
3. Delete the failover group. At this point login attempts using failover group endpoints will be failing.
4. Re-create the failover group with the same name between servers B and C.
5. Add all primary databases on B to the new failover group. At this point the login attempts will stop failing.
6. Perform a planned geo-failover of the failover group to switch B and C. Now server C will become the primary and B the secondary. All secondary databases on server A will be automatically linked to the primaries on C. As in step 1, the failover may result in several minutes of downtime.
7. Delete server A. All databases on A will be deleted automatically.

> [!IMPORTANT]
> When the failover group is deleted, the DNS records for the listener endpoints are also deleted. At that point, there is a non-zero probability of somebody else creating a failover group or a server DNS alias with the same name. Because failover group names and DNS aliases must be globally unique, this will prevent you from using the same name again. To minimize this risk, don't use generic failover group names.

## <a name="best-practices-for-sql-managed-instance"></a> Failover group best practices for SQL Managed Instance

The auto-failover group must be configured on the primary instance and will connect it to the secondary instance in a different Azure region.  All user databases in the instance will be replicated to the secondary instance. System databases like _master_ and _msdb_ will not be replicated.

The following diagram illustrates a typical configuration of a geo-redundant cloud application using managed instance and auto-failover group.

![auto failover diagram](./media/auto-failover-group-overview/auto-failover-group-mi.png)

> [!NOTE]
> See [Add managed instance to a failover group](../managed-instance/failover-group-add-instance-tutorial.md) for a detailed step-by-step tutorial adding a SQL Managed Instance to use failover group.

> [!IMPORTANT]
> If you deploy auto-failover groups in a hub-and-spoke network topology cross-region, replication traffic should go directly between the two managed instance subnets rather than be directed through the hub networks.

If your application uses SQL Managed Instance as the data tier, follow these general guidelines when designing for business continuity:

### <a name="creating-the-secondary-instance"></a> Create the geo-secondary managed instance

To ensure non-interrupted connectivity to the primary SQL Managed Instance after failover, both the primary and secondary instances must be in the same DNS zone. It will guarantee that the same multi-domain (SAN) certificate can be used to authenticate client connections to either of the two instances in the failover group. When your application is ready for production deployment, create a secondary SQL Managed Instance in a different region and make sure it shares the DNS zone with the primary SQL Managed Instance. You can do it by specifying an optional parameter during creation. If you are using PowerShell or the REST API, the name of the optional parameter is `DNSZonePartner`. The name of the corresponding optional field in the Azure portal is *Primary Managed Instance*.

> [!IMPORTANT]
> The first managed instance created in the subnet determines DNS zone for all subsequent instances in the same subnet. This means that two instances from the same subnet cannot belong to different DNS zones.

For more information about creating the secondary SQL Managed Instance in the same DNS zone as the primary instance, see [Create a secondary managed instance](../managed-instance/failover-group-add-instance-tutorial.md#create-a-secondary-managed-instance).

### <a name="using-geo-paired-regions"></a> Use paired regions

Deploy both managed instances to [paired regions](../../availability-zones/cross-region-replication-azure.md) for performance reasons. SQL Managed Instance failover groups in paired regions have better performance compared to unpaired regions.

### <a name="enabling-replication-traffic-between-two-instances"></a> Enable geo-replication traffic between two managed instances

Because each managed instance is isolated in its own VNet, two-directional traffic between these VNets must be allowed. See [Azure VPN gateway](../../vpn-gateway/vpn-gateway-about-vpngateways.md)

### <a name="creating-a-failover-group-between-managed-instances-in-different-subscriptions"></a> Create a failover group between managed instances in different subscriptions

You can create a failover group between SQL Managed Instances in two different subscriptions, as long as subscriptions are associated to the same [Azure Active Directory Tenant](../../active-directory/fundamentals/active-directory-whatis.md#terminology). When using PowerShell API, you can do it by specifying the `PartnerSubscriptionId` parameter for the secondary SQL Managed Instance. When using REST API, each instance ID included in the `properties.managedInstancePairs` parameter can have its own Subscription ID.
  
> [!IMPORTANT]
> Azure portal does not support creation of failover groups across different subscriptions. Also, for the existing failover groups across different subscriptions and/or resource groups, failover cannot be initiated manually via portal from the primary SQL Managed Instance. Initiate it from the geo-secondary instance instead.

### <a name="managing-failover-to-secondary-instance"></a> Manage geo-failover to a geo-secondary instance

The failover group will manage geo-failover of all databases on the primary managed instance. When a group is created, each database in the instance will be automatically geo-replicated to the geo-secondary instance. You cannot use failover groups to initiate a partial failover of a subset of databases.

> [!IMPORTANT]
> If a database is dropped on the primary managed instance, it will also be dropped automatically on the geo-secondary managed instance.

### <a name="using-read-write-listener-for-oltp-workload"></a> Use the read-write listener to connect to the primary managed instance

For read-write workloads, use `<fog-name>.zone_id.database.windows.net` as the server name. Connections will be automatically directed to the primary. This name does not change after failover. The geo-failover involves updating the DNS record, so the client connections are redirected to the new primary only after the client DNS cache is refreshed. Because the secondary instance shares the DNS zone with the primary, the client application will be able to reconnect to it using the same server-side SAN certificate. The read-write listener and read-only listener cannot be reached via [public endpoint for managed instance](../managed-instance/public-endpoint-configure.md).

### <a name="using-read-only-listener-to-connect-to-the-secondary-instance"></a> Use the read-only listener to connect to the geo-secondary managed instance

If you have logically isolated read-only workloads that are tolerant to data latency, you can run them on the geo-secondary. To connect directly to the geo-secondary, use `<fog-name>.secondary.<zone_id>.database.windows.net` as the server name. The read-write listener and read-only listener cannot be reached via [public endpoint for managed instance](../managed-instance/public-endpoint-configure.md).

> [!NOTE]
> In the Business Critical tier, SQL Managed Instance supports the use of [read-only replicas](read-scale-out.md) to offload read-only query workloads, using the `ApplicationIntent=ReadOnly` parameter in the connection string. When you have configured a geo-replicated secondary, you can use this capability to connect to either a read-only replica in the primary location or in the geo-replicated location.
>
> - To connect to a read-only replica in the primary location, use `ApplicationIntent=ReadOnly` and `<fog-name>.<zone_id>.database.windows.net`.
> - To connect to a read-only replica in the secondary location, use `ApplicationIntent=ReadOnly` and `<fog-name>.secondary.<zone_id>.database.windows.net`.

### Potential performance degradation after failover to the geo-secondary managed instance

A typical Azure application uses multiple Azure services and consists of multiple components. The automatic geo-failover of the failover group is triggered based on the state the Azure SQL components alone. Other Azure services in the primary region may not be affected by the outage and their components may still be available in that region. Once the primary databases switch to the secondary region, the latency between the dependent components may increase. To avoid the impact of higher latency on the application's performance, ensure the redundancy of all the application's components in the secondary region and fail over application components together with the database. At configuration time, follow [network security guidelines](#failover-groups-and-network-security) to ensure connectivity to the database in the secondary region.

### Potential data loss after failover to the geo-secondary managed instance

If an outage occurs in the primary region, recent transactions may not be able to replicate to the geo-secondary. If the automatic failover policy is configured, a geo-failover is triggered if there is zero data loss, to the best of our knowledge. Otherwise, failover is deferred for the period you specify using `GracePeriodWithDataLossHours`. If you configured the automatic failover policy, be prepared for data loss. In general, during outages, Azure favors availability. Setting `GracePeriodWithDataLossHours` to a larger number, such as 24 hours, or disabling automatic geo-failover lets you reduce the likelihood of data loss at the expense of database availability.

The DNS update of the read-write listener will happen immediately after the failover is initiated. This operation will not result in data loss. However, the process of switching database roles can take up to 5 minutes under normal conditions. Until it is completed, some databases in the new primary instance will still be read-only. If a failover is initiated using PowerShell, the operation to switch the primary replica role is synchronous. If it is initiated using the Azure portal, the UI will indicate completion status. If it is initiated using the REST API, use standard Azure Resource Manager’s polling mechanism to monitor for completion.

> [!IMPORTANT]
> Use manual planned failover to move the primary back to the original location once the outage that caused the geo-failover is mitigated.
  
### Change the secondary region of the managed instance failover group

Let's assume that instance A is the primary instance, instance B is the existing secondary instance, and instance C is the new secondary instance in the third region. To make the transition, follow these steps:

1. Create instance C with same size as A and in the same DNS zone.
2. Delete the failover group between instances A and B. At this point the logins will be failing because the SQL aliases for the failover group listeners have been deleted and the gateway will not recognize the failover group name. The secondary databases will be disconnected from the primaries and will become read-write databases.
3. Create a failover group with the same name between instance A and C. Follow the instructions in [failover group with SQL Managed Instance tutorial](../managed-instance/failover-group-add-instance-tutorial.md). This is a size-of-data operation and will complete when all databases from instance A are seeded and synchronized.
4. Delete instance B if not needed to avoid unnecessary charges.

> [!NOTE]
> After step 2 and until step 3 is completed the databases in instance A will remain unprotected from a catastrophic failure of instance A.

### Change the primary region of the managed instance failover group

Let's assume instance A is the primary instance, instance B is the existing secondary instance, and instance C is the new primary instance in the third region. To make the transition, follow these steps:

1. Create instance C with same size as B and in the same DNS zone.
2. Connect to instance B and manually failover to switch the primary instance to B. Instance A will become the new secondary instance automatically.
3. Delete the failover group between instances A and B. At this point login attempts using failover group endpoints will be failing. The secondary databases on A will be disconnected from the primaries and will become read-write databases.
4. Create a failover group with the same name between instance A and C. Follow the instructions in the [failover group with managed instance tutorial](../managed-instance/failover-group-add-instance-tutorial.md). This is a size-of-data operation and will complete when all databases from instance A are seeded and synchronized. At this point login attempts will stop failing.
5. Delete instance A if not needed to avoid unnecessary charges.

> [!CAUTION]
> After step 3 and until step 4 is completed the databases in instance A will remain unprotected from a catastrophic failure of instance A.

> [!IMPORTANT]
> When the failover group is deleted, the DNS records for the listener endpoints are also deleted. At that point, there is a non-zero probability of somebody else creating a failover group with the same name. Because failover group names must be globally unique, this will prevent you from using the same name again. To minimize this risk, don't use generic failover group names.

### Enable scenarios dependent on objects from the system databases

System databases are **not** replicated to the secondary instance in a failover group. To enable scenarios that depend on objects from the system databases, make sure to create the same objects on the secondary instance and keep them synchronized with the primary instance. 

For example, if you plan to use the same logins on the secondary instance, make sure to create them with the identical SID. 

```SQL
-- Code to create login on the secondary instance
CREATE LOGIN foo WITH PASSWORD = '<enterStrongPasswordHere>', SID = <login_sid>;
``` 
### Synchronize instance properties and retention policies between primary and secondary instance

Instances in a failover group remain separate Azure resources, and no changes made to the configuration of the primary instance will be automatically replicated to the secondary instance. Make sure to perform all relevant changes both on primary _and_ secondary instance. For example, if you change backup storage redundancy or long-term backup retention policy on primary instance, make sure to change it on secondary instance as well.

## Failover groups and network security

For some applications the security rules require that the network access to the data tier is restricted to a specific component or components such as a VM, web service, etc. This requirement presents some challenges for business continuity design and the use of failover groups. Consider the following options when implementing such restricted access.

### <a name="using-failover-groups-and-virtual-network-rules"></a> Use failover groups and virtual network service endpoints

If you are using [Virtual Network service endpoints and rules](vnet-service-endpoint-rule-overview.md) to restrict access to your database in SQL Database or SQL Managed Instance, be aware that each virtual network service endpoint applies to only one Azure region. The endpoint does not enable other regions to accept communication from the subnet. Therefore, only the client applications deployed in the same region can connect to the primary database. Since a geo-failover results in the SQL Database client sessions being rerouted to a server in a different (secondary) region, these sessions will fail if originated from a client outside of that region. For that reason, the automatic failover policy cannot be enabled if the participating servers or instances are included in the Virtual Network rules. To support manual failover, follow these steps:

1. Provision the redundant copies of the front-end components of your application (web service, virtual machines etc.) in the secondary region.
2. Configure the [virtual network rules](vnet-service-endpoint-rule-overview.md) individually for primary and secondary server.
3. Enable the [front-end failover using a Traffic manager configuration](designing-cloud-solutions-for-disaster-recovery.md#scenario-1-using-two-azure-regions-for-business-continuity-with-minimal-downtime).
4. Initiate manual geo-failover when the outage is detected. This option is optimized for the applications that require consistent latency between the front-end and the data tier and supports recovery when either front end, data tier or both are impacted by the outage.

> [!NOTE]
> If you are using the **read-only listener** to load-balance a read-only workload, make sure that this workload is executed in a VM or other resource in the secondary region so it can connect to the secondary database.

### Use failover groups and firewall rules

If your business continuity plan requires failover using groups with automatic failover, you can restrict access to your database in SQL Database by using public IP firewall rules. To support automatic failover, follow these steps:

1. [Create a public IP](../../virtual-network/ip-services/virtual-network-public-ip-address.md#create-a-public-ip-address)
2. [Create a public load balancer](../../load-balancer/quickstart-load-balancer-standard-public-portal.md) and assign the public IP to it.
3. [Create a virtual network and the virtual machines](../../load-balancer/quickstart-load-balancer-standard-public-portal.md) for your front-end components.
4. [Create network security group](../../virtual-network/network-security-groups-overview.md) and configure inbound connections.
5. Ensure that the outbound connections are open to Azure SQL Database in a region by using an `Sql.<Region>` [service tag](../../virtual-network/network-security-groups-overview.md#service-tags).
6. Create a [SQL Database firewall rule](firewall-configure.md) to allow inbound traffic from the public IP address you create in step 1.

For more information on how to configure outbound access and what IP to use in the firewall rules, see [Load balancer outbound connections](../../load-balancer/load-balancer-outbound-connections.md).

The above configuration will ensure that an automatic geo-failover will not block connections from the front-end components and assumes that the application can tolerate the longer latency between the front end and the data tier.

> [!IMPORTANT]
> To guarantee business continuity during regional outages you must ensure geographic redundancy for both front-end components and databases.

## <a name="enabling-geo-replication-between-managed-instances-and-their-vnets"></a> Enabling geo-replication between managed instance virtual networks

When you set up a failover group between primary and secondary SQL Managed Instances in two different regions, each instance is isolated using an independent virtual network. To allow replication traffic between these VNets ensure these prerequisites are met:

- The two instances of SQL Managed Instance need to be in different Azure regions.
- The two instances of SQL Managed Instance need to be the same service tier, and have the same storage size.
- Your secondary instance of SQL Managed Instance must be empty (no user databases).
- The virtual networks used by the instances of SQL Managed Instance need to be connected through a [VPN Gateway](../../vpn-gateway/vpn-gateway-about-vpngateways.md) or [Express Route](../../expressroute/expressroute-howto-circuit-portal-resource-manager.md). When two virtual networks connect through an on-premises network, ensure there is no firewall rule blocking ports 5022, and 11000-11999. Global VNet Peering is supported with the limitation described in the note below.

   > [!IMPORTANT]
   > [On 9/22/2020 support for global virtual network peering for newly created virtual clusters was announced](https://azure.microsoft.com/updates/global-virtual-network-peering-support-for-azure-sql-managed-instance-now-available/). It means that global virtual network peering is supported for SQL managed instances created in empty subnets after the announcement date, as well for all the subsequent managed instances created in those subnets. For all the other SQL managed instances peering support is limited to the networks in the same region due to the [constraints of global virtual network peering](../../virtual-network/virtual-network-manage-peering.md#requirements-and-constraints). See also the relevant section of the [Azure Virtual Networks frequently asked questions](../../virtual-network/virtual-networks-faq.md#what-are-the-constraints-related-to-global-vnet-peering-and-load-balancers) article for more details. To be able to use global virtual network peering for SQL managed instances from virtual clusters created before the announcement date, consider configuring non-default [maintenance window](./maintenance-window.md) on the instances, as it will move the instances into new virtual clusters that support global virtual network peering.

- The two SQL Managed Instance VNets cannot have overlapping IP addresses.
- You need to set up your Network Security Groups (NSG) such that ports 5022 and the range 11000~12000 are open inbound and outbound for connections from the subnet of the other managed instance. This is to allow replication traffic between the instances.

   > [!IMPORTANT]
   > Misconfigured NSG security rules leads to stuck database seeding operations.

- The secondary SQL Managed Instance is configured with the correct DNS zone ID. DNS zone is a property of a SQL Managed Instance and underlying virtual cluster, and its ID is included in the host name address. The zone ID is generated as a random string when the first SQL Managed Instance is created in each VNet and the same ID is assigned to all other instances in the same subnet. Once assigned, the DNS zone cannot be modified. SQL Managed Instances included in the same failover group must share the DNS zone. You accomplish this by passing the primary instance's zone ID as the value of DnsZonePartner parameter when creating the secondary instance.

   > [!NOTE]
   > For a detailed tutorial on configuring failover groups with SQL Managed Instance, see [add a SQL Managed Instance to a failover group](../managed-instance/failover-group-add-instance-tutorial.md).

## <a name="upgrading-or-downgrading-primary-database"></a> Scale primary database

You can scale up or scale down the primary database to a different compute size (within the same service tier) without disconnecting any geo-secondaries. When scaling up, we recommend that you scale up the geo-secondary first, and then scale up the primary. When scaling down, reverse the order: scale down the primary first, and then scale down the secondary. When you scale a database to a different service tier, this recommendation is enforced.

This sequence is recommended specifically to avoid the problem where the geo-secondary at a lower SKU gets overloaded and must be re-seeded during an upgrade or downgrade process. You could also avoid the problem by making the primary read-only, at the expense of impacting all read-write workloads against the primary.

> [!NOTE]
> If you created a geo-secondary as part of the failover group configuration it is not recommended to scale down the geo-secondary. This is to ensure your data tier has sufficient capacity to process your regular workload after a geo-failover.

## <a name="preventing-the-loss-of-critical-data"></a> Prevent loss of critical data

Due to the high latency of wide area networks, geo-replication uses an asynchronous replication mechanism. Asynchronous replication makes the possibility of data loss unavoidable if the primary fails. To protect critical transactions from data loss, an application developer can call the [sp_wait_for_database_copy_sync](/sql/relational-databases/system-stored-procedures/active-geo-replication-sp-wait-for-database-copy-sync) stored procedure immediately after committing the transaction. Calling `sp_wait_for_database_copy_sync` blocks the calling thread until the last committed transaction has been transmitted and hardened in the transaction log of the secondary database. However, it does not wait for the transmitted transactions to be replayed (redone) on the secondary. `sp_wait_for_database_copy_sync` is scoped to a specific geo-replication link. Any user with the connection rights to the primary database can call this procedure.

> [!NOTE]
> `sp_wait_for_database_copy_sync` prevents data loss after geo-failover for specific transactions, but does not guarantee full synchronization for read access. The delay caused by a `sp_wait_for_database_copy_sync` procedure call can be significant and depends on the size of the not yet transmitted transaction log on the primary at the time of the call.

## Failover groups and point-in-time restore

For information about using point-in-time restore with failover groups, see [Point in Time Recovery (PITR)](recovery-using-backups.md#point-in-time-restore).

## Limitations of failover groups

Be aware of the following limitations:

- Failover groups cannot be created between two servers or instances in the same Azure region.
- Failover groups cannot be renamed. You will need to delete the group and re-create it with a different name.
- Database rename is not supported for databases in failover group. You will need to temporarily delete failover group to be able to rename a database, or remove the database from the failover group.
- System databases are not replicated to the secondary instance in a failover group. Therefore, scenarios that depend on objects from the system databases require objects to be manually created on the secondary instances and also manually kept in sync after any changes made on primary instance. The only exception is Service master Key (SMK) for SQL Managed Instance, that is replicated automatically to secondary instance during creation of failover group. Any subsequent changes of SMK on the primary instance however will not be replicated to secondary instance.
- Failover groups cannot be created between instances if any of them are in an instance pool.

## <a name="programmatically-managing-failover-groups"></a> Programmatically manage failover groups

As discussed previously, auto-failover groups can also be managed programmatically using Azure PowerShell, Azure CLI, and REST API. The following tables describe the set of commands available. Active geo-replication includes a set of Azure Resource Manager APIs for management, including the [Azure SQL Database REST API](/rest/api/sql/) and [Azure PowerShell cmdlets](/powershell/azure/). These APIs require the use of resource groups and support Azure role-based access control (Azure RBAC). For more information on how to implement access roles, see [Azure role-based access control (Azure RBAC)](../../role-based-access-control/overview.md).

### <a name="manage-sql-database-failover"></a> Manage SQL Database geo-failover

# [PowerShell](#tab/azure-powershell)

| Cmdlet | Description |
| --- | --- |
| [New-AzSqlDatabaseFailoverGroup](/powershell/module/az.sql/new-azsqldatabasefailovergroup) |This command creates a failover group and registers it on both primary and secondary servers|
| [Remove-AzSqlDatabaseFailoverGroup](/powershell/module/az.sql/remove-azsqldatabasefailovergroup) | Removes a failover group from the server |
| [Get-AzSqlDatabaseFailoverGroup](/powershell/module/az.sql/get-azsqldatabasefailovergroup) | Retrieves a failover group's configuration |
| [Set-AzSqlDatabaseFailoverGroup](/powershell/module/az.sql/set-azsqldatabasefailovergroup) |Modifies configuration of a failover group |
| [Switch-AzSqlDatabaseFailoverGroup](/powershell/module/az.sql/switch-azsqldatabasefailovergroup) | Triggers failover of a failover group to the secondary server |
| [Add-AzSqlDatabaseToFailoverGroup](/powershell/module/az.sql/add-azsqldatabasetofailovergroup)|Adds one or more databases to a failover group|

# [Azure CLI](#tab/azure-cli)

| Command | Description |
| --- | --- |
| [az sql failover-group create](/cli/azure/sql/failover-group#az_sql_failover_group_create) |This command creates a failover group and registers it on both primary and secondary servers|
| [az sql failover-group delete](/cli/azure/sql/failover-group#az_sql_failover_group_delete) | Removes a failover group from the server |
| [az sql failover-group show](/cli/azure/sql/failover-group#az_sql_failover_group_show) | Retrieves a failover group configuration |
| [az sql failover-group update](/cli/azure/sql/failover-group#az_sql_failover_group_update) |Modifies a failover group's configuration  and/or adds one or more databases to a failover group|
| [az sql failover-group set-primary](/cli/azure/sql/failover-group#az_sql_failover_group_set_primary) | Triggers failover of a failover group to the secondary server |

# [Rest API](#tab/rest-api)

| API | Description |
| --- | --- |
| [Create or Update Failover Group](/rest/api/sql/failovergroups/createorupdate) | Creates or updates a failover group |
| [Delete Failover Group](/rest/api/sql/failovergroups/delete) | Removes a failover group from the server |
| [Failover (Planned)](/rest/api/sql/failovergroups/failover) | Triggers failover from the current primary server to the secondary server with full data synchronization.|
| [Force Failover Allow Data Loss](/rest/api/sql/failovergroups/forcefailoverallowdataloss) | Triggers failover from the current primary server to the secondary server without synchronizing data. This operation may result in data loss. |
| [Get Failover Group](/rest/api/sql/failovergroups/get) | Retrieves a failover group's configuration. |
| [List Failover Groups By Server](/rest/api/sql/failovergroups/listbyserver) | Lists the failover groups on a server. |
| [Update Failover Group](/rest/api/sql/failovergroups/update) | Updates a failover group's configuration. |

---

### <a name="manage-sql-managed-instance-failover"></a> Manage SQL Managed Instance geo-failover

# [PowerShell](#tab/azure-powershell)

| Cmdlet | Description |
| --- | --- |
| [New-AzSqlDatabaseInstanceFailoverGroup](/powershell/module/az.sql/new-azsqldatabaseinstancefailovergroup) |This command creates a failover group and registers it on both primary and secondary instances|
| [Set-AzSqlDatabaseInstanceFailoverGroup](/powershell/module/az.sql/set-azsqldatabaseinstancefailovergroup) |Modifies configuration of a failover group|
| [Get-AzSqlDatabaseInstanceFailoverGroup](/powershell/module/az.sql/get-azsqldatabaseinstancefailovergroup) |Retrieves a failover group's configuration|
| [Switch-AzSqlDatabaseInstanceFailoverGroup](/powershell/module/az.sql/switch-azsqldatabaseinstancefailovergroup) |Triggers failover of a failover group to the secondary instance|
| [Remove-AzSqlDatabaseInstanceFailoverGroup](/powershell/module/az.sql/remove-azsqldatabaseinstancefailovergroup) | Removes a failover group|


# [Azure CLI](#tab/azure-cli)

| Command | Description |
| --- | --- |
| [az sql failover-group create](/cli/azure/sql/failover-group#az_sql_failover_group_create) |This command creates a failover group and registers it on both primary and secondary servers|
| [az sql failover-group delete](/cli/azure/sql/failover-group#az_sql_failover_group_delete) | Removes a failover group from the server |
| [az sql failover-group show](/cli/azure/sql/failover-group#az_sql_failover_group_show) | Retrieves a failover group configuration |
| [az sql failover-group update](/cli/azure/sql/failover-group#az_sql_failover_group_update) |Modifies a failover group's configuration  and/or adds one or more databases to a failover group|
| [az sql failover-group set-primary](/cli/azure/sql/failover-group#az_sql_failover_group_set_primary) | Triggers failover of a failover group to the secondary server |

# [Rest API](#tab/rest-api)

| API | Description |
| --- | --- |
| [Create or Update Failover Group](/rest/api/sql/instancefailovergroups/createorupdate) | Creates or updates a failover group's configuration |
| [Delete Failover Group](/rest/api/sql/instancefailovergroups/delete) | Removes a failover group from the instance |
| [Failover (Planned)](/rest/api/sql/instancefailovergroups/failover) | Triggers failover from the current primary instance to this instance with full data synchronization. |
| [Force Failover Allow Data Loss](/rest/api/sql/instancefailovergroups/forcefailoverallowdataloss) | Triggers failover from the current primary instance to the secondary instance without synchronizing data. This operation may result in data loss. |
| [Get Failover Group](/rest/api/sql/instancefailovergroups/get) | retrieves a failover group's configuration. |
| [List Failover Groups - List By Location](/rest/api/sql/instancefailovergroups/listbylocation) | Lists the failover groups in a location. |

---

## Next steps

- For detailed tutorials, see
  - [Add SQL Database to a failover group](failover-group-add-single-database-tutorial.md)
  - [Add an elastic pool to a failover group](failover-group-add-elastic-pool-tutorial.md)
  - [Add a SQL Managed Instance to a failover group](../managed-instance/failover-group-add-instance-tutorial.md)
- For sample scripts, see:
  - [Use PowerShell to configure active geo-replication for Azure SQL Database](scripts/setup-geodr-and-failover-database-powershell.md)
  - [Use PowerShell to configure active geo-replication for a pooled database in Azure SQL Database](scripts/setup-geodr-and-failover-elastic-pool-powershell.md)
  - [Use PowerShell to add an Azure SQL Database to a failover group](scripts/add-database-to-failover-group-powershell.md)
- For a business continuity overview and scenarios, see [Business continuity overview](business-continuity-high-availability-disaster-recover-hadr-overview.md)
- To learn about Azure SQL Database automated backups, see [SQL Database automated backups](automated-backups-overview.md).
- To learn about using automated backups for recovery, see [Restore a database from the service-initiated backups](recovery-using-backups.md).
- To learn about authentication requirements for a new primary server and database, see [SQL Database security after disaster recovery](active-geo-replication-security-configure.md).
