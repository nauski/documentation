---
layout: doc
title: userdb
dovecotlinks:
  userdb: userdb
  userdb_fields:
    hash: fields
    text: "userdb: Fields"
  userdb_extra_fields:
    hash: extra-fields
    text: "userdb: Extra Fields"
  userdb_result_values:
    hash: result-values
    text: "userdb: Result Values"
---

# User Databases (`userdb`)

Dovecot uses [[link,passdb]] and `userdb` as part of the authentication
process.

[[link,passdb]] authenticated the user. `userdb` lookup then retrieves
post-login information specific to the authenticated user. This may include:

* Mailbox location information
* Quota limit
* Overriding settings for the user (almost any setting can be overridden)

| Userdb Lookups | Dovecot Proxy | Dovecot Backend |
| -------------- | ------------- | --------------- |
| IMAP & POP3 logins | No | YES |
| LMTP mail delivery | No | YES |
| doveadm commands | No | YES |

The `userdb` and [[link,passdb]] may be the same or they may be different
depending on your needs. You can also have [[link,auth_multiple_dbs]].

## Multiple Userdbs

You can use multiple databases, so if the user isn't found from the first
database, Dovecot checks the next one.

You can control the behavior of what happens after a userdb lookup is
successful or unsuccessful with settings:

 * [[setting,userdb_result_success]]
 * [[setting,userdb_result_failure]]
 * [[setting,userdb_result_internalfail]]

When using userdbs inside a `protocol ... { ... }` filter, the protocol-specific
userdbs are executed first, and only then followed by the globally defined
userdbs. Currently it's not possible to configure userdbs inside other filters.

## Fields

The user database lookup can return these fields:

### `uid`

User's UID (UNIX user ID), overrides the global [[setting,mail_uid]] setting.

### `gid`

User's GID (UNIX group ID), overrides the global [[setting,mail_gid]] setting.

### `home`

User's home directory, overrides the global [[setting,mail_home]] setting.

Although not required, it's highly recommended even for virtual users.

### Optional Extra Fields

The extra fields are also passed to post-login scripts. See
[[link,post_login_scripting]]. You can override settings in `dovecot.conf`, see
[[link,userdb_extra_fields]].

#### `user`

Changes the username (can also be done by the [[link,passdb]] lookup).

## Supported Databases

| Database | Description |
| -------- | ----------- |
| [[link,auth_passwd,Passwd]] | System users (NSS, `/etc/passwd`, or similar). |
| [[link,auth_passwd_file,Passwd-file]] | `/etc/passwd`-like file. |
| [[link,auth_ldap,LDAP]] | Lightweight Directory Access Protocol. |
| [[link,auth_sql,SQL]] | SQL database (PostgreSQL, MySQL, SQLite, Cassandra). |
| [[link,auth_staticdb,Static]] | Userdb information generated from a given template. |
| [[link,auth_prefetch,Prefetch]] | Prefetch database. This assumes that the [[link,passdb]] already returned also all the required user database information. |
| [[link,auth_lua,Lua]] | Lua script for authentication. |

## Settings

<SettingsComponent tag="userdb" />

## Result Values

The following values control the behavior of a userdb lookup result:

### `return-ok`

Return success, don't continue to the next `userdb`.

### `return-fail`

Return "user doesn't exist", don't continue to the next `userdb`.

### `return`

Return earlier `userdb`'s success or failure, don't continue to the
next `userdb`. If this was the first `userdb`, return "user doesn't exist".

### `continue-ok`

Set the current user existence state to "found", and continue to the next
`userdb`.

### `continue-fail`

Set the current user existence state to "not found", and continue to the
next `userdb`.

### `continue`

Continue to the next `userdb` without changing the user existence state.
The initial state is "not found".

## Extra Fields

A user database lookup typically returns `uid`, `gid`, and `home` fields,
as per traditional `/etc/passwd` lookups.

Other fields may also be stored in the `userdb`, and these are called 'extra
fields'.

These fields can be returned the exact same way as `uid`, `gid`, and `home`
fields.

It's also possible to override settings from `dovecot.conf`. For example the
[[setting,mail_path]] and [[setting,quota_storage_size]] settings are commonly
overridden to provide per-user mail path or quota limit.

The extra fields are also passed to [[link,post_login_scripting]].

::: info [[changed,extra_fields_empty]]
Extra fields can now also be set to empty string, while previously they were
changed to `yes`.

Extra fields without value (without `=`) will default to `yes`.
:::

### Suffixes

The following suffixes added to a field name are handled specially:

#### `:default`

Set this field only if it hasn't been set before.

::: info [[changed,extra_fields_default]]
This was called `:protected` in earlier versions.
:::

#### `:remove`

Remove this field entirely.

For example you can return `uid:remove` to remove the `uid` field from the
userdb reply. This differs from `uid=` in that the field is removed
entirely (and default is used) instead of just being set to an empty value.

### Fields

#### `nice`

Set the mail process's priority to be the given value.

#### `chroot`

Chroot to given directory. Overrides [[setting,mail_chroot]].

#### `system_groups_user`

Specifies the username whose groups are read from `/etc/group` (or
wherever NSS is configured to taken them from).

The logged in user has access to those groups.

This may be useful for shared mailboxes.

#### `userdb_import`

This allows returning multiple extra fields in one TAB-separated field.
It's useful for `userdb`s which are a bit less flexible for returning a
variable number of fields (e.g. SQL).

#### `uidgid_file`

Get `uid` and `gid` for user based on the given filename.

#### `user`

User can be overridden (normally set in [[link,passdb]]).

#### `event_<name>`

Import `name=value` to mail user event.

### Overriding Settings

For example if you have `quota_storage_size=100M` in `doveconf.conf` and the
`userdb` lookup returns `quota_storage_size=200M`, the original
quota setting gets overridden. In fact, if the lookup always returns a
`quota_storage_size` field, there's no point in having
[[setting,quota_storage_size]] in `dovecot.conf` at all, because it always gets
overridden anyway.

To understand how imap and pop3 processes see their settings, it may be
helpful to know how Dovecot internally passes them:

1. First all actual settings are first read into memory.
2. Next all the extra fields returned by `userdb` lookup are used to override
   the settings.
3. Last, if [[link,post_login_scripting]] is used, it may modify the
   settings.

If you want to override settings inside sections, you can separate the
section name and key with `/`. For example:

```[dovecot.conf]
namespace default {
  inbox = yes
  separator = .
}
```

The separator setting can be overridden by returning
[[setting_text,namespace_separator,namespace/default/separator]] extra field.

### Examples

#### SQL

::: code-group
```[dovecot.conf]
userdb sql {
  query = SELECT home, uid, gid, CONCAT(quota_bytes, 'B') AS quota_storage_size, separator AS "namespace/default/separator" \
    FROM users \
    WHERE username = '%{user | username}' and domain = '%{user | domain}'
}
```
:::

#### LDAP

::: code-group
```[dovecot.conf]
userdb ldap {
  ...
  fields {
    home       = %{ldap:homeDirectory}
    uid        = %{ldap:uidNumber}
    gid        = %{ldap:gidNumber}
    quota_storage_size = %{ldap:quotaBytes}B
    namespace/default/separator = %{ldap:mailSeparator}
  }
}
```
:::

#### passwd-file

Example that shows how to give two `userdb` extra fields (`mail_driver` and
`quota`).

Note that all `userdb` extra fields must be prefixed with `userdb_`,
otherwise they're treated as [[link,passdb_extra_fields]].

```
user:{plain}pass:1000:1000::/home/user::userdb_mail_driver=mbox userdb_mail_path=~/mail userdb_quota_storage_size=100M userdb_namespace/default/separator=/
user2:{plain}pass2:1001:1001::/home/user2::userdb_mail_driver=maildir userdb_mail_path=~/Maildir userdb_quota_storage_size=200M
```

## See Also

- [[link,auth_caching]]
