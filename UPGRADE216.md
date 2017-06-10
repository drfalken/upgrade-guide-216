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
* CANL/OCSP caches the queried result for a while, but the OCSP server might be a single point of failure that will receive a lot of requests. Some CAs advertises OCSP servers in the certificates they issue without actually supporting them. 

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
### New changes introduced on 2.14
#### Improved execution time statistics
Several cells provide information on request execution times. The code has been improved to reduce the effect of numerical instabilities. This improves the accuracy of the collected statistics. Rather than the population standard deviation we now report the sample standard deviation.

#### Use CANL for certificate verification
CANL is a library provided by EMI that provides support for OpenSSL style trust stores, PEM encoded certificates, proxy certificates, certificate path verification, CRL and OCSP validation, etc. In the past, dCache has relied upon the JGlobus library for such operations, but JGlobus has not seen much maintenance lately. We have switched to using CANL instead as we feel it is more up to date (an important aspect for security critical libraries), provides a cleaner API, has more features, and doesn’t lock us into old versions of other third party libraries. There may also be performance benefits - we know for certain that the startup time for client tools is improved, and time will tell if the server side is faster too.

The amount of changes required in dCache to implement this was quite large and touched most components, although the externally visible number of changes should be low. Still, with this amount of changes there is bound to be a regression or two.

One feature of JGlobus missing in CANL is an implementation of the GSI protocol. GSI is essentially TLS with an additional credential delegation step after the initial TLS handshake. In dCache 2.13 we introduced our own GSI implementation in dCache to provide support for non-blocking IO in the SRM door. In 2.14 this has been adopted by all doors providing GSI support, i.e. SRM, FTP and DCAP. Furthermore, the client side of GSI has been implemented and is used by our FTP and SRM clients, as well as our GridSite delegation shell.

As part of porting DCAP and FTP to our own GSI implementation, the SSL context is now cached between sessions, greatly reducing the per-connection overhead of these doors. Consequently these doors now expose settings for key pair cache lifetime:
```
#  ---- Delegation key pair reuse lifetime
#
#  X.509 clients may delegate credentials to dCache using GSI or the dedicated GridSite
#  delegation service. The delegation works by the server generating a certificate
#  signing request which the client signs. Since generating a key pair as part of the
#  signing request is expensive, dCache caches and reuses the key pair.
#
#  This setting controls for how long a key pair is reused. A compromised key could be used
#  for new signing requests for this amount of time. If the reuse time is too short, the
#  overhead of generating new key pairs increases.
#
dcache.authn.gsi.delegation.cache.lifetime = 30000
dcache.authn.gsi.delegation.cache.lifetime.unit = MILLISECONDS
```
The default host and CA certificate refresh periods have been significantly reduced to 60 seconds:
```
# ---- Host certificate refresh period
#
# This option influences in which intervals the host certificate will be
# reloaded on a running door.
#
dcache.authn.hostcert.refresh = 60
dcache.authn.hostcert.refresh.unit = SECONDS

# ---- CA certificates refresh period
#
# Grid-based authentication usually requires to load a set of
# certificates that are accepted as certificate authorities. This
# option influences in which interval these trust anchors are
# reloaded.
#
dcache.authn.capath.refresh = 60
dcache.authn.capath.refresh.unit = SECONDS
``` 
Since CANL provides configurable name space verification, CRL validation, and OCSP validation, all doors and pools relying on TLS now respect the following new configuration properties:
```
# ---- Certificate Authority Namespace usage mode
#
# A CA namespace restricts the certificates dCache accepts as issued by a given CA. The namespace
# is defined in .namespaces (EURGIDPMA) or .signing_policy (GLOBUS) files alongside the CA certificate.
#
# This setting controls the rules governing the verification of these namespaces. The following
# documentation is copied from the EMI CANL library used by dCache.
#
#       GLOBUS_EUGRIDPMA
#
#          A Globus EACL is checked first. If found for the issuing CA then it is used and enforced.
#             If not found then EuGridPMA namespaces definition is searched. If found for the issuing CA
#                then it is enforced.
#                   If no definition is present then namespaces check is considered to be passed.
#
#       EUGRIDPMA_GLOBUS
#
#          An EuGridPMA namespaces definition is checked first. If found for the issuing CA then it is enforced.
#             If not found then Globus EACL definition is searched. If found for the issuing CA
#                then it is enforced.
#                   If no definition is present then namespaces check is considered to be passed.
#
#       GLOBUS
#
#          A Globus EACL is checked only. If found for the issuing CA then it is used and enforced.
#             If no definition is present then namespaces check is considered to be passed.
#
#       EUGRIDPMA
#
#          An EuGridPMA namespaces definition is checked only. If found for the issuing CA then it is enforced.
#             If no definition is present then namespaces check is considered to be passed.
#
#       GLOBUS_EUGRIDPMA_REQUIRE
#
#          A Globus EACL is checked first. If found for the issuing CA then it is used and enforced.
#             If not found then EuGridPMA namespaces definition is searched. If found for the issuing CA
#                then it is enforced.
#                   If no definition is present then namespaces check is considered to be failed.
#
#       EUGRIDPMA_GLOBUS_REQUIRE
#
#          An EuGridPMA namespaces definition is checked first. If found for the issuing CA then it is enforced.
#             If not found then Globus EACL definition is searched. If found for the issuing CA
#                then it is enforced.
#                   If no definition is present then namespaces check is considered to be failed.
#
#       GLOBUS_REQUIRE
#
#          A Globus EACL is checked only. If found for the issuing CA then it is used and enforced.
#             If no definition is present then namespaces check is considered to be failed.
#
#       EUGRIDPMA_REQUIRE
#
#          An EuGridPMA namespaces definition is checked only. If found for the issuing CA then it is enforced.
#             If no definition is present then namespaces check is considered to be failed.
#
#       EUGRIDPMA_AND_GLOBUS
#
#          Both EuGridPMA namespaces definition and Globus EACL are enforced for the issuer.
#             If no definition is present then namespaces check is considered to be passed.
#
#       EUGRIDPMA_AND_GLOBUS_REQUIRE
#
#          Both EuGridPMA namespaces definition and Globus EACL are enforced for the issuer.
#             If no definition is present then namespaces check is considered to be failed.
#
#       IGNORE
#
#          CA namespaces are fully ignored, even if present.
#
dcache.authn.namespace-mode=EUGRIDPMA_AND_GLOBUS_REQUIRE

# ---- Certificate Revocation List usage mode
#
# CAs regularly publish certificate revocation lists (CRLs) containing the serial id of
# certificates that have been revoked. Such certificates should not be accepted by dCache.
#
# Such CRLs are stored in .r? files alongside the CA certificate. It is outside the scope
# of dCache to refresh the CRLs.
#
# This setting controls how dCache makes use of such CRLs. The following documentation is copied
# from the EMI CANL library used by dCache.
#
#       REQUIRE
#
#          A CRL for CA which issued a certificate being validated
#             must be present and valid and the certificate must not be on the list.
#
#       IF_VALID
#
#          If a CRL for CA which issued a certificate being validated
#             is present and valid then the certificate must not be listed on the CRL.
#                If the CRL is present but it is outdated (or anyhow else corrupted) then the validation fails.
#                   If CRL is missing then validation is successful.
#
#       IGNORE
#
#          CRL is not checked even if it exists.
#
dcache.authn.crl-mode=REQUIRE

# ---- On-line Certificate Status Protocol usage mode
#
# On-line Certificate Status Protocol (OCSP) is an alternative to CRLs in which
# dCache consults an external service to check the validity of a particular certificate.
#
# This setting controls how dCache makes use of this protocol. The following documentation is copied
# from the EMI CANL library used by dCache.
#
#       REQUIRE
#
#          Require, for each checked certificate, that at least one valid OCSP responder is defined and
#             that at least one responder of those defined returns a correct certificate status.
#                If all OCSP responders return error or unknown status, the last one received is treated as a
#                   critical validation error.
#                      Not suggested, unless it is guaranteed that well configured responder(s) is(are) defined
#                         and can handle all queries without timeouts.
#
#       IF_AVAILABLE
#
#          Use OCSP for each certificate if a responder is available. OCSP 'unknown' status and
#             query errors (as timeout) do not cause the validation to fail.
#                Also a lack of defined responder doesn't cause the validation to fail.
#
#       IGNORE
#
#          Do not use OCSP.
#
dcache.authn.ocsp-mode=IF_AVAILABLE
```
Finally, logging from CANL is different. Our initial impression is also that it is significantly better.

#### FTP Client
Another area in which dCache relied on the JGlobus client was for the FTP client. An FTP client is needed as part of our SRM client as well as for SRM and WebDAV third party copy. In the quest to drop the JGlobus dependency, we have forked the JGlobus FTP client and ported it to CANL. There should be minimal user visible changes from this. The most interesting may be that our client now identifies itself to the server as being dCache and using the dCache version number.

#### SRM client
As mentioned above, the SRM client has been updated to use our own GSI implementation on top of CANL for certificate handling. On a host with a large number of CA certificates, the startup time is significantly reduced.

The SRM protocol is implemented using SOAP over HTTP over GSI. Due to the old SOAP version used we are bound to the Axis 1 library. In the past we relied upon the minimalistic HTTP client integrated into Axis 1. In dCache 2.14 we added an Axis handler to use the Apache HTTP components client. The primary benefits are:
* Proper keep-alive support, meaning that the client no longer needs to reestablish the connection to the server for every low level SRM request. This both reduces the latency experienced by the client and reduces load on the server.
* Strict *RFC 2818* host name verification.

In partiular the latter point has the potential to break compatibility with some sites. This is the case if the host certificate used by the site does not comply with RFC 2818. Without strict RFC 2818 support, TLS/GSI connections are susceptible to man in the middle attacks. The Globus Toolkit (the C library, not the JGlobus Java version) will switch to strict RFC 2818 support by default at the end of 2015 and we expect that most sites will quickly request certificates with correct subject alternative names.

If compatibility with non-compliant sites is required, we recommend using a version of the SRM client prior to 2.14. On the server side, strict RFC 2818 host name verification is only used for server side srmCopy or WEBDAV copy. If these are needed with non-compilant sites, we suggest using dCache 2.13.

#### Updated help pages for many admin shell commands
dCache has for several years been in a transition between two infrastructures for providing admin shell commands. The newer of these provides much better help pages. In dCache 2.14 many commands have been ported from old to the new scheme.

#### Admin shell gains pool group globs
The new admin shell introduced in dCache 2.13 provides bulk commands with cell name globs. In dCache 2.14 pool group globs have been added to these commands. A pool group glob follows the pattern `pool/poolgroup`, with either side accepting `?` and `*` wildcards. It expands to all pools that match the left side of the pattern and which are within a pool group matching the right side. Either side may be empty and is equivalent to `*`.

Thus `/` will match any pool that appears in any pool group, `/atlas_disk` matches any pool in `atlas_disk`, and `*_rack1/atlas_buffer` matches any pool with a name ending in `_rack1` in the `atlas_buffer` pool group. These could for instance be used to perform bulk commands over a set of pool:
```
\s /atlas_tape flush set interval 600
\s /atlas_tape save
```

#### Significant schema changes in Chimera
The Chimera database schema has been updated to improve throughput, reduce latency and reduce disk space. It will be updated automatically the first time the `PnfsManager` is started, but is is advisable to apply the schema changes ahead of time by using the `dcache database update` command. It is also advisable to make a backup of the database *prior* to upgrading as well as run a test migration on a clone of the database to know how long it will take as well discover any incompatibilities with local schema modification.

Once upgraded to dCache 2.14, one cannot downgrade to 2.13 without rolling back the schema changes. This must be done *before* downgrading using the `dcache database rollbackToDate` command.

Several schema changes are applied durin upgrade:
* The `.` and `..` directory entries are no longer stored in the database. Since Chimera is stored in a relational database, it can find parent directories without these backpointers. Where applicable, these directories entries are created on the fly in dCache.
* Directory tags are no longer created by a trigger. In the past the trigger reacted upon the insertion of the `..` entry, but since we no longer insert this entry we cannot rely on this trigger. Furthermore, for the temporary directories created for every SRM upload the tags should not be copied and the trigger needlessly slowed down creating such directories.
* Inodes are now created using the new `f_create_inode` stored procedure. This eliminates several round trips between the client and PostgreSQL.
* The type of the `itagid` column of the `t_tags_inodes` table has been changed to a 64 bit auto sequence field. This reduces storage space as well as CPU load for these tables.
* The `t_access_latency` and `t_retention_policy` tables have been merged into the `t_inodes` table. The former two tables are dropped. This reduces storage space as well as latency in creating and reading inodes.
* The address mask is removed from the `t_acl`. Chimera ACLs no longer support access control by client IP.

#### Chimera PostgreSQL 9.5 driver
Chimera has pluggable RDBMS drivers. dCache 2.14 adds a driver specific for PostgreSQL 9.5 and newer. This driver avoids the excessive logging caused by primary key uniqueness violations that are common with Chimera.

To activate, set `chimera.db.dialect` to `PgSQL95`.

#### Chimera race conditions and performance improvements
Many rare races and subtle bugs have been fixed, and lots of small performance improvements have been made.

#### PnfsManager uses single request queue
Due to the extensive changes in Chimera, PnfsManager can now use a single request queue shared between all threads. Thus the `info` output no longer shows a queue per thread and it is easier to keep all threads busy.

Technically, there is a queue per thread-group, but since thread-groups are not currently used by Chimera, for all intends and purposes there is only one general request queue.

#### Forbidden and obsolete properties have been dropped
Properties marked as forbidden or obsolete in 2.13 have been removed from dCache 2.14. Properties marked deprecated in 2.13 have been marked obsolete. Please verify and fix any warnings produced by `dcache check-config` before upgrading.

#### Cryptographic cipher selection
dCache now bans RC4 ciphers by default. The ability to disable Diffie Hellman key exchange on JVMs in which this is broken has been removed since Diffie Hellman now works on all supported JVMs.

#### FTP gains new SITE commands
The FTP door now supports the `SITE CHGRP`, `SITE SYMLINKFROM` and `SITE SYMLINKTO` non-standard commands to change the file group and create symbolic links. These commands are supported by the UberFTP client.

#### FTP proxy limits internal network interface
FTP doors may act as proxies for the data connection. In this case the pool establishes a TCP connection to the door. Previously the door would listen to the wildcard address (i.e. all interfaces) even though only one specific address was sent to the pool to connect to. In dCache 2.14 the door now limits the server socket to this one address.

#### gPlazma drops support for plugin caching
gPlazma used to only instantiate a plugin once even if it was mentioned several times in gplazma.conf. This meant that only a single configuration could be applied, thus making several usecases impossible to handle.

dCache now instantiates every plugin every time it is referenced in gplazma.conf and each instance can have its own configuration. The options for forcing enabling the old behaviour have been removed.

#### gPlazma supports concurrent request processing
In the past gPlazma was single threaded, which in particular was a problem with plugins that called out to external services. gPlazma is now multi-threaded and will happily call several plugins concurrently. Even the same plugin may be called concurrently, which means that all plugins - including those by third-parties - must be thread safe.

#### gPlazma XACML plugin update for improved spec compliance
The XACML plugin has been ported to CANL too and in the process it was updated to improved compliance with the specification. The client library used by the plugin unfortunately depends on JGlobus and thus the current version of dCache still ships with JGlobus.

#### DCAP delegates X.509 and VOMS proxy processing to gPlazma
Previously DCAP would validate the VOMS signatures of a proxy in the door, thus requiring a vomsdir to be set up on the door. gPlazma now delegates this to gPlazma and the gPlazma voms plugin. Thus it is no longer necessary to keep a vomsdir setup on GSIDCAP doors.

#### NFS door publish exported paths login broker information
The NFS door now include the exported paths as part of the login broker information. This information is available to the info service and the srm door.

#### NFS door sees large number of fixes and improvements
As always, the NFS door has received a fair share of fixes and improvements. If you rely heavily on the NFS door, using the latest version of dCache is recommended.

#### Pool manager generates sorted configuration files
Previously entries where output in hash order, which meant that minor changes in configuration could cause drastic changes to the order.

#### WebDAV door links to https version of dCache home page
Thanks to a contribution by Christoph Anton Mitterer, the footer on WebDAV door directory listings now point to the https version of the dCache home page.

#### WebDAV door can report file locality
SRM has the concept of file locality, i.e. whether a file is on disk, on tape, both, offline or lost. The WebDAV door now exposes this information too:
* The HTML rendering of directory listings uses the icon to indicate the file locality.
* For programatic access, the WebDAV PROPFIND method.

An example of the latter follows:
```
$ curl -X PROPFIND -H Depth:0 http://localhost:2880/disk/test-1447231167-1 \
  --data '<?xml version="1.0" encoding="utf-8"?>
          <D:propfind xmlns:D="DAV:">
              <D:prop xmlns:R="http://www.dcache.org/2013/webdav"
                      xmlns:S="http://srm.lbl.gov/StorageResourceManager">
                  <R:Checksums/>
                  <S:AccessLatency/>
                  <S:RetentionPolicy/><S:FileLocality/>
              </D:prop>
          </D:propfind>'

<?xml version="1.0" encoding="utf-8" ?>
<d:multistatus xmlns:cal="urn:ietf:params:xml:ns:caldav"
               xmlns:cs="http://calendarserver.org/ns/"
               xmlns:card="urn:ietf:params:xml:ns:carddav"
               xmlns:ns2="http://www.dcache.org/2013/webdav"
               xmlns:ns1="http://srm.lbl.gov/StorageResourceManager"
               xmlns:d="DAV:">
    <d:response>
        <d:href>/disk/test-1447231167-1</d:href>
        <d:propstat>
            <d:prop>
                <ns1:FileLocality>ONLINE</ns1:FileLocality>
                <ns1:RetentionPolicy>REPLICA</ns1:RetentionPolicy>
                <ns2:Checksums>adler32=3b8711d6</ns2:Checksums>
                <ns1:AccessLatency>ONLINE</ns1:AccessLatency>
            </d:prop>
            <d:status>HTTP/1.1 200 OK</d:status>
        </d:propstat>
    </d:response>
</d:multistatus>
```

#### WebDAV door provides human-friendly file sizes
An abbreviated file size is shown as a hint in the WebDAV door HTML rendering. To see, however the mouse over the file entry.

#### WebDAV door enables `BASIC` authentication by default over https connections
This provides a means for clients without a certificate to authenticate with the server. Note that `BASIC` authentication should not be used over unencrypted connections.

#### Pools now always compute checksums on the fly
Recently we changes the default checksum policy for new pools from `onwrite` to `ontransfer`. The latter requires less resources as it computes the checksum during the upload rather than reading the file back from disk after the transfer.

In dCache 2.14 the `ontransfer` checksum calculation is no longer optional - it is always computed for streaming uploads. The `onwrite` policy can still be enabled, but it would calculate the checksum a second time. A future update of dCache will remove the `onwrite` policy.

#### SRM database schema changes
The SRM database is now managed by the liquibase schema management library. An existing schema from a previous installation is automatically detected, but we recommend doing a test upgrade on a clone of the database to check compatibility with any local schema modifications.

Several changes are made to the schema of the SRM database:
* Missing indexes are added.
* Unused indexes are removed.
* The `count` column of the `lsrequests` table is renamed to `cnt` to avoid conflicts with the like named SQL keyword.
* The encoding of user information is redone entirely and user information is now stored in a binary format.

In particular the latter change is worth paying attention to. Due to this change, existing requsts will fail during upgrade as the existing user information is deleted upon upgrade. Already finished requests will be kept in the database until they are garbage collected, but they too loose all information about who created it. Furthermore, the binary encoding in the new user record means that it is no longer easy to access this information in local queries. This change was necessary to work around limitations in the previous serialization - in short, we cannot serialize X.500 names in string form without loosing information.

#### SRM gains a pluggable transfer strategy mechanism
Plugins may decide in which order to serve transfer requests by clients. Two strategies are shipped with dCache with the default implementing a simple fair share strategy:
```
# ---- Request transfer strategy
#
# srmPrepareToPut and srmPrepareToGet requests enter the READY state when the TURL is handed
# out to the client. If the request is processed asynchronously (client is polling for the result),
# the request stays in the RQUEUED state until the client queries the result.
#
# A transfer strategy plugin has the ability to prevent the transition from the RQUEUED to READY
# state. A plugin may allow the number of concurrent transfers to be limited, or may provide
# some fair-share between parties.
#
# Two plugins ship with dCache:
#
#     first-come-first-served
#
#         The maximum number of requests in the READY state is limited by
#         srm.request.*.max-transfers, but otherwise TURLs are handed out to
#         whomever comes first.
#
#     fair-share
#
#         The maximum number of requests in the READY state is limited by
#         srm.request.*.max-transfers, but slots are kept free such that each party with
#         requests in RQUEUED will receive its fair share.
#
#           A configurable discriminator defines the grouping between which to provide a
#           fair share.
#
srm.plugins.transfer = fair-share

# Discriminator for the fair share transfer strategy.
srm.transfer.fair-share!discriminator = ${srm.plugins.discriminator}
```
#### Deprecation of SRM 1 support
The SRM 1 protocol has long been superseeded by the SRM 2.2 protocol. dCache 2.14 adds options to disable SRM 1 support and disables it by default:
```
#
# SRM versions to support.
#
# Comma separated list of SRM versions to support.
#
(any-of?1|2)srm.version=2
```
You can reenable SRM 1 support by adding it to the above configuration option. Please contact us if you do so we can ascertain when to drop SRM 1 entirely.

This change also removes support for third party srmCopy to/from SRM 1 servers, no matter whether SRM 1 support in the server is enabled or not.

#### SRM request scheduler state simplification
SRM scheduler states have been simplified by collapsing several states into the new `InProgress` state. Besides changing the output of the `info` command, this will change the state values observed in the database. Sites that monitor these states will have to update their monitoring scripts. The job state mappings can be viewed here: *https://github.com/dCache/dcache/blob/master/modules/srm-server/src/main/resources/org/dcache/srm/request/sql/srmjobstate–214.csv*

#### SRM no longer retries requests
SRM used to have support to retry requests internally upon errors. The implementation of this functionality was problematic since
* many errors were not retried even when they should, and worse
* many errors that should not be retried were.

In the interest of fail fast behaviour and simpler code, support for internally retrying requests has been dropped. Failures are propagated to the client and it is up to the client to determine whether and how to retry. This also allows client to more quickly fall back to other sites in case of problems.

#### SRM changes algorithm to detect a local SURL
The SRM protocol has a concept of site URLs and classifies these into those local to the storage system and those belonging to other storage systems. This classification is important when the SRM is to determine which file is local and which is remote in a third party SRM copy operation, or just to determine if the SURL is indeed local in put or get operations.

The most important changes are:
* srmCopy is now limited to having a local source or local destination SURL. It no longer allows a transfer between a local non-SRM door and a remote system.
* no DNS lookup is performed in determining whether the SURL is local. Previously the host name would be resolved to an IP addressed and the IP addressed would be compared to local addresses. Now the list of local host names is determined during startup and the host name in the SURL is compared to the local names directly. An administrator still has control over which names to consider local by setting the `srm.net.local-hosts` property.

#### Several minor SRM fixes
Several corner cases and race conditions have been fixed, as well as better failure handling to prevent leaking pins or upload directories.

#### Configurable kXR_Qconfig replies for xrootd
The xrootd protocol has a mechanism to query the server configuration using a kXR_Qconfig request. dCache provided a few hard-coded properties, but the xrootd protocol specifies an open list of properties.

In dCache 2.14 the admin may inject additional properties to be returned to the client. These are configured through the dCache configuration system:
```
#  ----- Custom kXR_Qconfig responses
#
#   xrootd clients may query the server configuration using a kXR_Qconfig request.
#   These key value pairs can be queried. Additional key value pairs may be added as
#   needed, see the xrootd protocol specification at http://xrootd.org/ for details.
#
(prefix)xrootd.query-config = kXR_Qconfig responses
xrootd.query-config!version = dCache ${dcache.version}
xrootd.query-config!sitename = ${dcache.description}
xrootd.query-config!role = none

#  ----- Custom kXR_Qconfig responses
#
#   xrootd clients may query the server configuration using a kXR_Qconfig request.
#   These key value pairs can be queried. Additional key value pairs may be added as
#   needed, see the xrootd protocol specification at http://xrootd.org/ for details.
#
(prefix)pool.mover.xrootd.query-config = kXR_Qconfig responses
pool.mover.xrootd.query-config!version = dCache ${dcache.version}
pool.mover.xrootd.query-config!sitename = ${dcache.description}
pool.mover.xrootd.query-config!role = none
```

#### Many third party libraries have been updated
Most notably the new version of the PostgreSQL JDBC driver has some nice performance improvements.

### New changes introduced on 2.15
#### Glob patterns gain alternation lists
Adds support for {} alternation lists in many places were globs are supported. I.e. `1{foo,bar}2` matches both `1foo2` and `1bar2`. Alternation lists can be nested. Among others these are used for matching pool names in pool manager, admin shell and migration module, patterns for file name matching in directory listings, and in gPlazma.

#### Cell communication gets fail fast path on overload
dCache services communicate through message passing. For request-reply style interactions messages have a time to live field. If messages are delivered late due to overload, messages are discarded to avoid inducing even more load on the receiving service. When this happens the requesting service will wait for the reply until the request times out and thus an overloaded system will give the appearance to hang.

This release adds a fail fast path in the receiving service in which it immediately generates a timeout reply if the expected queuing time exceeds the time to live of the message. Thus with this release services may generate timeout responses quickly when another service approaches an overload situation.

#### Cell message thread pool configuration is harmonized
dCache services communicate through message passing. Messages are delivered to a service from a pool of threads. For some services, the number of threads and the maximum message queue length are configurable. Keeping with dCache tradition, the configuration properties were different for different services. Trying to establish a new tradition, these properties have now been harmonized.

The following are the relevant configuration properties (copied verbatim from the default property files).

```
#  ---- Maximum number of concurrent requests to process.
#
#  The number of login requests that gPlazma will process
#  concurrently.  Setting this number too high may result in large
#  spikes of CPU activity and the potential to run out of memory.
#  Setting the number too lower results in potentially slow login
#  activity.
#
(deprecated)gplazma.cell.limits.threads=30
gplazma.cell.max-message-threads=${gplazma.cell.limits.threads}

#  ---- Maximum number of requests to queue.
#
#  The number of login requests that gPlazma will queue before
#  rejecting requests. Unlimited if left empty.
#
gplazma.cell.max-messages-queued=


#
# NFS door message processing thread pool configuration
#
(deprecated)nfs.cell.limits.message.threads.max=8
(deprecated)nfs.cell.limits.message.queue.max=1000
nfs.cell.max-message-threads=${nfs.cell.limits.message.threads.max}
nfs.cell.max-messages-queued=${nfs.cell.limits.message.queue.max}

#  ---- Cell message processing parameters
#
#  Settings for the request processing thread pool.
#
#  The thread pool will stay at the minimum number of threads until the
#  maximum request queue length has been reached. At that point the number
#  of threads is increased up to the maximum, after which further requests
#  will be dropped. Idle threads are terminated after the idle time until
#  the number of threads drops back to the minimum.
#
#  Except for database operations, the pin manager operates in an
#  asynchronous fashion. The minimum number of threads should be chosen
#  such that the database can be saturated under load. If network latency
#  between the pin manager and the database is high, then the minimum
#  number of threads could be increased to hide this latency. The default
#  setting is likely fine for most cases.
#
#  The maximum number of threads should be below the database connection
#  limit - otherwise threads end up blocking for idle connections and may
#  potentially deadlock in case the same thread needs more than one
#  connection (e.g. for nested transactions).
(obsolete)pinmanager.cell.threads.min=
(obsolete)pinmanager.cell.threads.max-idle-time=
(obsolete)pinmanager.cell.threads.max-idle-time.unit=

(deprecated)pinmanager.cell.threads.max=45
(deprecated)pinmanager.cell.queue.max=10000

pinmanager.cell.max-message-threads=${pinmanager.cell.threads.max}
pinmanager.cell.max-messages-queued=${pinmanager.cell.queue.max}

# Cell message processing limits
(obsolete)pool.cell.limits.message.threads.min=
(obsolete)pool.cell.limits.message.threads.max-idle-time=
(obsolete)pool.cell.limits.message.threads.max-idle-time.unit=

(deprecated)pool.cell.limits.message.threads.max=50
(deprecated)pool.cell.limits.message.queue.max=1000

pool.cell.max-message-threads=${pool.cell.limits.message.threads.max}
pool.cell.max-messages-queued=${pool.cell.limits.message.queue.max}

# Cell message processing limits
(deprecated)srm.cell.limits.message.threads.max = 10
(deprecated)srm.cell.limits.message.queue.max = 100

srm.cell.max-message-threads = ${srm.cell.limits.message.threads.max}
srm.cell.max-messages-queued = ${srm.cell.limits.message.queue.max}
```

#### Several obsolete admin shell commands are dropped
The following deprecated admin shell commands have been removed or marked obsolete: `load cellprinter`, `load interpreter`, `set classloader`, and `show classloader`. If you used these (you didn’t) then stop doing so.

#### Chimera gets new compact primary key

The Chimera database schema has been heavily updated. The schema has traditionally used the variable length and relatively long PNFS ID as a primary and foreign key. In this release a 64 bit inumber is introduced to replace the PNFS ID as a key in the schema. The PNFS ID is still an essential concept to dCache as it is used as a persistent identifier for files on pools, however in Chimera it is just another file attribute.

Upon upgrade all Chimera tables will be rewritten. One has to expect that for large instances the schema update will take many hours. We recommend testing the schema migration on a clone of the database prior to deploying dCache 2.15. The schema can be rolled back to the previous structure using the `dcache database rollbackToDate` command, but this must be done before downgrading dCache.

Custom Chimera queries and views will likely have to be updated. If custom views or triggers have been installed, we recommend extra attention to testing the migration prior to upgrade to ensure that the custom modifications do not cause the upgrade to fail.

The new schema uses less disk space which means more data can be cached in available memory on the RDBMS server. Depending on hardware, this may provide a significant performance improvement. On the other hand, hardware with plenty of RAM and SSDs may only observe a minor improvement.

#### Chimera can store several checksums for a file
Chimera can now store several checksums of different types for a file.

#### Chimera performs consistency check upon upgrade
Previous dCache releases have fixed several bugs in Chimera. These bugs may however have introduced inconsistencies such as wrong link counts or unlinked files.

Upon upgrade, the Chimera schema migration will fix these inconsistencies. It will create a `lost+found` directory in the root of the name space and link unreachable files and directories from here. After upgrade the content should be inspected and either deleted or moved out of the `lost+found` directory. Running this consistency check takes significant time and is one of the reasons why the Chimera schema upgrade in this release is time consuming. The check cannot be triggered manually. The inconsistencies observed were caused by bugs that should be fixed now. If they are not, that’s a bug.

#### Admin shell commands gain better documentation

Many admin shell commands have been updated with additional documentation. Most commands should be functionally unchanged. Affected services are `cleaner`, `dir`, `pool`, `pnfsmanager`, and `poolmanager`.

#### gPlazma gains new feature to restrict file access
gPlazma plugins now have access to a new powerful abstraction called a restriction. A restriction is invoked upon any name space operation and the restriction can block the access. dCache itself currently uses restrictions to implement read-only accounts, but third party session plugins can generate custom restrictions as part of a client login.

Developers should consult the JavaDoc of the [Restriction class] (https://github.com/dCache/dcache/blob/master/modules/common/src/main/java/org/dcache/auth/attributes/Restriction.java) and contact the dCache team.

#### DCAP can accept ACLs on file or directory creation
At **dCache.org** we love standards, but DCAP is still around and utilized by many users. Some of those users like to use ACLs too, and this release adds support for specifying ACLs when creating files or directories. Programmatically this can be done like so:
```
dc_setExtraOpotion("-acl=\"<ace1> ... <aceN>\"");
```
From the command line you would do something like:
```
dccp -X-acl="\"A::3750:rwatTnNcCoy A::EVERYONE@:rtncy\"" .....
```
The first one to decipher that ACE without consulting the documentation gets a free license!

#### *FTP AUTH* return codes are now *RFC 2228* compliant
Return codes for error results of the AUTH command have been changed to be more in compliance with RFC 2228. In the unlikely event that your client relied on dCache not being compliant with RFC 2228, your client is now broken.

#### gPlazma produces better diagnostic information
Proxy certificates include an embedded attribute certificate when they are a voms proxy. This attribute certificate contains group-membership information and is signed by the VOMS server’s credential.

An `.lsc` file associates a trusted VOMS server with some already trusted certificate authority. This requires that the CA that signed the VOMS server’s certificate matches the DN in the `.lsc` file, otherwise the VOMS server is not trusted and no FQANs are extracted. Therefore, when diagnosing a problem where the voms plugin fails to extract any FQANs it is sometimes useful to know which CA signed the VOMS server’s certificate.

This release extends the output logged upon failure to login and by the gPlazma `explain login` command to show the attribute certificate extensions. One VOMS-specific extension includes the certificate chain of the VOMS server. With this release, the various issuers of this chain are shown.

Additionally, the output of the DN of the VOMS certificate is updated to *RFC 2253* format. Yay standards!

#### gPlazma authdb can be sufficient
gPlazma uses a PAM style structure with plugins being marked `optional`, `sufficient`, `required` or `prerequisite`. `sufficient` indicates that if a plugin succeeds then that is sufficient (a very descriptive name, it seems) to complete the current phase. For this to work it requires that a plugin can fail too - otherwise it would always be sufficient.

`authzdb` was a plugin that would not fail in the session phase if no session information was found - now it does.

#### Pool `rep ls` command gains filter on storage info
The `rep ls` admin shell command lists the files on a pool. It now accepts the `-storage` option to filter by storage class and thus list only a subset of the files. The option even accepts a glob pattern.

#### Pool startup has been improved
The pool startup logic has been heavily refactored in this release. A pool will now immediately switch to read-only mode when it starts. Like before, the pool will remain read-only until a full inventory has been build and any inconsistencies have been addressed.

Lock contention during pool initialization has been reduced. In particular this avoids a period of unresponsiveness just prior to switching to read-write mode.

#### NFS becomes more robust against mover timeouts
If the NFS door times out while waiting for a write mover to be activated, it would retry the creation and erroneously create a read mover instead. It no longer does this.

#### ANONYMOUS and AUTHENTICATED ACEs gain consistent semantics
ANONYMOUS and AUTHENTICATED ACEs in name space ACLs in previous releases was inconsistent and erroneous and in some situations both ACEs were applied simultaneously despite RFC 3530 stating that they are mutually exclusive. The interpretation is now no longer erroneous nor inconsistent.

#### SRM `info` command provides cache statistics
The SRM maintains several internal caches. Statistics about the awesome effectiveness of these caches are now included in the output of the `info` admin shell command:
```
--- storage (dCache plugin for SRM) ---
Custom reverse DNS lookup cache: CacheStats{hitCount=0, missCount=0, loadSuccessCount=0, loadExceptionCount=0, totalLoadTime=0, evictionCount=0}
Space token by owner cache: CacheStats{hitCount=0, missCount=0, loadSuccessCount=0, loadExceptionCount=0, totalLoadTime=0, evictionCount=0}
Space by token cache: CacheStats{hitCount=0, missCount=0, loadSuccessCount=0, loadExceptionCount=0, totalLoadTime=0, evictionCount=0}
```

#### Admin shell gets configurable history length
The admin shell can store a persistent command history. This release increases the default and adds a configuration property for how many commands to store. Gone are the days of forgetting that brilliant one-liner.
```
#  ---- Admin door history size
#
#   The size of the history in lines and thereby the history can be
#   limited by setting admin.history.size to an integer value. The
#   standard size is 500.
#
admin.history.size = 500
```

#### gPlazma produces better log messages for VOMS failures
FQANs in a VOMS proxy are only trusted if signed by a VOMS server for which an `.lsc` file is present. Other FQANs are silently dropped (you don’t want to reject a user just because he signed up with an unknown VOMS server). In previous releases it was really hard to distinguish the situation in which a user didn’t have any VOMS extensions in the certificate or if they had all been rejected due to validation failures. It was also really hard to know why validation failed. This is now really easy as gPlazma logs this information to the log file.

#### WebDAV logs `X-Forwarded-For` header to reveal clients behind proxies
The WebDAV door produces an access log detailing who has been using it and how. Among the information logged is the client IP address, however if a user cowardly hides behind a proxy only the proxy’s IP address was logged. Now the WebDAV door logs the client IP address as reported by the proxy, too. Obviously, this information is only as trustworthy as the proxy.

#### Enforce source address in proxy certificates
An X.509 proxy certificate is a certificate signed by an end user certificate or another proxy certificate. Whoever got the proxy certificate can act upon the original end user’s behalf. The proxy can however be limited in various ways. One of these is by adding a policy element that restricts from which IP addresses the certificate can be used. Whoever got the proxy can still do bad things, but at least we limit them to doing it from a particular node. In the past dCache ignored this optional policy - now it no longer does.

#### gPlazma xacml plugin uses updated XACML client library
The `xacml` gPlazma plugin integrates dCache with GUMS serves. The plugin relies on a third party XACML client library. In this release we upgraded that library to a new version with awesome new features. Among other things it no longer relies on JGlobus, which means dCache 2.15 is the first release in decades without JGlobus (yay!). It also understands HTTP keep alive now, so we don’t have to reconnect to the GUMS server all the time.

To not just make this about hidden internal things, we obsoleted two configuration properties and introduced a new one:
```
#  ---- Path to the vomsdir directory
gplazma.xacml.vomsdir=${dcache.authn.vomsdir}
(obsolete)gplazma.xacml.vomsdir.dir = Use gplazma.xacml.vomsdir
(obsolete)gplazma.xacml.vomsdir.ca = Use gplazma.xacml.ca
```

### New changes introduced on 2.16
#### Introduced Apache ZooKeeper as an external coordination service
Apache ZooKeeper is a naming, coordination, and synchronization services. Starting with the release of dCache 2.16, Apache ZooKeeper is a required dependency to run dCache. It is used by several components in dCache – e.g. for locating dCache domains, locating services, and leader election. Future versions are most likely increasing reliance on Apache ZooKeeper.

To make migration of existing deployments easy, we support using an embedded ZooKeeper, i.e. the ZooKeeper server runs inside a regular dCache service in a dCache domain. It is configured through dCache’s own configuration system. For easy upgrade, add the following as the first domain in the layout file of the host with `dCacheDomain`:
```
[zookeeperDomain]
[zookeeperDomain/zookeeper]
```
By placing this domain on the host running `dCacheDomain`, other domains in dCache will be able to locate the ZooKeeper instance without configuration changes. Eventually you should configure `dcache.zookeeper.connection` rather than `dcache.broker.host` to allow dCache domains to locate ZooKeeper.

If using the embedded ZooKeeper service, we recommend running `zookeeper` in its own dCache domain to avoid runtime dependencies between the ZooKeeper server and the client inside dCache connecting to it. In particular the low level cells system itself relies on ZooKeeper, presenting problems during startup and shutdown of the domain hosting `zookeeper`. Such problems are minimized by running the service in its own domain, but error messages in the log files during startup and shutdown are inevitable.

For improved stability, we recommend installing a stand-alone ZooKeeper independently from dCache. Many Linux distributions have packaged ZooKeeper already.

ZooKeeper can be deployment in a redundant setup, typically on three or five servers, allowing respectively one or two instances to fail without loosing the service.

We have placed further information on using ZooKeeper with dCache at *https://github.com/dCache/dcache/wiki/ZooKeeper*.

#### Add user information to billing database
If using the `billing` database backend, the `billinginfo` and `doorinfo` tables are extended with additional columns to represent information about the user performing the transfer. This includes the owner string, the primary FQAN, the UID and primary GID.

Upon downgrading to an earlier release, the schema must be rolled back using the `dcache database rollbackToDate` command. Doing so will erase the data stored in the new columns.

#### Storage class was added when listing movers
The output of the `mover ls`, `st ls` and `rh ls` commands have been extended to show the storage class of the file being transferred, flushed, or restored.

#### Obsoleted dcache.authn.gsi.crl.cache.lifetime
dCache no longer uses JGlobus and thus this property is no longer used.

#### New resilience service
The `resilience` service replaces the `replica` service. The `replica` service is still included in dCache 2.16, but we encourage sites that rely on it to migrate to the `resilience` service. The `replica` service will eventually be removed from dCache.

In contrast to the `replica` service, `resilience` offers many benefits, such as not relying on its own database and thus not suffering from state synchronization issues, more flexible configuration in terms of which files are to be replicated, not relying on dedicated pools, more robust operation, works with any retention policy, detecting and recovering broken replicas, allowing temporary suppression of replication on particular pools, integration with the `alarm` service, and more.

Separate documentation for the `resilience` service will be posted.

#### Overhaul of cell routing topology
dCache cell communication used to have a complex configuration system allowing arbitrary topologies of dCache domains to be formed. A hidden service called the location manager was embedded in all dCache domains and coordinated how connections between domains were established. Inside `dCacheDomain` the location manager deployed a simple UDP service that other domains used to discover how they should interconnect. To our knowledge all sites used the default star topology in which the `dCacheDomain` was the central messaging hub for all other domains.

In dCache 2.16 this has been completely reimplemented. The UDP based configuration discovery system previously used by location manager has been replaced by Apache ZooKeeper. The UDP based service is still supported in a legacy mode to allow rolling upgrades from earlier versions. dCache 2.17 will not support the legacy mode.

The location manager service no longer supports arbitrary communication topologies to be formed. Domains are configured individually to be either core domains or satellite domains, with the former being messaging hubs. The default is for `dCacheDomain` to be the only core domain. This maintains the previous default behavior.

The major new feature is the ability to have multiple core domains, i.e. multiple message hubs, with all other domains connecting to all core domains. This enables a redundant communication topology in which a core domain can fail and the remaining services can maintain connectivity through the remaining core domains.

We have published documentation about the dCache message passing system at https://github.com/dCache/dcache/wiki/Message-passing. This also includes information on redundant topologies.

The **System** cell embedded into every dCache domain provides the new `traceroute` command which allows the route between two cells to be discovered.

#### dCache Service Names
Traditionally central services in dCache have had well-known names like `PnfsManager` and `PoolManager`. Other services – and the operator through the `admin` service – could communicate with those services using the short well-known name rather than the longer fully qualified cell address.

Starting with dCache 2.16 such services have a configurable service name instead. The service name typically coincides with the cell name, but they are no longer linked.

E.g. the `pnfsmanager` service defaults to a cell name `PnfsManager` (defined by the `pnfsmanager.cell.name` property), which when instantiated in a domain called `namespaceDomain` may be addressed as `PnfsManager@namespaceDomain`. The `pnfsmanager` service now also has the new `pnfsmanager.cell.service` property, which happens to default to `PnfsManager` too, allowing the cell to be addressed with the shorter `PnfsManager` name, just like before.

The difference is that these are no longer coupled. The cell name and the service name may be changed independently. This change applies to all central services in dCache, i.e. most services except pools and doors.

The `cell.export` properties are all marked deprecated. Affected services have the new `cell.service` property.

More information about service names may be found at *https://github.com/dCache/dcache/wiki/Message-passing*.

#### Replicable services
Under the hood, the changes related to the new service names described above is much bigger than what may be apparent from the simple description. Several services may now be replicated. In such a setup, each instance of the service shares the same service name. This was the motivation for separating cell names from service names.

In dCache 2.16 not all services are replicable. Those that are define a `cell.replicable` property set to `true`, e.g. `pnfsmanager.cell.replicable`. The `dcache services` command shows whether services are replicable or not.

This feature is considered experimental. Documentation will be published at *https://github.com/dCache/dcache/wiki/Replicable-services*.

#### Split of SRM into a frontend and a backend
The SRM service has been split into a frontend service called `srm` and a backend service called `srmmanager`. The frontend listens to the SRM TCP port and is the endpoint clients communicate with. The backend does all the request processing, including scheduling and persistence in the SRM database.

This change allows multiple frontends to be deployed in a load balanced setup while using a single `srmmanager` backend. Furthermore the `srmmanager` backend may be replicated for improved fault tolerance, scalability and rolling upgrades, but this is currently considered experimental.

Upon upgrading, the simplest course of action is to add an `srmmanager` service in the same domain as the existing `srm` service. It is recommended to place `srmmanager` in from of the `srm` service to ensure the backend starts before the frontend.

Note that many configuration properties previously defined for the `srm` service now need to be defined for `srmmanager` instead. Such properties have been renamed accordingly. Use `dcache check-config` to validate your configuration prior to starting dCache.

Also note that if the export point of the namespace is changed using `srmmanager.root` (formerly `srm.root`), then a similar change must be applied to `srm.loginbroker.root`. Both default to the new `dcache.root` property, so in most deployments it may be easier to change the export point for doors by setting that property instead.

Upon upgrading, it is important that the scheduler ID of the `srmmanager` matches the original cell name of the `srm` service. This is because entries in the database are tagged with this ID and using a different value means that old SRM requests are not cleaned upon restart. The default scheduler ID matches the old default cell name of the `srm` service and thus if deploying `srm` and `srmmanager` on the same host and if the original service did no use a custom cell name, no additional configuration changes are needed on upgrade.

More documentation will be published at https://github.com/dCache/dcache/wiki/Replicable-services.

#### Chimera auto-discovers the correct database driver
Chimera used to have the `chimera.db.dialect` property to select the database driver. The correct driver is now selected automatically and `chimera.db.dialect` and similar service specific properties are now deprecated.

#### New `frontend` service
The new `frontend` service offers a RESTful webapi and a modern user facing web interface. The service is currently fairly limited, but we expect it to be aggressively extended over the next feature releases. It currently offers a RESTful API to navigate the name space, and a browser interface to navigate this name space. The regular `webdav` service is used as a backend for file transfers.

More documentation on this service will be published soon.

#### New property to define name space export point
Many doors now respect the new dcache.root property to define the directory in the name space that is exposed to users. If defaults to the chimera root directory. Note that this also applies to FTP doors. Thus by default, FTP doors no longer use the per account root directory defined in gPlazma. To restore the old behavior, set `ftp.root` to the empty value.

#### gPlazma LDAP plugin now supports authenticating LDAP servers
The gplazma LDAP plugin can now authenticate with the LDAP server using the new `gplazma.ldap.auth`, `gplazma.ldap.binddn`, and `gplazma.ldap.bindpw` configuration properties.

#### Add user information to active transfer pages of httpd
The various active transfer pages now show the UID, primary GID and primary FQAN of the user doing a transfer.

#### JSON output for active transfers page
The classic `httpd` web pages provides JSON output for ongoing transfers. This complements the classic txt output. The information can be accessed with something like `curl -s http://localhost:2288/context/transfers.json`.

#### Pin manager database backend was rewritten
The `pinmanager` database backend was rewritten to no longer use the DataNucleus ORM. Minor schema changes are applied during upgrade. Upon downgrade these schema changes must be rolled back using the `dcache database rollbackToDate` command.

#### The use of `Infinity` as a pool size is deprecated
Pools could be configured with `Infinity` as a size to indicate that the size should be derived automatically from available disk space. This value is now deprecated and the `pool.size` should be left empty instead for automatically sized pools.

#### Improved `poolmanager` configuration reload
Several race conditions have been fixed in the reload command of the `poolmanager` service. The command now also preserves the already cached pool state, thus reducing the side effects of reloading the `poolmanager` configuration.

#### Improved publishing of pool state
The `poolmanager` service provides information about pools to several other services. This has been refactored to use publish-subscribe messaging rather than have each service pool `poolmanager` for updates. This change also allows `poolmanager` to publish updates on pool status changes immediately, i.e. pools going offline or coming back online are immediately propagated to e.g. `spacemanager` and `resilience`.

#### `/var/lib/dcache` is owned by the `dcache` user
When using the Debian package, `/var/lib/dcache` was owned by user `dcache` all along. This is now also the case for the RPM package.

#### ***SRMv1*** support has been dropped
The dCache `srm` service no longer supports version 1 of the SRM protocol.

#### The internal cells of the `transfermanagers` service have been merged
The `RemoteTransferManager` and `CopyManager` cells have been merged. Several properties related to the `CopyManager` cell have been marked obsolete. Some of the runtime configuration commands of these cells have been dropped.

#### Added caching of gPlazma mappings in `webdav` service
The WebDAV service now caches gPlazma mappings. See `webdav.service.gplazma.cache.size` and `webdav.service.gplazma.cache.timeout` for the new configuration properties.

#### Improved pool queue plots in `webadmin` service
The pool queue plots in the `webadmin` service have been restructured and are now easier to use.

#### Limited *OpenID Connect* support
An *OpenID Connect* plugin (`oidc`) for gPlazma was added as well as limited support in the `webdav` service to accept *OpenID Connect* bearer tokens. Separate documentation on how to make use of this will be published later.

#### Generic gPlazma mapping plugin
A new generic mapping plugin (`multimap`) was added to gPlazma. It allows mappings from any principal to multiple other principals. Currently only `dn`, `oidc`, `username`, `uid`, `gid`, `kerberos`, `email` principals are supported. An example configuration file looks like:
```
"dn:/C=DE/O=Hamburg/OU=desy.de/CN=Kermit The Frog" username:kermit
oidc:googleoidcsubject gid:1000,true uid:1000
"dn:/C=DE/O=Hamburg/OU=desy.de/CN=Kermit The Frog" uid:1000
"dn:/C=DE/O=Hamburg/OU=desy.de/CN=Kermit The Frog" uid:1000 gid:500,true gid:100
```
#### Cell address of the `info` service is no longer hard-coded in `httpd`
The new `httpd.service.info` property allows the cell address of the `info` service to be configured in the `httpd` service. Sites with custom `httpd.conf` definitions will have to update the configuration such that the info alias looks something like
```
set alias info class org.dcache.services.info.InfoHttpEngine -- -cell=${httpd.service.info}
```
#### NFS is better
Better spec compliance, faster.

### Admin Changes

### Databases

#### Schema Management

#### Schema Changes

## Upgrade procedure
1. Upgrade to the latest patch level release of the dCache release being in use.
2. Prior to upgrading, run `dcache check-config` and fix any warnings. Information about properties marked deprecated, obsolete or forbidden in version 2.13 or earlier has been dropped in 2.16. Failing to do this before upgrading will mean that you will not receive warnings or errors for using an old property.
3. If the node relies on any databases (you may check the output of dcache database ls to recognize the services that do), then tag the current schema version by running `dcache database tag dcache-2.10`.
4. If you have installed any third party plugins that offer new services (that you have instantiated in the layout file), then remove these and check with the vendor for updated versions.
* **CMS-TFC Plugins** can be downloaded from [XRootD CMS-TFC Releases](https://www.dcache.org/downloads/xrootd4j/index.shtml) in [dCache.org](https://www.dcache.org/).
 * Package `xrootd4j-cms-plugin-1.3.7-1.noarch.rpm` is actually working with dCache 2.16.
* **ATLAS N2N Plugins** can be downloaded from the [WLCG Repository](http://linuxsoft.cern.ch/wlcg/sl6/x86_64/).
 * Package `dcache-xrootd-n2n-plugin-6.0.7-0.noarch.rpm` is actually working with dCache 2.16.
* **XRootD Monitoring** plugins can be found in the [WLCG Repository](http://linuxsoft.cern.ch/wlcg/sl6/x86_64/).
 * Package `dcache-plugin-xrootd-monitor-7.0.0-0.noarch.rpm` is actually working with dCache 2.16.
5. Run `dcache services` and compare the services with the [table](https://github.com/dCache/upgrade-guide-216/blob/master/UPGRADE216.md#services) listing changed services. If any of those services are used, replace them with the suggested alternative after upgrading.
6. Ensure that `Java 8` is installed as your default Java (or unique). If you come from dCache 2.13 or higher you should have this done already.
7.  Install the dCache 2.16 package using your package manager.
8.  Reset `/etc/dcache/logback.xml` to the default file. How this is done depends on your package manager: Debian will ask during package installation whether you want the new version from the package (say yes). RPM will leave the new version as /etc/dcache/logback.xml.rpmnew and expects you to copy it over.
9.  If you used either head, pool, or single as the layout name, you should check that the package manager hasn't renamed your layout file.
10.  Run `dcache check-config`. You will receive an error for any forbidden property and a warning for any deprecated or obsolete property. You will receive errors about unrecognized services defined in the layout file. You will also receive information about properties not recognized by dCache - if these are typos, fix them. Fix any errors and repeat. As a bare minimum, the following changes commonly have to be applied during this step:
* Remove deprecated services.
* Configure **Zookeeper** according to the [above](https://github.com/dCache/upgrade-guide-216/blob/master/UPGRADE216.md#introduced-apache-zookeeper-as-an-external-coordination-service) information. You can run Zookeeper as a service in dCache or as a Standalone service.  Please, for more information refer to the [dCache+Zookeeper GitHub Wiki](https://github.com/dCache/dcache/wiki/ZooKeeper).
* Configure and update the **SRM** configuration as specified in this [documentation](https://github.com/dCache/upgrade-guide-216/blob/master/UPGRADE216.md#split-of-srm-into-a-frontend-and-a-backend). You should run two cells per each SRM service: a *frontend* (`srm`) and a *backend* (`srmmanager`).
* `dcache.enable.space-reservation`is set to `true` by default. Check if this needs to be disabled for specific services.
11. Verify that your HSM script can handle the remove command. If not, update the HSM script or disable the HSM cleaner.
* Sites using [Enstore](http://www.fnal.gov/docs/products/enstore/) will have to set `cleaner.enable.hsm=false` in the `cleaner` service.
12. If you have customized the look of the webdav door, you should reset the style to the default and reapply any customizations.
13. If you are using [pCells](https://www.dcache.org/downloads/gui/index.shtml), please update to the latest version. Check the [GUI Download Page](https://www.dcache.org/downloads/gui/index.shtml) in order to see which versions work with dCache 2.13.
14. If the node relies on any databases, then update the schemas by running `dcache database update`.
15. Start dCache and verify that it works.
16. Run `dcache check-config` and fix any warnings.

Note that some properties that were previously used by several services have to be replaced with a configuration for each service type. Check the documentation in `/usr/share/dcache/defaults` for any property you need to change.

Also note that some properties have changed values or are inverted. The deprecated properties retain their old interpretation, however when replacing those with their new names, you have to update the value. In particular boolean properties that used to accept values like yes, no, enabled, disabled, now only accept true and false.

## Frequently Asked Questions

