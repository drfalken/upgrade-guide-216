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

#### Highlights introduced on dCache 2.15

#### Highlights introduced on dCache 2.16

### Incompatibilities

## Services

### Removed Services

## What's new

### Admin Changes

### Databases

#### Schema Management

#### Schema Changes

## Upgrade procedure

## Frequently Asked Questions

