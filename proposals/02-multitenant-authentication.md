# Multi tenant authentication: 2.0

## Motivation

As our 2.0 is based on *openshift origin/kubernetes* which has ability to authenticate and authorize api requests. It doesn't provide multi-tenant ability with user management, this means we need to add an authentication layer that MegamVertice can communicate.

For the *Onboard cloud project* we have externalized the auth to be managed using **Google**.

Here we will have to build an **authserver** backed by LDAP using the same API that nilavu  uses today. MegamVertice will integrate directly to the ldap.

## Solution Overview

The auth architecture link [refer slide6](https://docs.google.com/presentation/d/1tzkWbHu6RclA0QWnoEFy9HK0KmISdCjLNfv5QxwJ3Mg/edit?usp=sharing)

We'll have to think through on

1. How does an user authenticate with Nilavu ?
2. How does MegamVertice authenticate an API request  ?
3. How do our third party systems (WHMCS) talk to MegamVertice ?

### 1. How does an user authenticate with Nilavu ?

Nilavu will use the same API and call the **authserver** which is backed by LDAP.

The crux here is make sure the engine based on **openshift origin/kubernetes** can talk to the authentication server.

### 2. How does MegamVertice authenticate an API request

We refer *MegamVertice* in the context of **openshift origin/kubernetes** modified to suit our need. This engine will be configured to authenticate using **LDAPProvider**.

### 3. How will third party systems (WHMCS) talk to MegamVertice

We will have *bot*  **ServiceAccounts** for every 3rd party system integration. The **billbot** in our case will handle billing related ops.

The billing design will be covered in a separate documentation.

## Detailed design

An `OCaml` based [authserver - name needed](https://github.com/megamsys/authserver) will form the fulcrum for MegamVertice2.0 user management.

For detailed architecture [refer slide-7](https://docs.google.com/presentation/d/1tzkWbHu6RclA0QWnoEFy9HK0KmISdCjLNfv5QxwJ3Mg/edit?usp=sharing) of Architecture v2.0.

`Openshift/Origin` doesn't have ability to create new users and manage them. But SaaS products need them.

`Openshift/Origin` has ability to authenticate when an user is available in a 3rd party system like `LDAP`, `OAuth providers`, `Basic auth using http via a flat file`.

Hence we need a REST based [authserver](https://github.com/megamsys/authserver) for user management backed by `LDAP`.

As `Openshift/Origin` supports `LDAP`, it will directly bind to the `LDAP` for authentication verification.

### OCaml authserver

We are yet to coin a name for the server, but the purpose is clear as we have worked on [Onboard cloud - abcd project](https://github.com/megamsys/abcd) which used the OAuth technique.

This will be a REST based server (or) `gRPC`  based.  Apparently `gRPC` isn't available yet for OCaml.

The OCaml authserver requirements are:

#### 1. Secure TLS for system to system communication

We'll generate TLS certificate when the authserver starts and store it in **$MEGAM_HOME/authserver** path. The security key files will be named as **<authserver.key>** and **<authserver.pem>**

If `Nilavu` and the `OCaml authserver` run in different machines then we have to make sure the **$MEGAM_HOME/authserver/keyfiles** are copied over to the `nilavu` machine in the directory mentioned above.


#### 2. API Protection

The user level multitenancy was handled historically used a cyptographic HMAC per user to access the API, (or) global service accounts (or) BasicAuth: userid/pw.  We had proctected every API request.

In this case we protect systems using `TLS`. The `login`, `account.show` are handled by `Openshift` API via the kubeclient gem.


#### 4. REST

Here are the API calls we'll cover, although we'll migrate to `gRPC` in the future, as `OCaml gRPC` seems to be missing.


| Verb | REST                     | Description                                                                                                 |
|------|--------------------------|-------------------------------------------------------------------------------------------------------------|
| GET  | /accounts/forgot/:email  | Initiate the process where an user forgot an email. Generate a unique token and store it in userPassword    |
| POST | /accounts/content        | Create a new user                    |
| POST | /accounts/update         | Modify an user                       |
| POST | /accounts/password_reset | Reset the password by doing a due-diligence verification of the password_reset_token sent with userPassword |


### Native: Account

`Native Account` means user management is handled in MegamVertice.

Nilavu `AuthDispatcher`, and the `User model` needs to trim the variables and change them accordingly.

Nilavu `User model` needs to convert `integer` based edge comparison to `boolean based` flags to indicate `active, staged, blocked, suspended, approved`.

The below table is needed for the  internals of the `authserver` which handles user management backed by `LDAP`.

This table may not be off much importance to nilavu, since the `AuthDispatcher` will act as a fascade to nilavu to feed the correct JSON as it uses today. We may have to trim some of the fields that are not used and which doesn't make sense.


| Cassandra               | LDAP                                  | Description                                                                                                            |
|-------------------------|---------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| first_name              | givenname                             | The first name of the user                                                                                             |
| last_name               | sn (surname)                          | The last name of the user                                                                                              |
| email                   | mail                                  | The email id of the user                                                                                               |
| api_key                 | will reside in etcd as `access_token` | An unique key used by every user to identify themselves to the API server during every transaction.                    |
| mobile                  |                                       | Mobile number of an user                                                                                               |
| phone_verified          |                                       | Indicates if the users mobile phone is enabled for OTP                                                                 |
| password_hash           | `userpassword`                        | A super encrypted password hash                                                                                        |
| password_reset_key      | `userpassword`                        | An unique token shared via email to an user, which allows an user to rest their password.                              |
| password_reset_sent_at  | system field `modifytimestamp`        | The generated time when the password reset token.                                                                      |
| authority               | `employeetype`                        | To decide if the user has admin, normal  (or) clusteradmin access                                                      |
| active                  | shadowinactive = 0 shadowinactive = 1 | 0     = Indicates an user is active > 0 = Indicates an user is inactive                                                |
| blocked                 | shadowflag = 5                        | 5 = Indicates an user is blocked.                                                                                      |
| staged                  | shadowflag = 1                        | 1 = Indicates an user is staged.  There are still steps pending to get onboarded.                                      |
| approved                | shadowflag = 4                        | 4 = Indicates an user is approved and has completed the onboarding steps. The user starts at 1..and progresses till 4. |
| approved_by_id          | system field `modifiersname`          |                                                                                                                        |
| approved_at             |                                       | The timestamp an user was approved at                                                                                  |
| suspended               | shadowflag = 6                        | 6 = Indicates an user is suspended                                                                                     |
| suspended_at            | system field `modifiertimestamp`      | The timestamp an user was suspended at                                                                                 |
| suspended_till          |                                       | The timestamp an user will be suspended till                                                                           |
| registration_ip_address | iphostnumber                          | The ip address from where the user registered.                                                                         |
| last_posted_at          |                                       | The last launch timestamp                                                                                              |
| last_emailed_at         |                                       | The last emailed timestamp when notifications are sent                                                                 |
| previous_visit_at       |                                       | The visit prior an user made into the system                                                                           |
| first_seen_at           |                                       | The first visit an user made into the system                                                                           |
| created_at              | system field `createtimestamp`        | The timestamp the entry was created                                                                                    |


```

The LDIF record. Here we create an user with email `kenji.kouda@jap.ai` in dc `example.org`

##

# Entry 1: cn=kenji.kouda@jap.ai,dc=example,dc=org
dn: cn=kenji.kouda@jap.ai,dc=example,dc=org
cn: kenji.kouda@jap.ai
employeetype: admin
givenname: kenji
iphostnumber: 192.168.1.1
mail: kenji.kouda@jap.ai
objectclass: inetOrgPerson
objectclass: topshadowexpire: 1

objectclass: shadowAccount
objectclass: ipHost
shadowexpire: 1
shadowflag: 1
shadowinactive: 0
sn: kouda
uid: kenjikouda
userpassword: {SSHA}+21Wj2mKpQV0k9z4wyF/aHK82i/oQkVo

```

Groups

```
# LDIF fragment to create group branch under root

dn: ou=groups,dc=megam, dc=local
objectclass:organizationalunit
ou: groups
description: generic groups branch

# Create the admins entry

dn: cn=admins,ou=groups,dc=megam,dc=local
objectclass: groupofnames
cn: admins
description: Megam Vertice Admin Group
# add the group members all of which are
# assumed to exist under people
member: cn=road runner,ou=people,dc=megam,dc=local
member: cn=micky mouse,ou=people,dc=megam,dc=local

# Create the cluster-admins entry

dn: cn=clusteradmins,ou=groups,dc=megam,dc=local
objectclass: groupofnames
cn: clusteradmins
description: Megam Vertice Cluster adminitrators
# add the group members all of which are
# assumed to exist under people
member: cn=road runner,ou=people,dc=megam,dc=local

```

## OAuth: Account

Upon authentication with the OAuth provier, we create an `Account` in the `authserver` if it doesn't exists. The rest of the workflow is as per `NativeAccount`

### Development setup

1. Run openldap

```
docker run --name openldap --detach osixia/openldap:1.1.8

```

2. *optional* Get the openldap servers and start a ldapadmin

This step is optional and only needed in development, since we can manage using command line.

```

LDAP_IP=$(docker inspect -f "{{ .NetworkSettings.IPAddress }}" openldap)

$ echo $LDAP_IP
172.17.0.2

docker run --name ldapadmin -p 6443:443 \
       --env PHPLDAPADMIN_LDAP_HOSTS=$LDAP_IP \
       --detach osixia/phpldapadmin:0.6.12

```

3.  Browse openldap using the ldapadmin

```

LDAP_ADMIN=$(docker inspect -f "{{ .NetworkSettings.IPAddress }}" ldapadmin)

$ echo $LDAP_ADMIN
172.17.0.3

```

Type [https://$LDAP_ADMIN:6443](https://$LDAP_ADMIN:6443) to manage LDAP accounts.


## Production setup

1. Run openldap in TLS [section - Use auto-generated certificate](https://github.com/osixia/docker-openldap#tls)


# References

- [Book: LDAP System administration](http://shop.oreilly.com/product/9781565924918.do)
- [Designing LDAP schema](https://www.skills-1st.co.uk/papers/ldap-schema-design-feb-2005/ldap-schema-design-feb-2005.html)
- [LDAP Schemas, objectClasses](http://www.zytrax.com/books/ldap/ch3/)
