# Migrate from OpenLDAP to IPA Server

Instructions for migrating an OpenLDAP service to IPA.

## Config files, data and aspects to consider

If you haven't installed FreeIPA server, follow this tutorial (including a replica) [
](Git)

Configuration file for the LDAP service  on IPA Server:
```
cat /etc/openldap/ldap.conf
```

This file contains setup data for LDAP service. This file is in line with the IPA Service, containing schemas and parameter set by IPA Server.

In this case, you would use:

```
ldapsearch -x uid=admin
```
without parameters like *host*, *port*, *URI* o *base-dn*, due to in ```/etc/openldap/ldap.conf``` it has been configured by IPA installation.

Indifferently you can use IPA commands or LDAP commands to interact with the directory service. We have chosen to perform the migration using the IPA tools and complete some aspects with LDAP.


## Starting migration


Review the your old LDAP Directory in your server and try to create queries, in order to define with branch or tree will be imported.

Search in your old OpenLDAP server:

```
 ldapsearch -h myOldServerLDAP -D "cn=adm,dc=ugr,dc=es" -W -b "ou=users,dc=ugr,dc=es"
```
Here we use ``admin`` binding, due to we want to show everything (included passwords [hashed]).

## Start session on IPA Server

Start IPA Session with ``admin`` credentials:

```
kinit admin
```

Enable IPA Migration mode (after migration, consider disable migration label)
```
ipa config-mod --enable-migration=TRUE
```
It returns the following: 
```
ipa: ERROR: no modifications to be performed
```
This is correct, because the mode was TRUE.

## IPA Migration  from OpenLDAP to IPA Server:

This command considers:

- Solve error with attribute SN:  ```missing attribute "sn" required by object class "organizationalPerson"``` adding ```--user-ignore-attribute="sn"``` and ```--user-ignore-objectclass={organizationalPerson,inetOrgPerson,person}```
- Import all the directory (Users)
- Import password, due to the use of  ```--bind-dn="cn=admin,ou=...``` , it provides search on the remote LDAP and extract the passwords.
- Use a remote OpenLDAP server ``myOldServerLDAP``


Then command is:

```
ipa migrate-ds --base-dn="dc=ugr,dc=es" \
  --bind-dn="cn=adm,ou=usr,dc=ugr,dc=es" \
  ldap://myOldServerLDAP --user-objectclass=account  \
  --group-objectclass=organizationalUnit  \
  --user-container="ou=users" \
  --group-container="ou=users" \
  --group-objectclass="account" \
  --continue  --group-overwrite-gid --schema="RFC2307" \
  --user-ignore-attribute="sn" \
  --user-ignore-objectclass={organizationalPerson,inetOrgPerson,person}
```

*This command is really bad documented, with no examples, many thing as default and error output not really detailed.*

Once all users and groups are migrated, user needs validate the password, due to  Kerberos, so, each user must to go http://server.ipa/ipa/migration and write down your credentials, in order to enable the password with Kerberos in the new IPA server.

The output at the end will show:

````
Passwords have been migrated in pre-hashed format. IPA is unable to generate Kerberos keys unless provided with clear text passwords. All migrated users need to login at https://your.domain/ipa/migration/ before they can use their Kerberos accounts.
````

In this moment you can authenticate in the main FreeIPA website with your credential (*Kerberized*) and change your attributes or similar, and authenticate in Server with SSH if enabled (with IPAClients installed).


## How to use LDAP commands in FreeIPA

Remember, FreeIPA use ``-D "cn=Directory Manager" `` to access main tree.

Delete entry using OpenLDAP inside FreeIPA:
(Not the old OpenLDAP, the new LDAP provided by FreeIPA)

Delete a group:
````
ldapdelete -D "cn=Directory Manager" -h freeipa.imuds "cn=manuel jesus parra royn,cn=groups,cn=accounts,dc=imuds" -W
````

Delete an user:
````
ldapdelete -D "cn=Directory Manager" -h freeipa.imuds "cn=mparra,cn=users,cn=accounts,dc=imuds" -W
````

Add new user:

This is strongly not recommended because you must know IPA server rules for LDAP, instead you must use ```ipa migrate-ds```

Example:

````
ldapadd -x  -h freeipa.imuds -D "cn=Directory Manager" -c -f mparra.ldif
````

If the user definition in the ``ldif`` file contains user password, it return an error: ```Password cannot imported hashed```

# Post migration

After the migration, only a few directory maintenance tasks remain. If no default group assignment has been specified for imported users, regardless of the group, it will add the users and groups for each user (user group). 
These groups are migrated and associated with the user, but by default IPA assigns them several default groups to have them containerized. So now that users have their correct password within IPA (and migrated from the password migration web), all that is left to do is to re-establish the new user groups or clean them up.

# IPA Commands and receipts

Search users:

``ipa user-find``

Search all users:

``ipa user-find --all``

Show user details:

```ipa user-show mparra```

Show all user details:

```ipa user-show mparra --all```

Create new user:

Minimal creation require, user, name, surname and email, all other parameter will be set by default (including uid, guid, etc.).

```ipa user-create mparra --email="mparra@cookingbigdata.com"```

Delete users:

```ipa user-del mparra ```

Create group:

```ipa group-create bigdata```

