# Multi tenant authentication: 2.0

## Motivation

As our 2.0 is based on *openshift origin/kubernetes* which has ability to authenticate and authorize api requests. It doesn't provide multi-tenant ability with user management, this means we need to an authentication layer that MegamVertice can communicate.

For the *Onboard cloud project* we have externalized the auth to be managed using **Google**.

Here we will have to build an **authserver** backed by LDAP using the same API that nilavu  uses. MegamVertice will integrate directory to the ldap.

## Solution Overview

The auth architecture link [here](https://docs.google.com/presentation/d/1tzkWbHu6RclA0QWnoEFy9HK0KmISdCjLNfv5QxwJ3Mg/edit?usp=sharing)

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
