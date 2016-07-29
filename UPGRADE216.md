# The ultimate golden release upgrade guide

Community effort to write an upgrade guide for dCache 2.13

## How to get from dCache 2.13 to dCache 2.16

### About this document

The dCache golden release upgrade guides aim to make migrating between golden releases easier by compiling all important information into a single document. The document is based on the release notes of versions 2.13, 2.14, 2.15 and 2.16. In contrast to the release notes, the guide ignores minor changes and is structured more along the lines of functionality groups rather than individual services.

Although the guide aims to collect all information required to upgrade from dCache 2.13 to 2.16, there is information in the release notes not found in this guide. The reader is advised to at least have a brief look at the release notes found at dCache.org.

There are plenty of changes to configuration properties, default values, log output, admin shell commands, admin shell output, etc. If you have scripts that generate dCache configuration, issue shell commands, or parse the output of shell commands or log files, then you should expect that these scripts need to be adjusted. This document does not list all those changes.

### About this Release

dCache 2.16 is the seventh golden (long term support) release of dCache. The release will be supported at least until summer 2018. This is the recommended release for the second LHC run.

As always, the amount of changes between this and the previous golden release is staggering. We have tried to make the upgrade process as smooth as possible, while at the same time moving the product forward. This means that we try to make dCache backwards compatible with the previous golden release, but not at the expense of changes that we believe improve dCache in the long run.

The most visible change when upgrading is that most configuration properties have been renamed, and some have changed default values. Many services have been renamed, split, or merged. These changes require work when upgrading, even though most of them are cosmetic - they improve the consistency of dCache and thus ease the configuration, but those changes do not fundamentally affect how dCache works.

dCache pools from version 2.13 can be mixed with services from dCache 2.16. This enables a staged roll-out in which non-pool nodes are upgraded to 2.16 first, followed by pools later. It is however recommended to upgrade to the latest version of 2.13 before performing a staged roll-out.

Upgrading directly from versions prior to 2.13 is not officially supported. If you run versions before 2.13, we recommend upgrading to 2.13 first. That is not to say that upgrading directly to 2.16 is not possible, but we have not explicitly tested nor documented such an upgrade path.

Downgrading after upgrade is not possible without manual intervention: In particular database schemas have to be rolled back before downgrade. If pools use the Berkeley DB meta data back-end, these need to be converted to the file back-end before downgrading and then back to the Berkeley DB back-end after downgrade.

## Executive summary

### Highlights from dCache release 2.13

#### Highlights introduced on dCache 2.14
* Significant performance and storage space improvements in Chimera.
* Almost all certificiate handling code has been ported to the EMI CANL library rather than JGlobus. Provides OCSP support and EUGRIDPMA name space support.
* Concurrent request processing in gPlazma.
* Pool group patterns in admin shell.
* Improved UberFTP support.
* WebDAV door can report file locality.
* Fair share scheduling for SRM transfers.
* SRM database managed by liquibase.
* Configurable kXR_Qconfig support for xrootd.

#### Highlights introduced on dCache 2.15
* Improved glob patterns.
* More robust handling of over load situations.
* fsck for Chimera on upgrade.
* Reduced storage requirements for Chimera.
* Chimera can store multiple checksum for a file.
* Admin shell documentation updates.
* gPlazma restriction attributes.
* Set ACLs through DCAP.
* Pool startup improvements.

#### Highlights introduced on dCache 2.16
* New experimental RESTful API.
* New experimental end user web interface called dCacheView.
* New experimental support for redundant cell communication using multiple message brokers.
* New resilience service for managing file replicas. Will eventually replace the replica manager.
* New experimental support for replicable services.
* Scalable SRM frontend service.
* Limited OpenID Connect support.

### Incompatibilities

#### Incompatibilities introducted in 2.14
* `Strict RFC 2818` host name checking in SRM/HTTP clients - this in particular affects server side srmCopy and WebDAV COPY. It is expected that this will break with server side copy to/from some WLCG sites. Globus Toolkit enables the same strict mode by default by the end of the year, so we expect the affected sites to get new host certificates soon. Additionally, server side copy is not used much, so it is likely not be a big issue for dCache.
* Significant schema changes in Chimera. Upon upgrade the schema will be updated, which may take a while for large sites. To downgrade, the schema changes have to be rolled back with the appropriate command in dCache *before* downgrading dCache.
* `PnfsManager` now uses a single request queue. This could possibly affected third-party monitoring scripts that scrape the info output.
* All forbidden and obsolete properties in 2.13 have been removed and all deprecated properties in 2.13 have been marked obsolete.
* RC4 ciphers are banned by default and the option to ban Diffie Hellman key exchange if broken has been removed.
* gPlazma now instantiates plugins separately for every use in `gplazma.conf`. This means that the configuration has to be specified for all lines, not just the first. This provides the flexibility to use the same plugin several times with different configurations.
* The WebDAV door enables `BASIC` authentication by default over HTTPS.
* Pools always compute checksums on the fly and the `ontransfer` checksum policy has been dropped.
* The SRM by default uses a fair share scheduler for scheduling transfers. Thus the observed behaviour under load may have changed.
* SRM 1 support is deprecated and disabled by default.
* SRM scheduler states have been simplified. Third party monitoring scripts that track request states may have to be updated.
* SRM no longer retries SRM requests internally. Failures are propagated to the client to provide fail-fast behaviour.
* SRM no longer resolves host names in SURLs when determining whether they are local to this dCache instance or not. In particular, this may affect sites that use DNS aliases.
* The SRM database schema is now managed by liquibase and the schema is updated upon upgrade. To downgrade, the schema changes have to be rolled back with the appropriate command in dCache *before* downgrading dCache. As a consequence, user information for existing SRM requests will be lost from the database. Third party scripts that access the SRM database directly may have to be updated.

#### Incompatibilities introducted in 2.15
* Some configuration properties have been deprecated or made obsolete. `dcache check-config` can tell you if you are affected.
* Several Chimera schema changes are applied upon upgrade. To downgrade those schema changes have to be rolled back prior to downgrade.
* Several bugs have been fixed. In the unlikely event that you relied on these bugs, upgrading may break your installation.
* X.509 proxy certificate source address restrictions are now enforced. Clients that relied on these not being enforced will fail after upgrade.

#### Incompatibilities introducted in 2.16
* *Apache ZooKeeper* is now a required dependency.
* Several database schemas will be updated upon upgrade. This includes `billing` and `pinmanager`. Upon downgrade, these schemas must be rolled back using the `dcache database rollbackToDate` command prior to downgrading dCache. Third party code that uses these database will possibly have to be modified.
* The dCache location manager can no longer be configured to establish arbitrary topologies.
* The `srm` service was split into two services - a frontend service called `srm` and a backend service called `srmmanager`. Both are required to operate an SRM service in dCache.
* Support for the ***SRMv1*** protocol has been dropped.
* The FTP door’s default root path behavior now matches that of other doors. That is, it no longer uses the per user root setting as the root directory. The original behavior can be restored by appropriate configuration.
* The `RemoteTransferManager` and `CopyManager` cells of the transfermanagers service have been merged.
* Sites with a custom `httpd.conf` configuration will have to adjust this upon upgrade to inject the cell address of the info service.

## Services
* This section describes renamed, collapsed, split, and removed services. A summary of the changes is contained in the following table.

| Service                 | Configuration / Remark
|------------------------:|:--------------------------------------------------------------------------------------------------------|
| **srm**                   | SRM has been split: this is the *frontend* service. ***SRMv1*** has been dropped.                     |
| **srmmanager**            | SRM has been split: this is the *backend* service                                                     |
| **zookeeper**             | New service for locating services and leader election. May run as `standalone` service out of dCache. |
| **resilience**            | Replaces the **replica** service.                                                                     |
| **transfermanagers**      | The `RemoteTransferManager` and `CopyManager` cells have been merged.                                 |
| **gplazma**               | Added a new generic `multimap` plugin.                                                                |
| **info**                  | No longer hardcoded in **httpd**.                                                                     |
| **pnfsmanager**           | Chimera has pluggable RDBMS drivers for PostgreSQL 9.5.                                               |

## What's new
### Introduced Apache ZooKeeper as an external coordination service
Apache ZooKeeper is a naming, coordination, and synchronization services. Starting with the release of dCache 2.16, Apache ZooKeeper is a required dependency to run dCache. It is used by several components in dCache – e.g. for locating dCache domains, locating services, and leader election. Future versions are most likely increasing reliance on Apache ZooKeeper.

To make migration of existing deployments easy, we support using an embedded ZooKeeper, i.e. the ZooKeeper server runs inside a regular dCache service in a dCache domain. It is configured through dCache’s own configuration system. For easy upgrade, add the following as the first domain in the layout file of the host with `dCacheDomain`:

> [zookeeperDomain]
> [zookeeperDomain/zookeeper]

By placing this domain on the host running `dCacheDomain`, other domains in dCache will be able to locate the ZooKeeper instance without configuration changes. Eventually you should configure `dcache.zookeeper.connection` rather than `dcache.broker.host` to allow dCache domains to locate ZooKeeper.

If using the embedded ZooKeeper service, we recommend running `zookeeper` in its own dCache domain to avoid runtime dependencies between the ZooKeeper server and the client inside dCache connecting to it. In particular the low level cells system itself relies on ZooKeeper, presenting problems during startup and shutdown of the domain hosting `zookeeper`. Such problems are minimized by running the service in its own domain, but error messages in the log files during startup and shutdown are inevitable.

For improved stability, we recommend installing a stand-alone ZooKeeper independently from dCache. Many Linux distributions have packaged ZooKeeper already.

ZooKeeper can be deployment in a redundant setup, typically on three or five servers, allowing respectively one or two instances to fail without loosing the service.

We have placed further information on using ZooKeeper with dCache at *https://github.com/dCache/dcache/wiki/ZooKeeper*.

### Add user information to billing database
If using the `billing` database backend, the `billinginfo` and `doorinfo` tables are extended with additional columns to represent information about the user performing the transfer. This includes the owner string, the primary FQAN, the UID and primary GID.

Upon downgrading to an earlier release, the schema must be rolled back using the `dcache database rollbackToDate` command. Doing so will erase the data stored in the new columns.

### Storage class was added when listing movers
The output of the `mover ls`, `st ls` and `rh ls` commands have been extended to show the storage class of the file being transferred, flushed, or restored.

### Obsoleted dcache.authn.gsi.crl.cache.lifetime
dCache no longer uses JGlobus and thus this property is no longer used.

### New resilience service
The `resilience` service replaces the `replica` service. The `replica` service is still included in dCache 2.16, but we encourage sites that rely on it to migrate to the `resilience` service. The `replica` service will eventually be removed from dCache.

In contrast to the `replica` service, `resilience` offers many benefits, such as not relying on its own database and thus not suffering from state synchronization issues, more flexible configuration in terms of which files are to be replicated, not relying on dedicated pools, more robust operation, works with any retention policy, detecting and recovering broken replicas, allowing temporary suppression of replication on particular pools, integration with the `alarm` service, and more.

Separate documentation for the `resilience` service will be posted.

### Overhaul of cell routing topology
dCache cell communication used to have a complex configuration system allowing arbitrary topologies of dCache domains to be formed. A hidden service called the location manager was embedded in all dCache domains and coordinated how connections between domains were established. Inside `dCacheDomain` the location manager deployed a simple UDP service that other domains used to discover how they should interconnect. To our knowledge all sites used the default star topology in which the `dCacheDomain` was the central messaging hub for all other domains.

In dCache 2.16 this has been completely reimplemented. The UDP based configuration discovery system previously used by location manager has been replaced by Apache ZooKeeper. The UDP based service is still supported in a legacy mode to allow rolling upgrades from earlier versions. dCache 2.17 will not support the legacy mode.

The location manager service no longer supports arbitrary communication topologies to be formed. Domains are configured individually to be either core domains or satellite domains, with the former being messaging hubs. The default is for `dCacheDomain` to be the only core domain. This maintains the previous default behavior.

The major new feature is the ability to have multiple core domains, i.e. multiple message hubs, with all other domains connecting to all core domains. This enables a redundant communication topology in which a core domain can fail and the remaining services can maintain connectivity through the remaining core domains.

We have published documentation about the dCache message passing system at https://github.com/dCache/dcache/wiki/Message-passing. This also includes information on redundant topologies.

The **System** cell embedded into every dCache domain provides the new `traceroute` command which allows the route between two cells to be discovered.

### dCache Service Names
Traditionally central services in dCache have had well-known names like `PnfsManager` and `PoolManager`. Other services – and the operator through the `admin` service – could communicate with those services using the short well-known name rather than the longer fully qualified cell address.

Starting with dCache 2.16 such services have a configurable service name instead. The service name typically coincides with the cell name, but they are no longer linked.

E.g. the `pnfsmanager` service defaults to a cell name `PnfsManager` (defined by the `pnfsmanager.cell.name` property), which when instantiated in a domain called `namespaceDomain` may be addressed as `PnfsManager@namespaceDomain`. The `pnfsmanager` service now also has the new `pnfsmanager.cell.service` property, which happens to default to `PnfsManager` too, allowing the cell to be addressed with the shorter `PnfsManager` name, just like before.

The difference is that these are no longer coupled. The cell name and the service name may be changed independently. This change applies to all central services in dCache, i.e. most services except pools and doors.

The `cell.export` properties are all marked deprecated. Affected services have the new `cell.service` property.

More information about service names may be found at *https://github.com/dCache/dcache/wiki/Message-passing*.

### Replicable services
Under the hood, the changes related to the new service names described above is much bigger than what may be apparent from the simple description. Several services may now be replicated. In such a setup, each instance of the service shares the same service name. This was the motivation for separating cell names from service names.

In dCache 2.16 not all services are replicable. Those that are define a `cell.replicable` property set to `true`, e.g. `pnfsmanager.cell.replicable`. The `dcache services` command shows whether services are replicable or not.

This feature is considered experimental. Documentation will be published at *https://github.com/dCache/dcache/wiki/Replicable-services*.

### Split of SRM into a frontend and a backend
The SRM service has been split into a frontend service called `srm` and a backend service called `srmmanager`. The frontend listens to the SRM TCP port and is the endpoint clients communicate with. The backend does all the request processing, including scheduling and persistence in the SRM database.

This change allows multiple frontends to be deployed in a load balanced setup while using a single `srmmanager` backend. Furthermore the `srmmanager` backend may be replicated for improved fault tolerance, scalability and rolling upgrades, but this is currently considered experimental.

Upon upgrading, the simplest course of action is to add an `srmmanager` service in the same domain as the existing `srm` service. It is recommended to place `srmmanager` in from of the `srm` service to ensure the backend starts before the frontend.

Note that many configuration properties previously defined for the `srm` service now need to be defined for `srmmanager` instead. Such properties have been renamed accordingly. Use `dcache check-config` to validate your configuration prior to starting dCache.

Also note that if the export point of the namespace is changed using `srmmanager.root` (formerly `srm.root`), then a similar change must be applied to `srm.loginbroker.root`. Both default to the new `dcache.root` property, so in most deployments it may be easier to change the export point for doors by setting that property instead.

Upon upgrading, it is important that the scheduler ID of the `srmmanager` matches the original cell name of the `srm` service. This is because entries in the database are tagged with this ID and using a different value means that old SRM requests are not cleaned upon restart. The default scheduler ID matches the old default cell name of the `srm` service and thus if deploying `srm` and `srmmanager` on the same host and if the original service did no use a custom cell name, no additional configuration changes are needed on upgrade.

More documentation will be published at https://github.com/dCache/dcache/wiki/Replicable-services.

### Chimera auto-discovers the correct database driver
Chimera used to have the `chimera.db.dialect` property to select the database driver. The correct driver is now selected automatically and `chimera.db.dialect` and similar service specific properties are now deprecated.

### New `frontend` service
The new `frontend` service offers a RESTful webapi and a modern user facing web interface. The service is currently fairly limited, but we expect it to be aggressively extended over the next feature releases. It currently offers a RESTful API to navigate the name space, and a browser interface to navigate this name space. The regular `webdav` service is used as a backend for file transfers.

More documentation on this service will be published soon.

### New property to define name space export point
Many doors now respect the new dcache.root property to define the directory in the name space that is exposed to users. If defaults to the chimera root directory. Note that this also applies to FTP doors. Thus by default, FTP doors no longer use the per account root directory defined in gPlazma. To restore the old behavior, set `ftp.root` to the empty value.

### gPlazma LDAP plugin now supports authenticating LDAP servers
The gplazma LDAP plugin can now authenticate with the LDAP server using the new `gplazma.ldap.auth`, `gplazma.ldap.binddn`, and `gplazma.ldap.bindpw` configuration properties.

### Add user information to active transfer pages of httpd
The various active transfer pages now show the UID, primary GID and primary FQAN of the user doing a transfer.

### JSON output for active transfers page
The classic `httpd` web pages provides JSON output for ongoing transfers. This complements the classic txt output. The information can be accessed with something like `curl -s http://localhost:2288/context/transfers.json`.

### Pin manager database backend was rewritten
The `pinmanager` database backend was rewritten to no longer use the DataNucleus ORM. Minor schema changes are applied during upgrade. Upon downgrade these schema changes must be rolled back using the `dcache database rollbackToDate` command.

### The use of `Infinity` as a pool size is deprecated
Pools could be configured with `Infinity` as a size to indicate that the size should be derived automatically from available disk space. This value is now deprecated and the `pool.size` should be left empty instead for automatically sized pools.

### Improved `poolmanager` configuration reload
Several race conditions have been fixed in the reload command of the `poolmanager` service. The command now also preserves the already cached pool state, thus reducing the side effects of reloading the `poolmanager` configuration.

### Improved publishing of pool state
The `poolmanager` service provides information about pools to several other services. This has been refactored to use publish-subscribe messaging rather than have each service pool `poolmanager` for updates. This change also allows `poolmanager` to publish updates on pool status changes immediately, i.e. pools going offline or coming back online are immediately propagated to e.g. `spacemanager` and `resilience`.

### `/var/lib/dcache` is owned by the `dcache` user
When using the Debian package, `/var/lib/dcache` was owned by user `dcache` all along. This is now also the case for the RPM package.

### ***SRMv1*** support has been dropped
The dCache `srm` service no longer supports version 1 of the SRM protocol.

### The internal cells of the `transfermanagers` service have been merged
The `RemoteTransferManager` and `CopyManager` cells have been merged. Several properties related to the `CopyManager` cell have been marked obsolete. Some of the runtime configuration commands of these cells have been dropped.

### Added caching of gPlazma mappings in `webdav` service
The WebDAV service now caches gPlazma mappings. See `webdav.service.gplazma.cache.size` and `webdav.service.gplazma.cache.timeout` for the new configuration properties.

### Improved pool queue plots in `webadmin` service
The pool queue plots in the `webadmin` service have been restructured and are now easier to use.

### Limited *OpenID Connect* support
An *OpenID Connect* plugin (`oidc`) for gPlazma was added as well as limited support in the `webdav` service to accept *OpenID Connect* bearer tokens. Separate documentation on how to make use of this will be published later.

### Generic gPlazma mapping plugin
A new generic mapping plugin (`multimap`) was added to gPlazma. It allows mappings from any principal to multiple other principals. Currently only `dn`, `oidc`, `username`, `uid`, `gid`, `kerberos`, `email` principals are supported. An example configuration file looks like:
```
"dn:/C=DE/O=Hamburg/OU=desy.de/CN=Kermit The Frog" username:kermit
oidc:googleoidcsubject gid:1000,true uid:1000
"dn:/C=DE/O=Hamburg/OU=desy.de/CN=Kermit The Frog" uid:1000
"dn:/C=DE/O=Hamburg/OU=desy.de/CN=Kermit The Frog" uid:1000 gid:500,true gid:100
```

### Cell address of the `info` service is no longer hard-coded in `httpd`
The new `httpd.service.info` property allows the cell address of the `info` service to be configured in the `httpd` service. Sites with custom `httpd.conf` definitions will have to update the configuration such that the info alias looks something like

> set alias info class org.dcache.services.info.InfoHttpEngine -- -cell=${httpd.service.info}

### NFS is better
Better spec compliance, faster.

### Admin Changes

### Databases

#### Schema Management

#### Schema Changes

## Upgrade procedure

## Frequently Asked Questions

