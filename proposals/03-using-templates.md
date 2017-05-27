# Managing Templates: 2.0

## Motivation

The current 1.5 marketplace needs to revamped to an advanced Hybrid cloud marketplace with customers X sharing their contribution and pushing their contribution to the global marketplace.

The move to *openshift origin/kubernetes* warrants us to redesign the **marketplaces** based on **templates** as managed by **openshift/origin**

# Requirements

* Configure prebuilt templates

* Deploy & Manage templates

## Configure prebuilt templates

Once MegamVertice is installed the prebuilt templates will be available in the container `megam:vertice` or from the templates  [here](https://github.com/megamsys/abcd/tree/master/examples/vertice)

```

oc login

$ oc create -f <filename> -l name=otherLabel

```

The `otherLabel` can be labels like `marketplace=dev` which can separate dev to prod.

It appears that templates are loaded for a file only by one. Read this [link](https://docs.openshift.org/latest/dev_guide/templates.html#creating-from-templates-using-the-cli) to see if the templates can be loaded from a directory.

The template **must have** the following

| Name  	| Description  	|
|---	    |---	          |
|   	    |   	          |
|   	    |              	|
|   	    |             	|  


The template that are run as virtualmachines will have **cloud-init** inject data and hence the flags as per the link [here](https://github.com/Mirantis/virtlet/blob/fcf4615a65a13470c73ac44f8df316824e9c73e1/docs/design-proposals/cloud-init-data-generation.md) will have to be in the templates.

The different `category` templates are elaborated here:

### Category: Virtual machines

A sample template for Ubuntu is

```

```

### Category: Apps

A sample template for custom application

#### Ruby

[Link](https://docs.openshift.org/latest/using_images/s2i_images/ruby.html#using-images-s2i-images-ruby)

```

```

#### Node.js

```

```

#### PHP

```

```

### Category: Services

A sample template for services

- [postgresql](https://docs.openshift.org/latest/using_images/db_images/postgresql.html#using-images-db-images-postgresql)
- [mysql](https://docs.openshift.org/latest/using_images/db_images/postgresql.html#using-images-db-images-postgresql)
- [mongodb](https://docs.openshift.org/latest/using_images/db_images/mongodb.html#using-images-db-images-mongodb)

can be used to make services

```

```

### Category: Containers

A sample template for a containers [link](https://docs.openshift.org/latest/using_images/db_images/mysql.html#using-images-db-images-mysql)

```

```

## List templates in the Marketplaces

User when clicks marketplaces an api call is made to https://0.0.0.0:8443/oapi/v1/templates.

The **marketplaces** will call the modified *ApiDispatcher(marketplaces)* using the *kubeclient* gem.

#### Create SSH

User selects SSH page and list all existing ssh keys if exist.

Creates Generate ssh public and private keys and send data parameter to megam_api dispatcher

ApiDispatcher have to ensure the data with values of ssh-privatekey and ssh-publickey

  ```  {
    "apiVersion":"v1",
    "data":{
        "ssh-privatekey":"LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb2dJQkFBS0NBUUVBMVZ0S0lw....",
        "ssh-publickey":"c3NoLXJzYSBBQUFBQjNOemFDMXljMkVBQUFBREFRQ....",
    },
    "kind":"Secret",
    "metadata":{
        "name":"ssh",
        "namespace":"test",
    },
  }
  ```

#### Upload SSH

User clicks upload on sshkeys page

Upload public and private keys into data parameter to megam_api dispatcher

ApiDispatcher have to ensure the data with values of ssh-privatekey and ssh-publickey

#### List Secrets

List secrets GET using bellow request.

ApiDispatcher send parsed Megam ruby object.

Show ssh keys

#### Launcher moves to Step 2

List Templates as per `List marketplace` 
