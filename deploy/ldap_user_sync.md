# Syncing Users from LDAP/AD

In community edition, a LDAP/AD user is imported into Seafile database upon first time login. There are a few drawbacks:

* User name, department and other information are not imported into Seafile
* If an user is deleted in LDAP/AD, it is not deactivated in Seafile. He/She is still able to access files via syncing or via existing browser session.

The Pro Edition supports syncing users from LDAP or Active Directory to Seafile's internal database (ccnet-db/LDAPUser).

A related feature is [importing groups from LDAP/AD](ladp_group_sync.md).

## How It Works

The syncing process imports users from LDAP directory server to Seafile's internal database. This process is one-way.

* All existing users on LDAP server will be created as new users in Seafile database. **The new users will be active by default. They'll be counted as licensed users.**
* Any changes to user's property in the database won't propagate back to LDAP.
* If an imported user is deleted in Seafile, it will be re-imported in the next sync operation.

In addition to syncing the user from LDAP, the syncing process provides more features:

* Addtional information, including user full name, department, can be synced to database. With these information in the database, the user experience of a few other functionalities can be improved. For example, when users share a folder to others, he/she can type in user's full name to search the user.
* When a user was removed on LDAP server, the corresponding user in Seafile will be deactivated.

Since Pro Edition 4.4.4, we introduce a new syncing mode, that the user syncing process won't create new users in the database. New users are only created when the user logs in for the first time (the same behavior as in Community Edition). This way the syncing process won't add new licensed users automatically, which is more convenient for some user cases. Below is how it works:

* New users are only created on first login.
* Any changes to user's property in the database won't propagate back to LDAP.
* If a user is deleted from LDAP server, the user will be deactivated in Seafile.
* If an imported user is deleted in Seafile, it **won't** re-imported in the next sync operation. But the user will be re-imported on next login.
* Addtional information for existing users in the database, including user full name, department, can be synced to database.

There are two ways to trigger user sync:

* Periodical: the syncing process will be executed in a fixed interval
* Manual: there is a script you can run to trigger the syncing once

## Prerequisite

You have to install python-ldap library in your system.

For Debian or Ubuntu

```
sudo apt-get install python-ldap
```

For CentOS or RedHat

```
sudo yum install python-ldap
```

## Configuration

Before enabling LDAP user sync, you should have configured LDAP authentication. See [Configure Seafile to use LDAP](using_ldap.md) for details.

The following are LDAP user sync related options. They're in the "[LDAP_SYNC]" section of [ccnet.conf](../../config/ccnet-conf.md).

* **ENABLE_USER_SYNC**: set to "true" if you want to enable ldap user syncing
* **SYNC_INTERVAL**: The interval to sync. Unit is minutes. Default to 60 minutes.
* **USER_OBJECT_CLASS**: This is the name of the class used to search for user objects. In Active Directory, it's usually "person"; in OpenLDAP or others, you may use other object classes, depends on your LDAP server. The default value is "person".
* **PWD_CHANGE_ATTR**: The attribute field used to detect user password change. The sync process compare the values of this attribute from LDAP and database. In AD, you can use the "pwdLastSet" attribute, which is the last change time of user password. The default value is "pwdLastSet".

The "FILTER" option in the "[LDAP]" section also applies to LDAP user sync. That is, you can add further filtering option to the sync process.

The search base for users is the "BASE_DN" set in "[LDAP]" section of ccnet.conf. 

Here is an example configuration for Active Directory:

```
[LDAP]
HOST = ldap://192.168.1.123/
BASE = cn=users,dc=example,dc=com
USER_DN = administrator@example.local
PASSWORD = secret
LOGIN_ATTR = mail

[LDAP_SYNC]
ENABLE_USER_SYNC = true
SYNC_INTERVAL = 60
```

For AD, you usually don't need to configure other options except for "ENABLE_USER_SYNC". That's because the default values for other options are the usual values for AD. If you have special settings in your LDAP server, just set the corresponding options.

Here is an example configuration for OpenLDAP:

```
[LDAP]
HOST = ldap://192.168.1.123/
BASE = ou=users,dc=example,dc=com
USER_DN = cn=admin,dc=example,dc=com
PASSWORD = secret
LOGIN_ATTR = mail

[LDAP_SYNC]
ENABLE_USER_SYNC = true
SYNC_INTERVAL = 60
USER_OBJECT_CLASS = userOfNames
PWD_CHANGE_ATTR = userPassword
```

**NOTE** Periodical sync won't happen immediately after you restart seafile server. It gets scheduled after the first sync interval. For example if you set sync interval to 30 minutes, the first auto sync will happen after 30 minutes you restarts. To sync immediately, you need to manually trigger it. This is covered in the next section.

After the sync is run, you should be able to see the users in system admin page.

### Syncing Additional User Information

There are a lot more information in the LDAP server that can be useful for Seafile application. The syncing process can now sync user full name and department information to Seafile database. The following options are available:

* **ENABLE_EXTRA_USER_INFO_SYNC**: Enable syncing additional user information.
* **FIRST_NAME_ATTR**: Attribute for user's first name. It's "givenName" by default.
* **LAST_NAME_ATTR**: Attribute for user's last name. It's "sn" by default.
* **USER_NAME_REVERSE**: In some laguages, such as Chinese, the display order of first name and last name is reversed. Set this option if you need it.
* **DEPT_ATTR**: Attribute for user's department. It's "department" by default.

Since Pro edition 5.0.0, two new attributes can be synced:

* **UID_ATTR**: Attribute for username or uid. If this is synced, users can also log in with their uid. In AD, the attribute `sAMAccountName` can be used as `UID_ATTR`; in OpenLDAP, the attribute `uid` or something similar can be used. This attribute is not synced by default.
* **CONTACT_EMAIL_ATTR**: If synced, this email address will be used to send notification emails to users. Otherwise the `LOGIN_ATTR` attribute in `[LDAP]` section will be used for sending emails. This attribute is not synced by default.

You should set these options in the "[LDAP_SYNC]" section of ccnet.conf.

### Don't Import New Users

Since Pro Edition 4.4.4, we introduce a new syncing mode, that the user syncing process won't create new users in the database. More details was described in the "How It Works" section above. To enable this mode, add following line to [LDAP_SYNC] section in ccnet.conf:

```
[LDAP_SYNC]
IMPORT_NEW_USER = false
```

## Manually Trigger Syncing

To trigger LDAP sync manually,

```
cd seafile-server-lastest
./pro/pro.py ldapsync
```
