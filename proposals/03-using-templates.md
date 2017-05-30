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

Annotations fields to category (VirtualMachine, Application) Templates

| Fields       | Description                            |   Values |
|--------------|----------------------------------------|----------|
|annotations   |The detailed descrioptions and category fields | To category VirtualMachine or Application Templates
|objects       |The objects are group of kinds, every template have atleast one kind   | objests like Pod, DeploymentConfig, Service
|nodeSelector  |Used to select nodes based on key value match expressions
|service       |Service is an object that expose ports and endpoint |
|parameters    |The parameters are used feed values on run time of template | Like name of virutal machine



| Fields       | Description                            |   Values |
|--------------|----------------------------------------|----------|
|display-name  |The name of the marketplace item (Ghost)|  The name of the product Ghost                       |
|provided_by   |The provider of the product             |  The currently supported providers *vertice, bitnami*|
|cattype       |Category type to show under marketplaces|  The prepackaged apps can be added in COLLABORATION  |
|catorder      |A numeric value show the order in which the category shall be shown| The category the value for COLLABORATION is 6|
|image         |The name of picture to show the user     |ghost.png|
|url           |The product website for more information |http://ghost.org|





The template that are run as virtualmachines will have **cloud-init** inject data and hence the flags as per the link [here](https://github.com/Mirantis/virtlet/blob/fcf4615a65a13470c73ac44f8df316824e9c73e1/docs/design-proposals/cloud-init-data-generation.md) will have to be in the templates.

The different `category` templates are elaborated here:

### Category: Virtual machines

A sample template for Ubuntu is

```
---
kind: Template
apiVersion: v1
metadata:
  name: ubuntu
  annotations:
    openshift.io/display-name: Ubuntu
    description: |-
      Ubuntu is a Debian-based Linux operating system. Xenial Xerus is the Ubuntu codename for version 16.04 LTS of the Ubuntu Linux-based operating system., see http://www.ubuntu.com/server.

      WARNING: Any data stored will be lost upon pod destruction.
    iconClass: icon-machine
    tags: torpeto,ubuntu
    template.openshift.io/long-description: This template provides a standalone OpenNebula
      master server with a database created.  The database is not stored on persistent storage,
      so any restart of the service will result in all data being lost.
    template.openshift.io/provider-display-name: Megam Systems.
    template.openshift.io/documentation-url: https://docs.megam.io
    template.openshift.io/support-url: https://github.com/megamsys/support
    template.openshift.io/url: http://www.ubuntu.com/server
    template.openshift.io/provided_by:  vertice
    template.openshift.io/image: ubuntu.png
    template.openshift.io/cattype: TORPEDO
    template.openshift.io/catorder: 1
    template.openshift.io/os: ubuntu

message: |-
  The following service(s) have been created in your project: ${TORPEDO_NAME}.

         Username: ${USER}
         Password: ${PASSWORD}
   Connection URL: http://${TORPEDO_NAME}-${DOMAIN}/

  For more information about using this template, including OpenShift considerations, see https://github.com/megamsys/kubeshift.
labels:
  template: ubuntu
objects:
- kind: DeploymentConfig
  apiVersion: v1
  metadata:
    name: "${TORPEDO_NAME}"
    annotations:
      kubernetes.io/target-runtime: virtlet
    creationTimestamp:
  spec:
    restartPolicy: Never
    replicas: 1
    selector:
      name: "${TORPEDO_NAME}"
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: extraRuntime
              operator: In
              values:
              - virtlet
    containers:
    - name: "${TORPEDO_NAME}"
      image: "${IMAGE_NAME}-${VERSION}"
    volumes:
    - name: test
      flexVolume:
        driver: "virtlet/flexvolume_driver"
        options:
          type: nocloud
          metadata: |
            instance-id: "${VM_ID}"
            local-hostname: "${TORPEDO_NAME}-${DOMAIN}"
          userdata: |
            #cloud-config
            users:
            - name: root
              ssh-authorized-keys:
              - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCaJEcFDXEK2ZbX0ZLS1EIYFZRbDAcRfuVjpstSc0De8+sV1aiu+dePxdkuDRwqFtCyk6dEZkssjOkBXtri00MECLkir6FcH3kKOJtbJ6vy3uaJc9w1ERo+wyl6SkAh/+JTJkp7QRXj8oylW5E20LsbnA/dIwWzAF51PPwF7A7FtNg9DnwPqMkxFo1Th/buOMKbP5ZA1mmNNtmzbMpMfJATvVyiv3ccsSJKOiyQr6UG+j7sc/7jMVz5Xk34Vd0l8GwcB0334MchHckmqDB142h/NCWTr8oLakDNvkfC1YneAfAO41hDkUbxPtVBG5M/o7P4fxoqiHEX+ZLfRxDtHB53 me@localhost
            ssh_pwauth: True
parameters:
- name: VERSION
  displayName: Ubuntu Xenial
  description: Version of Ubuntu image to be used (16.04).
  value: '16.04'
- name: VERSION
  displayName: Ubuntu Trusty
  description: Version of Ubuntu image to be used (14.04).
  value: '14.04'
- name: IMAGE_NAME
  displayName: Ubuntu_Xenial
  description: Image name that is stored in image store server.
  value: 'virtlet/cloud-images.ubuntu.com/xenial/current/xenial-server'
- name: TORPEDO_NAME
  description: Name of instance.
  value: 'blue-sky-5624'
- name: DOMAIN
  description: domain name of instance.
  value: 'megambox.com'
- name: VM_ID
  description: Unique id for instance.
  value: '1235-254'
```

Parameter will be update when process based on User Choice for example

If your picks a vertion in two version parameters that Vertion parameter only used while process

TORPEDO_NAME will be updated dynamically each process

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
