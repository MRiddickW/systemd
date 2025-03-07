---
title: Users, Groups, UIDs and GIDs on systemd Systems
category: Users, Groups and Home Directories
layout: default
SPDX-License-Identifier: LGPL-2.1-or-later
---

# Users, Groups, UIDs and GIDs on systemd Systems

Here's a summary of the requirements `systemd` (and Linux) make on UID/GID
assignments and their ranges.

Note that while in theory UIDs and GIDs are orthogonal concepts they really
aren't IRL. With that in mind, when we discuss UIDs below it should be assumed
that whatever we say about UIDs applies to GIDs in mostly the same way, and all
the special assignments and ranges for UIDs always have mostly the same
validity for GIDs too.

## Special Linux UIDs

In theory, the range of the C type `uid_t` is 32bit wide on Linux,
i.e. 0…4294967295. However, four UIDs are special on Linux:

1. 0 → The `root` super-user

2. 65534 → The `nobody` UID, also called the "overflow" UID or similar. It's
   where various subsystems map unmappable users to, for example file systems
   only supporting 16bit UIDs, NFS or user namespacing. (The latter can be
   changed with a sysctl during runtime, but that's not supported on
   `systemd`. If you do change it you void your warranty.) Because Fedora is a
   bit confused the `nobody` user is called `nfsnobody` there (and they have a
   different `nobody` user at UID 99). I hope this will be corrected eventually
   though. (Also, some distributions call the `nobody` group `nogroup`. I wish
   they didn't.)

3. 4294967295, aka "32bit `(uid_t) -1`" → This UID is not a valid user ID, as
   `setresuid()`, `chown()` and friends treat -1 as a special request to not
   change the UID of the process/file. This UID is hence not available for
   assignment to users in the user database.

4. 65535, aka "16bit `(uid_t) -1`" → Before Linux kernel 2.4 `uid_t` used to be
   16bit, and programs compiled for that would hence assume that `(uid_t) -1`
   is 65535. This UID is hence not usable either.

The `nss-systemd` glibc NSS module will synthesize user database records for
the UIDs 0 and 65534 if the system user database doesn't list them. This means
that any system where this module is enabled works to some minimal level
without `/etc/passwd`.

## Special Distribution UID ranges

Distributions generally split the available UID range in two:

1. 1…999 → System users. These are users that do not map to actual "human"
   users, but are used as security identities for system daemons, to implement
   privilege separation and run system daemons with minimal privileges.

2. 1000…65533 and 65536…4294967294 → Everything else, i.e. regular (human) users.

Note that most distributions allow changing the boundary between system and
regular users, even during runtime as user configuration. Moreover, some older
systems placed the boundary at 499/500, or even 99/100. In `systemd`, the
boundary is configurable only during compilation time, as this should be a
decision for distribution builders, not for users. Moreover, we strongly
discourage downstreams to change the boundary from the upstream default of
999/1000.

Also note that programs such as `adduser` tend to allocate from a subset of the
available regular user range only, usually 1000..60000. And it's also usually
user-configurable, too.

Note that systemd requires that system users and groups are resolvable without
networking available — a requirement that is not made for regular users. This
means regular users may be stored in remote LDAP or NIS databases, but system
users may not (except when there's a consistent local cache kept, that is
available during earliest boot, including in the initial RAM disk).

## Special `systemd` GIDs

`systemd` defines no special UIDs beyond what Linux already defines (see
above). However, it does define some special group/GID assignments, which are
primarily used for `systemd-udevd`'s device management. The precise list of the
currently defined groups is found in this `sysusers.d` snippet:
[basic.conf](https://raw.githubusercontent.com/systemd/systemd/main/sysusers.d/basic.conf.in)

It's strongly recommended that downstream distributions include these groups in
their default group databases.

Note that the actual GID numbers assigned to these groups do not have to be
constant beyond a specific system. There's one exception however: the `tty`
group must have the GID 5. That's because it must be encoded in the `devpts`
mount parameters during earliest boot, at a time where NSS lookups are not
possible. (Note that the actual GID can be changed during `systemd` build time,
but downstreams are strongly advised against doing that.)

## Special `systemd` UID ranges

`systemd` defines a number of special UID ranges:

1. 60001…60513 → UIDs for home directories managed by
   [`systemd-homed.service(8)`](https://www.freedesktop.org/software/systemd/man/systemd-homed.service.html). UIDs
   from this range are automatically assigned to any home directory discovered,
   and persisted locally on first login. On different systems the same user
   might get different UIDs assigned in case of conflict, though it is
   attempted to make UID assignments stable, by deriving them from a hash of
   the user name.

2. 61184…65519 → UIDs for dynamic users are allocated from this range (see the
   `DynamicUser=` documentation in
   [`systemd.exec(5)`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html)). This
   range has been chosen so that it is below the 16bit boundary (i.e. below
   65535), in order to provide compatibility with container environments that
   assign a 64K range of UIDs to containers using user namespacing. This range
   is above the 60000 boundary, so that its allocations are unlikely to be
   affected by `adduser` allocations (see above). And we leave some room
   upwards for other purposes. (And if you wonder why precisely these numbers:
   if you write them in hexadecimal, they might make more sense: 0xEF00 and
   0xFFEF). The `nss-systemd` module will synthesize user records implicitly
   for all currently allocated dynamic users from this range. Thus, NSS-based
   user record resolving works correctly without those users being in
   `/etc/passwd`.

3. 524288…1879048191 → UID range for `systemd-nspawn`'s automatic allocation of
   per-container UID ranges. When the `--private-users=pick` switch is used (or
   `-U`) then it will automatically find a so far unused 16bit subrange of this
   range and assign it to the container. The range is picked so that the upper
   16bit of the 32bit UIDs are constant for all users of the container, while
   the lower 16bit directly encode the 65536 UIDs assigned to the
   container. This mode of allocation means that the upper 16bit of any UID
   assigned to a container are kind of a "container ID", while the lower 16bit
   directly expose the container's own UID numbers. If you wonder why precisely
   these numbers, consider them in hexadecimal: 0x00080000…0x6FFFFFFF. This
   range is above the 16bit boundary. Moreover it's below the 31bit boundary,
   as some broken code (specifically: the kernel's `devpts` file system)
   erroneously considers UIDs signed integers, and hence can't deal with values
   above 2^31. The `systemd-machined.service` service will synthesize user
   database records for all UIDs assigned to a running container from this
   range.

Note for both allocation ranges: when an UID allocation takes place NSS is
checked for collisions first, and a different UID is picked if an entry is
found. Thus, the user database is used as synchronization mechanism to ensure
exclusive ownership of UIDs and UID ranges. To ensure compatibility with other
subsystems allocating from the same ranges it is hence essential that they
ensure that whatever they pick shows up in the user/group databases, either by
providing an NSS module, or by adding entries directly to `/etc/passwd` and
`/etc/group`. For performance reasons, do note that `systemd-nspawn` will only
do an NSS check for the first UID of the range it allocates, not all 65536 of
them. Also note that while the allocation logic is operating, the glibc
`lckpwdf()` user database lock is taken, in order to make this logic race-free.

## Figuring out the system's UID boundaries

The most important boundaries of the local system may be queried with
`pkg-config`:

```
$ pkg-config --variable=systemuidmax systemd
999
$ pkg-config --variable=dynamicuidmin systemd
61184
$ pkg-config --variable=dynamicuidmax systemd
65519
$ pkg-config --variable=containeruidbasemin systemd
524288
$ pkg-config --variable=containeruidbasemax systemd
1878982656
```

(Note that the latter encodes the maximum UID *base* `systemd-nspawn` might
pick — given that 64K UIDs are assigned to each container according to this
allocation logic, the maximum UID used for this range is hence
1878982656+65535=1879048191.)

Systemd has compile-time default for these boundaries. Using those defaults is
recommended. It will nevertheless query `/etc/login.defs` at runtime, when
compiled with `-Dcompat-mutable-uid-boundaries=true` and that file is present.
Support for this is considered only a compatibility feature and should not be
used except when upgrading systems which were created with different defaults.

## Considerations for container managers

If you hack on a container manager, and wonder how and how many UIDs best to
assign to your containers, here are a few recommendations:

1. Definitely, don't assign less than 65536 UIDs/GIDs. After all the `nobody`
user has magic properties, and hence should be available in your container, and
given that it's assigned the UID 65534, you should really cover the full 16bit
range in your container. Note that systemd will — as mentioned — synthesize
user records for the `nobody` user, and assumes its availability in various
other parts of its codebase, too, hence assigning fewer users means you lose
compatibility with running systemd code inside your container. And most likely
other packages make similar restrictions.

2. While it's fine to assign more than 65536 UIDs/GIDs to a container, there's
most likely not much value in doing so, as Linux distributions won't use the
higher ranges by default (as mentioned neither `adduser` nor `systemd`'s
dynamic user concept allocate from above the 16bit range). Unless you actively
care for nested containers, it's hence probably a good idea to allocate exactly
65536 UIDs per container, and neither less nor more. A pretty side-effect is
that by doing so, you expose the same number of UIDs per container as Linux 2.2
supported for the whole system, back in the days.

3. Consider allocating UID ranges for containers so that the first UID you
assign has the lower 16bits all set to zero. That way, the upper 16bits become
a container ID of some kind, while the lower 16bits directly encode the
internal container UID. This is the way `systemd-nspawn` allocates UID ranges
(see above). Following this allocation logic ensures best compatibility with
`systemd-nspawn` and all other container managers following the scheme, as it
is sufficient then to check NSS for the first UID you pick regarding conflicts,
as that's what they do, too. Moreover, it makes `chown()`ing container file
system trees nicely robust to interruptions: as the external UID encodes the
internal UID in a fixed way, it's very easy to adjust the container's base UID
without the need to know the original base UID: to change the container base,
just mask away the upper 16bit, and insert the upper 16bit of the new container
base instead. Here are the easy conversions to derive the internal UID, the
external UID, and the container base UID from each other:

    ```
    INTERNAL_UID = EXTERNAL_UID & 0x0000FFFF
    CONTAINER_BASE_UID = EXTERNAL_UID & 0xFFFF0000
    EXTERNAL_UID = INTERNAL_UID | CONTAINER_BASE_UID
    ```

4. When picking a UID range for containers, make sure to check NSS first, with
a simple `getpwuid()` call: if there's already a user record for the first UID
you want to pick, then it's already in use: pick a different one. Wrap that
call in a `lckpwdf()` + `ulckpwdf()` pair, to make allocation
race-free. Provide an NSS module that makes all UIDs you end up taking show up
in the user database, and make sure that the NSS module returns up-to-date
information before you release the lock, so that other system components can
safely use the NSS user database as allocation check, too. Note that if you
follow this scheme no changes to `/etc/passwd` need to be made, thus minimizing
the artifacts the container manager persistently leaves in the system.

## Summary

|               UID/GID | Purpose               | Defined By    | Listed in                     |
|-----------------------|-----------------------|---------------|-------------------------------|
|                     0 | `root` user           | Linux         | `/etc/passwd` + `nss-systemd` |
|                   1…4 | System users          | Distributions | `/etc/passwd`                 |
|                     5 | `tty` group           | `systemd`     | `/etc/passwd`                 |
|                 6…999 | System users          | Distributions | `/etc/passwd`                 |
|            1000…60000 | Regular users         | Distributions | `/etc/passwd` + LDAP/NIS/…    |
|           60001…60513 | Human users (homed)   | `systemd`     | `nss-systemd`                 |
|           60514…60577 | Host users mapped into containers | `systemd` | `systemd-nspawn`           |
|           60578…61183 | Unused                |               |                               |
|           61184…65519 | Dynamic service users | `systemd`     | `nss-systemd`                 |
|           65520…65533 | Unused                |               |                               |
|                 65534 | `nobody` user         | Linux         | `/etc/passwd` + `nss-systemd` |
|                 65535 | 16bit `(uid_t) -1`    | Linux         |                               |
|          65536…524287 | Unused                |               |                               |
|     524288…1879048191 | Container UID ranges  | `systemd`     | `nss-systemd`                 |
| 1879048192…2147483647 | Unused                |               |                               |
| 2147483648…4294967294 | HIC SVNT LEONES       |               |                               |
|            4294967295 | 32bit `(uid_t) -1`    | Linux         |                               |

Note that "Unused" in the table above doesn't mean that these ranges are
really unused. It just means that these ranges have no well-established
pre-defined purposes between Linux, generic low-level distributions and
`systemd`. There might very well be other packages that allocate from these
ranges.

Note that the range 2147483648…4294967294 (i.e. 2^31…2^32-2) should be handled
with care. Various programs (including kernel file systems, see `devpts`) have
trouble with UIDs outside of the signed 32bit range, i.e any UIDs equal to or
above 2147483648. It is thus strongly recommended to stay away from this range
in order to avoid complications. This range should be considered reserved for
future, special purposes.

## Notes on resolvability of user and group names

User names, UIDs, group names and GIDs don't have to be resolvable using NSS
(i.e. getpwuid() and getpwnam() and friends) all the time. However, systemd
makes the following requirements:

System users generally have to be resolvable during early boot already. This
means they should not be provided by any networked service (as those usually
become available during late boot only), except if a local cache is kept that
makes them available during early boot too (i.e. before networking is
up). Specifically, system users need to be resolvable at least before
`systemd-udevd.service` and `systemd-tmpfiles.service` are started, as both
need to resolve system users — but note that there might be more services
requiring full resolvability of system users than just these two.

Regular users do not need to be resolvable during early boot, it is sufficient
if they become resolvable during late boot. Specifically, regular users need to
be resolvable at the point in time the `nss-user-lookup.target` unit is
reached. This target unit is generally used as synchronization point between
providers of the user database and consumers of it. Services that require that
the user database is fully available (for example, the login service
`systemd-logind.service`) are ordered *after* it, while services that provide
parts of the user database (for example an LDAP user database client) are
ordered *before* it. Note that `nss-user-lookup.target` is a *passive* unit: in
order to minimize synchronization points on systems that don't need it the unit
is pulled into the initial transaction only if there's at least one service
that really needs it, and that means only if there's a service providing the
local user database somehow through IPC or suchlike. Or in other words: if you
hack on some networked user database project, then make sure you order your
service `Before=nss-user-lookup.target` and that you pull it in with
`Wants=nss-user-lookup.target`. However, if you hack on some project that needs
the user database to be up in full, then order your service
`After=nss-user-lookup.target`, but do *not* pull it in via a `Wants=`
dependency.
