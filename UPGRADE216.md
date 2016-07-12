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
* Strict RFC 2818 host name checking in SRM/HTTP clients - this in particular affects server side srmCopy and WebDAV COPY. It is expected that this will break with server side copy to/from some WLCG sites. Globus Toolkit enables the same strict mode by default by the end of the year, so we expect the affected sites to get new host certificates soon. Additionally, server side copy is not used much, so it is likely not be a big issue for dCache.
* Significant schema changes in Chimera. Upon upgrade the schema will be updated, which may take a while for large sites. To downgrade, the schema changes have to be rolled back with the appropriate command in dCache before downgrading dCache.
* PnfsManager now uses a single request queue. This could possibly affected third-party monitoring scripts that scrape the info output.
* All forbidden and obsolete properties in 2.13 have been removed and all deprecated properties in 2.13 have been marked obsolete.
* RC4 ciphers are banned by default and the option to ban Diffie Hellman key exchange if broken has been removed.
* gPlazma now instantiates plugins separately for every use in gplazma.conf. This means that the configuration has to be specified for all lines, not just the first. This provides the flexibility to use the same plugin several times with different configurations.
* The WebDAV door enables BASIC authentication by default over HTTPS.
* Pools always compute checksums on the fly and the ontransfer checksum policy has been dropped.
* The SRM by default uses a fair share scheduler for scheduling transfers. Thus the observed behaviour under load may have changed.
* SRM 1 support is deprecated and disabled by default.
* SRM scheduler states have been simplified. Third party monitoring scripts that track request states may have to be updated.
* SRM no longer retries SRM requests internally. Failures are propagated to the client to provide fail-fast behaviour.
* SRM no longer resolves host names in SURLs when determining whether they are local to this dCache instance or not. In particular, this may affect sites that use DNS aliases.
* The SRM database schema is now managed by liquibase and the schema is updated upon upgrade. To downgrade, the schema changes have to be rolled back with the appropriate command in dCache before downgrading dCache. As a consequence, user information for existing SRM requests will be lost from the database. Third party scripts that access the SRM database directly may have to be updated.

#### Incompatibilities introducted in 2.15
* Some configuration properties have been deprecated or made obsolete. dcache check-config can tell you if you are affected.
* Several Chimera schema changes are applied upon upgrade. To downgrade those schema changes have to be rolled back prior to downgrade.
* Several bugs have been fixed. In the unlikely event that you relied on these bugs, upgrading may break your installation.
* X.509 proxy certificate source address restrictions are now enforced. Clients that relied on these not being enforced will fail after upgrade.

#### Incompatibilities introducted in 2.16
* Apache ZooKeeper is now a required dependency.
* Several database schemas will be updated upon upgrade. This includes billing and pinmanager. Upon downgrade, these schemas must be rolled back using the dcache database rollbackToDate command prior to downgrading dCache. Third party code that uses these database will possibly have to be modified.
* The dCache location manager can no longer be configured to establish arbitrary topologies.
* The srm service was split into two services - a frontend service called srm and a backend service called srmmanager. Both are required to operate an SRM service in dCache.
* Support for the SRM 1 protocol has been dropped.
* The FTP door’s default root path behavior now matches that of other doors. That is, it no longer uses the per user root setting as the root directory. The original behavior can be restored by appropriate configuration.
* The RemoteTransferManager and CopyManager cells of the transfermanagers service have been merged.
* Sites with a custom httpd.conf configuration will have to adjust this upon upgrade to inject the cell address of the info service.

## Services

### Removed Services

## What's new

### Admin Changes

### Databases

#### Schema Management

#### Schema Changes

## Upgrade procedure

## Frequently Asked Questions

