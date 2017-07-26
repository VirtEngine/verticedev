# NOTE: This proposal is parked, as we decided to get rid of `openshift/origin (or) kubernetes`. Please refer the new [multitenant-authentication](verticedev/proposals/02-multitenant-authentication.md)

# Multi tenant authentication - NOT ACTIVE: 2.0

*Terminology*,

We'll use `MOOV` to refer `MegamVertice Engine`  based on *openshfit/origin*
We'll use `MegamVertice2.0` to refer 2.0 release of `MegamVertice` based on *openshfit/origin*

## Motivation

*Openshift Origin* assumes that the user is created, updated in a 3rd party system. It assumes there exists an user and provides identityproviders to authenticate/authorize.

*Openshift Origin* doesn't have ability to create new users and manage them. But SaaS products need them.

*Openshift Origin* has ability to authenticate an user when available in a 3rd party system like `LDAP`, `OAuth providers`, `Basic auth using http via a flat file`.

For the *Onboard cloud project* we have externalized the auth to be managed using `Google` and have used the identity provider - `OAuthProvider` in openshift.

For MegamVertice2.0 we need ability to create, update users using an identityprovider plugin supported by *Openshift Origin*.

So once an user is available in a 3rd party system *Openshift Origin* has ability to authenticate and authorize api requests for that user.

As *Openshift Origin* supports `LDAP`, it will directly bind to the `LDAP` for authentication verification.

This mean we need to build an authentication layer that **Nilavu** & **MOOV** can communicate. **Nilavu** must be able to do user creation, updation using REST based API or gRPC. **MOOV** must be able to authenticate the user.

So we will build an **authserver** backed by LDAP accessed via a REST API. The REST API will talk the same JSON as used by gateway today. So nilavu  doesn't have to worry much.

**MOOV** based on *Openshfit Origin* will integrate directly to the identityprovider - LDAP.

## Solution Overview

The auth architecture link [refer slide6](https://docs.google.com/presentation/d/1tzkWbHu6RclA0QWnoEFy9HK0KmISdCjLNfv5QxwJ3Mg/edit?usp=sharing)

We focus on the usecases

1. How does an user onboardin Nilavu ?
2. How does MOOV authenticate an API request  ?
3. How do our third party systems (WHMCS) talk to MOOV ?


### 1. How does an user onboard in Nilavu ?

Nilavu will use the API and call the **authserver** which is backed by LDAP to create/update users. Nilavu will call **MOOV** API for  authentication.

### 2. How does MOOV authenticate an API request

**MOOV** uses LDAP bind operation using the `cn=admin,dc=megam,dc=org` distinguished admin userid of LDAP and its password.

Any API request **MOOV** receives will need the bearer token which is compared with what is available for that user in `oauthaccesstokens`.

If it didn't find anything then **MOOV** using the bind connection will do a search by `cn=email id`. Upon successful validation, a token is saved in API `oauthaccesstokens`.


### 3. How will third party systems (WHMCS) talk to MOOV

We will have *bots* based on **ServiceAccounts** for every 3rd party system integration. The **billerbot** in our case will handle billing related ops.

The billing design will be covered in a separate documentation.

## Detailed design

We wanted to pick another language that is native and other than *golang*, *scala*, *python*..

### Choice of language

We'll use [Rust lang](https://rust-lang.org) as its native and super fast.

Some of projects that use **Rust** are

- [Habitat.sh](https://habitat.sh)

An `Rust` based [authserver - Boomi](https://github.com/megamsys/boomi) will form the fulcrum for **MegamVertice2.0** user management.

We'll had originally intended to `OCaml` as its functional, native and super fast.

Some of projects that use OCaml are

- [Janestreet](https://www.youtube.com/watch?v=hKcOkWzj0_s)
- [Facebook Reactive](https://facebook.github.io/reason/)
- [Rust compiler in OCaml](https://github.com/rust-lang/rust/tree/ef75860a0a72f79f97216f8aaa5b388d98da6480/src/boot)
- [MirageOS](http://mirage.io/)
- [Irmin](https://github.com/mirage/irmin/tree/master/src/irmin-http)

The learning curve to use OCaml for newer members will be steep, there are no good community projects that help us to build *API* faster.

The project we found close enough was  [Irmin](https://github.com/mirage/irmin/tree/master/src/irmin-http)

### Rust authserver boomi

The purpose is clear as we have worked on [Onboard cloud - abcd project](https://github.com/megamsys/abcd) which used the OAuth technique.

The **Rust boomi authserver** is a simple server that accepts user creation, user updation backed by LDAP.

For detailed architecture [Refer slide-7](https://docs.google.com/presentation/d/1tzkWbHu6RclA0QWnoEFy9HK0KmISdCjLNfv5QxwJ3Mg/edit?usp=sharing) of Architecture v2.0.

This will be a REST based server (or) `gRPC`  based.  

The **Rust authserver boomi** requirements are:

#### 1. Secure TLS for system to system communication

We'll generate TLS certificate when the authserver starts and store it in **$MEGAM_HOME/boomi** path. The security key files will be named as **<boomi.key>** and **<boomi.pem>**

If `Nilavu` and the `Rust boomi` run in different machines then we have to make sure the **$MEGAM_HOME/boomi/keyfiles** are copied over to the `nilavu` machine in the directory mentioned above.


#### 2. API Protection

The user level multitenancy was handled historically used a cyptographic HMAC per user to access the API, (or) global service accounts (or) BasicAuth: userid/pw.  We had proctected every API request.

In this case we protect systems using `TLS`. The `login`, `account.show` are handled by `Openshift` API via the kubeclient gem.


#### 4. REST

Here are the API calls we'll cover, although we'll migrate to `gRPC` in the future (or) today.


| Verb | REST                     | Description                                                                                                 |
|------|--------------------------|-------------------------------------------------------------------------------------------------------------|
| GET  | /accounts/forgot/:email  | Initiate the process where an user forgot an email. Generate a unique token and store it in userPassword    |
| POST | /accounts/content        | Create a new user                    |
| POST | /accounts/update         | Modify an user                       |
| POST | /accounts/password_reset | Reset the password by doing a due-diligence verification of the password_reset_token sent with userPassword |


### Native: Account

`Native Account` means user management is handled in MegamVertice2.0.

Nilavu will have an `AuthDispatcher` which will perform REST calls to the `OCaml auth server`.

Nilavu must have its `User model` trimmed with the `not needed` variables. Please refer the table.

The `User model` must convert `integer` based edge comparison to `boolean based` flags to indicate `active, staged, blocked, suspended, approved`.

The below table is needed for the  internals of the `Rust boomi` which handles user management backed by `LDAP`.

This table may not be off much importance to nilavu, since the `AuthDispatcher` will act as a fascade to `User model` which will  feed the correct JSON as it uses today. We may have to trim some of the fields that are not used and which doesn't make sense.
Please pay attention to that.

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

The LDIF record. Here we create an user with email `kenji.kouda@jap.ai` in dc `megam.org`

##
# Entry 1: cn=kenji.kouda@jap.ai,dc=megam,dc=org
dn: cn=kenji.kouda@jap.ai,dc=megam,dc=org
cn: kenji.kouda@jap.ai
employeetype: admin
givenname: kenji
iphostnumber: 192.168.1.1
mail: kenji.kouda@jap.ai
objectclass: inetOrgPerson
objectclass: top
objectclass: shadowAccount
objectclass: ipHost
shadowexpire: 1
shadowflag: 1
shadowinactive: 0
sn: kouda
uid: kenjikouda
userpassword: {MD5}X03MO1qnZdYdgyfeuILPmQ==

```

## OAuth: Account

Upon authentication with the OAuth provider like `GitHub`, `Google`, we create an `Account` in the `authserver` if it doesn't exists. The rest of the workflow is as per `NativeAccount`

### Development setup

1. Run slapd (openldap)

```

docker run --name slapd --env LDAP_ORGANISATION="users" --env LDAP_DOMAIN="megam.org" \
--env LDAP_ADMIN_PASSWORD="chennai28v" --detach osixia/openldap:1.1.8

docker logs -f slapd

```

2. *optional* Run sladpui

slapdui is a web based ldap administration to manage slapd.

This step is optional and only needed in development, as `slapd` can be managed from command line.

```

LDAP_IP=$(docker inspect -f "{{ .NetworkSettings.IPAddress }}" sldapd)

$ echo $LDAP_IP
172.17.0.2

docker run --name slapdui -p 6443:443 \
       --env PHPLDAPADMIN_LDAP_HOSTS=$LDAP_IP \
       --detach osixia/phpldapadmin:0.6.12

```

3.  Open sladpui web administration

```

LDAP_ADMIN=$(docker inspect -f "{{ .NetworkSettings.IPAddress }}" ldapadmin)

$ echo $LDAP_ADMIN
172.17.0.3

```

Type [https://$LDAP_ADMIN:6443](https://$LDAP_ADMIN:6443) to manage LDAP accounts.

Login: **username** `cn=admin,dc=megam,dc=org` **password** `chennai28v`

You can see that an admin account exists.

4.  Openshift configuration

We'll have make sure Openshift runs with LDAPPasswordIdentityProvider. To do that,

Open the file `$OPENSHIFT_HOME/openshift.local.config/master/master-config.yaml`

```yaml

oauthConfig:
  alwaysShowProviderSelection: false
  assetPublicURL: https://192.168.1.11:8443/console/
  grantConfig:
    method: auto
    serviceAccountMethod: prompt
  identityProviders:
  - challenge: true
    login: true
    mappingMethod: claim
    name: "my-ldap-provider"
    provider:
      apiVersion: v1
      kind: LDAPPasswordIdentityProvider
      attributes:
        id:
        - dn
        email:
        - mail
        name:
        - cn
        preferredUsername:
        - uid
      bindDN: "cn=admin,dc=megam,dc=org"
      bindPassword: "chennai28v"
      insecure: true
      url: "ldap://172.17.0.2/dc=megam,dc=org?cn"              

```

Refer this link for the flag details. [LDAPProvider](https://docs.openshift.org/latest/install_config/configuring_authentication.html#LDAPPasswordIdentityProvider)


5. Setup an account manually

Open the `slapdui` as per `Step 4`

Just import another account from the below LDIF format. whose credentials are

username: rajt@code.in
password: password

```

The LDIF record. Here we create an user with email `rajt@code.in` and password `password` in dc `megam.org`

The `userpassword` is Base64 encoded version of md5 encrypted string. So in our case the text is "password" which was md5 encrypted and then Base64 encoded to produce `X03MO1qnZdYdgyfeuILPmQ==`

##
# Entry 1: cn=raj.t@code.in,dc=megam,dc=org
dn: cn=raj.t@code.in,dc=megam,dc=org
cn: raj.t@code.in
employeetype: admin
givenname: rajt
iphostnumber: 192.168.1.11
mail: rajt@code.ai
objectclass: inetOrgPerson
objectclass: top
objectclass: shadowAccount
objectclass: ipHost
shadowexpire: 1
shadowflag: 1
shadowinactive: 0
sn: thilak
uid: rajt
userpassword: {MD5}X03MO1qnZdYdgyfeuILPmQ==

```

6. Nilavu login in

Try logging in from the UI, once you are in. You can use the CLI to see the happenings on the **MOOV**

A token is produced.

```

sudo ./oc get oauthaccesstokens | grep kenji.kouda@jap.ai
SNoiBTIemHUfD46PrBt8YcYBlB0ObTlVAZbYawZC-EU   kenji.kouda@jap.ai   openshift-challenging-client   2017-06-12 17:17:02 +0530 IST   2017-06-13 17:17:02 +0530 IST   https://192.168.1.11:8443/oauth/token/implicit   user:full

```

An user is created.

```

sudo ./oc get users | grep kenji
kenji.kouda@jap.ai   dc5deaac-4f63-11e7-8bb8-848f69adc5b4   kenji.kouda@jap.ai   my-ldap-provider:cn=kenji.kouda@jap.ai,dc=megam,dc=org

```

## Production setup

1. Run openldap in TLS [section - Use auto-generated certificate](https://github.com/osixia/docker-openldap#tls)

2. A script that sets up the full process for authentication (containers, and LDAP configuration)

```

curl https://get.megam.io/auth megam.org testing4

#dc=megam.org
#admin password=testing4

```


# References

- [Book: LDAP System administration](http://shop.oreilly.com/product/9781565924918.do)
- [Designing LDAP schema](https://www.skills-1st.co.uk/papers/ldap-schema-design-feb-2005/ldap-schema-design-feb-2005.html)
- [LDAP Schemas, objectClasses](http://www.zytrax.com/books/ldap/ch3/)
