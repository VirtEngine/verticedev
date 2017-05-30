# Add software in Marketplace

Here are the steps to configure your site’s marketplace manually.

Hopefully your have customized your image and taken a snapshot from MegamVertice.

## Step 1: ssh into your MegamVertice Master

```
$ ssh <userid>@megamvertice.master
```

## Step 2: cqlsh into your cassandra database

Lets say we want to add the product “Ghost” blog into your marketplace.


| Fields       | Description                            |   Values |
|--------------|----------------------------------------|----------|
|settings_name |The provider of the product             |  The currently supported providers *vertice, bitnami*|
|cattype       |Category type to show under marketplaces|  The prepackaged apps can be added in COLLABORATION  |
|flavor        |The name of the marketplace item (Ghost)|  The name of the product Ghost                       |
|catorder      |A numeric value show the order in which the category shall be shown| The category the value for COLLABORATION is 6|
|image         |The name of picture to show the user     |ghost.png|
|url           |The product website for more information |http://ghost.org|


```
$ cqlsh 127.0.0.1 -u vertadmin -p vertadmin
```

If the above didn’t work then try,
$ cqlsh <ip_address> -u vertadmin -p vertadmin

```
> use vertice;
>

INSERT INTO marketplaces (settings_name, cattype, flavor, image, catorder, url, json_claz, envs, options, plans) values (vertice, 'COLLABORATION', 'Ghost',  'ghost.png', '6', 'https://bitnami.com/stack/ghost', 'Megam::MarketPlace' , ['{"key":"http-port", "value":"80"}','{"key":"path", "value":"/"}'], ['{"key":"oneclick", "value":"yes"}','{"key":"bitnami_username", "value":"team4megam"}','{"key":"bitnami_password", "value":"team4megam"}','{"key":"bitnami_url", "value":"https://s3-ap-southeast-1.amazonaws.com/megampub/bitnami/Blog/bitnami-ghost-0.7.8-0-linux-x64-installer.run"}'], {'0.7.8':'Ghost is built on Node.js and is extremely easy to configure and customize.'});
```

## Step 3: Copy the product pictures

Make sure you have two product pictures named as ghost.png with sizes (187 x 100), (32 x 32)

```
$ cd /var/www/virtenginenilavu

$ cp ghost.png /var/www/virtenginenilavu/public/brands *size 32 x 32

$ cp ghost.png /var/www/virtenginenilavu/public/brands/saas *size 187 x 100
```

## Step 4: Restart VirtEngine UI (virtenginenilavu)

```
$ cd /var/www/virtenginenilavu

$ rake assets:clean

$ sv stop unicorn

$ sv start unicorn
```

## Step 5: Open up OpenNebula

Make sure you have the oneadmin:password handy

Open the link http://<master_ip>:9869

Search the snapshot you did with MegamVertice and rename it to `ghost`

Hurray, you are all set,  watch your customers launch their favorite blog `ghost` and ring you $$$
