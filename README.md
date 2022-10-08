# Installation of Docker ERPNext CustomApps
# Let's get Start

**Configure A records:**
* Add A record for portainer.excelbd.com pointing the public IP
* Add A record for traefik.excelbd.com pointing the public IP
* Add A record for erp.com pointing the public IP
* Add A record for rma.excelbd.com pointing the public IP
* Add A record for warranty.excelbd.com pointing the public IP

**Install Ubuntu Server with Docker Swarm Mode**

```
//Use root user 
sudo su
apt update && apt upgrade -y
apt install openssh-server
ufw allow ssh
systemctl status ssh
init 6
```
**Starting the docker swarm installation**
```
export USE_HOSTNAME=docker.excelbd.com
echo $USE_HOSTNAME > /etc/hostname
hostname -F /etc/hostname
apt update && apt upgrade -y
apt install curl
curl -fsSL get.docker.com -o get-docker.sh
CHANNEL=stable sh get-docker.sh
rm get-docker.sh
docker swarm init
docker ps
//Post-installation steps for Linux: Manage Docker as a non-root user
sudo groupadd docker
sudo usermod -aG docker $USER
//Log out and log back in so that your group membership is re-evaluated.
```
**Traefik Proxy with HTTPS**
```
docker network create --driver=overlay traefik-public
export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')
docker node update --label-add traefik-public.traefik-public-certificates=true $NODE_ID
export EMAIL=admin@example.com
export DOMAIN=traefik.excelbd.com
export USERNAME=admin
//AVOID all sorts of characters [~!@#$%, etc.] as a pass
export PASSWORD=Admin123
export HASHED_PASSWORD=$(openssl passwd -apr1 $PASSWORD)
//Deploy the stack
curl -L dockerswarm.rocks/traefik-host.yml -o traefik-host.yml
docker stack deploy -c traefik-host.yml traefik
docker stack ps traefik
```

**Portainer web user interface**
```
export DOMAIN=portainer.excelbd.com 
export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')
docker node update --label-add portainer.portainer-data=true $NODE_ID
//Create the Docker Compose file
curl -L dockerswarm.rocks/portainer.yml -o portainer.yml
//Deploy the stack
// to update/install the portainer to the community edition, edit the yaml and change portainer image tag to portainer/portainer-ce and deploy the stack.
docker stack deploy -c portainer.yml portainer
docker stack ps portainer
```

# Login to portainer\[setup password\]. Follow the rest of the steps using Portainer console.

**Deploy MariaDB Replication**

Stacks > Add Stacks > `frappe-mariadb`

Add mariadb configuration from [frappe mariadb yml](https://github.com/excelbd/Excel-ERP-Production-Setup/wiki/frappe-mariadb-yaml)

**Deploy Frappe/ERPNext**

Stacks > Add Stacks > `frappe-bench-v12`

Add the yaml configuration from [ frappe bench v12 yaml](https://github.com/excelbd/Excel-ERP-Production-Setup/wiki/frappe-bench-v12-yml)

**Use environment variables:**

* FRAPPE_VERSION= v12.13.0
* EXCEL_ERPNEXT_VERSION= v12.2.0
* MARIADB_HOST= frappe-mariadb_mariadb-master
* SITES variable is list of sites in back tick and separated by comma
* SITES= `` `erp.excelbd.com` `` **{Note: Don't forgot to add `` }**


**Now you can see some of container are `ready` `asigned` and `restarting`**

**Let's Solve this first**

* Go to python container console & connect with `root`

![image](https://user-images.githubusercontent.com/92359442/185776847-90d52297-2ce1-4c24-847a-c16aad73251a.png)

* After Connect with root you will be redirect to this path `root@49290287e8dd:/home/frappe/frappe-bench/sites#`

* Now Run this command one by one

```
apt update
apt install sudo
apt install nano
```

* Create a new file `apps.txt` with this name
* `nano apps.txt`
* Past this on `apps.txt`
```
frappe
excel_erpnext
erpnext
```


* Create a new file `common_site_config.json` with this name 
* `nano common_site_config.json`
* Past this on `common_site_config.json` file
```
{
 "db_host": "frappe-mariadb_mariadb-master",
 "db_port": 3306,
 "maintenance_mode": 0,
 "pause_scheduler": 1,
 "redis_cache": "redis://redis-cache:6379",
 "redis_queue": "redis://redis-queue:6379",
 "redis_socketio": "redis://redis-socketio:6379",
 "socketio_port": 9000
}
```
* Then save and exit from console

**Yeah! Can you see? All container are now running fine. Let's move forward**

# ERPNext Install

Once all stacks are running then go to python container console with `root`

![image](https://user-images.githubusercontent.com/92359442/172530794-82aa2be9-1720-4b98-b47d-da4f1527316b.png)

![image](https://user-images.githubusercontent.com/92359442/172530931-51fbe643-71b5-4927-95e8-1834553b9ed8.png)

![image](https://user-images.githubusercontent.com/92359442/172531043-3b4d8a90-eacf-4f4b-94c9-53efbe5fbf33.png)

**Back to `frappe-bench` directory**

`cd ..`

**Create new site**

`bench new-site etlerp.arcapps.org --mariadb-root-password admin2022 --admin-password admin2022 --no-mariadb-socket --force`

**Install ERPNext**

`bench --site etlerp.arcapps.org install-app erpnext`

**Install Excel ERPNext**

`bench --site etlerp.arcapps.org install-app excel_erpnext`

**Brows your site** `https://etlerp.arcapps.org/`

# Getting started with RMA Server \[Custom Apps Installations\]

**Add MongoDB Stack**

Stacks > Add Stacks > `mongodb`

Add Configuration file from [mongodb yml](https://github.com/excelbd/Excel-ERP-Production-Setup/wiki/mongodb-yml)

Let the MongoBD Database run completely and from the terminal:

![image](https://user-images.githubusercontent.com/92359442/172532650-18f510c0-14d9-424e-a37b-9dd1e2bfa8d6.png)

Add `cache-db` database credentials:

Execute following command in mongodb container. Change the SecretPassword to desired password.

```
mongo cache-db --port 27017 -u root -p SecretPassword --authenticationDatabase admin --eval "db.createUser({user: 'cache-db', pwd: 'SecretPassword', roles:[{role:'dbOwner', db: 'cache-db'}]});"
```

# Add RMA-SERVER Stack

Stacks > Add Stacks > `rma-server`

Add Configuration file from [RMA SERVER Stack yaml](https://github.com/excelbd/Excel-ERP-Production-Setup/wiki/RMA-SERVER-Stack-yml)

Environment variables:

* DB_NAME=rma-server
* RMA_SERVER_VERSION=2.0.0-alpha.0
* RMA_WARRANTY_VERSION=2.0.0-alpha.0
* RMA_FRONTEND_VERSION=2.0.0-alpha.0
* DB_USER=rma-server
* DB_PASSWORD=SecretPassword
* CACHE_DB_NAME=cache-db
* CACHE_DB_USER=cache-db
* CACHE_DB_PASSWORD=SecretPassword
* SITE=rma.excelbd.com
* WARRANTY_SITE=warranty.excelbd.com

# Restore ERPNext and RMA-Server

To restore from databases, first, we have to install-configure a fresh-server with the Frappe Stacks, RMA-Server Stacks, etc. till Add Site Container. We don't need to add the site container as restore will create the site and everything in it. Just need to change the DNS management.

Now, the site name should be already included while installing frappe-bench stack. But if we're planning to restore to different site name, then later we will need to change the site name and configurations.

Before getting started, on python container terminal, on bench directory, run this

**Give Permission**

```
cd frappe-bench
chown -R frappe:frappe sites
```

**Restore** `mariadb` Database for ERPNext

Download mariadb backup from asw s3 

```
wget https://files-for-excel-bd.s3.ap-southeast-1.amazonaws.com/20220612_060135/20220612_060135-testerp_excelbd_com-database.sql.gz

```

It will take some time to Complete download successfully

**Restore downloaded database**

```
bench --site etlerp.arcapps.org --force restore 20211215_060004-erp_excelbd_com-database.sql.gz
```

It will take long time to restore the backup keep patient and wait until database restore successfully completed.

Now brows the site with your main ERP credentials.

# Restore RMA-Server

**Prerequisite**

Go to MongoDB container terminal and run.

```
cd /opt
mkdir restore
cd ./restore
```
Setup AWS CLI on local Windows CMD and copy the S3 bucket URL for MongoDB backup file and run

`aws s3 presign s3://mondbodb_database_backup_file_url`

This will generate a Public URL for the backup file. Now, from MongoDB terminal, run

```
curl https://files-for-excel-bd.s3.ap-southeast-1.amazonaws.com/20220612000013.tar.gz --output dump.tar.gz
tar xvfz dump.tar.gz
rm -fr /dump/admin
mongorestore -u root -p $MONGODB_ROOT_PASSWORD --authenticationDatabase admin dump
```
This will restore everything from RMA-Server and the portal/warranty site should be now available and point to old site. Warranty.excelbd.com/api/info will show the details.

Now we have to change the setups to integrate the apps again as we changed the site names, otherwise, everything should start working just fine.

Go to ERPNext oAuth Client list, Social Login Key list and change everything to new site/app names. And to change API info, go to MongoDB ternmial and run

```
mongo -u root -p $MONGODB_ROOT_PASSWORD --authenticationDatabase admin
show dbs;
use rma-server;
db.server_settings.findOne()
```
# Once done everything we need to configure some basic changes

* **Delete all Webhook**
* **Update RMA `server_settings` from mongodb with your credentials**

Run the command, change the variables as required. Before running this, generate the serviceAccount ApiSecret again for the account and replace it. As we are restoring database, frontend/backend client id doesn't change, service account already exists, check the erpnext site name and then run this.

```
db.server_settings.updateOne({},{
	$set:{
        "_id" : ObjectId("600c02e7165b02000df96bb1"),
        "service" : "rma-server",
        "uuid" : "939d9d3c-d4f6-4dad-8192-dfe4dd64e7d2",
        "appURL" : "https://etlportal.arcapps.org",
        "warrantyAppURL" : "https://etlwarranty.arcapps.org",
        "frontendClientId" : "1cae8c4f76",
        "backendClientId" : "fe6df0a17b",
        "serviceAccountUser" : "bot@excelbd.com",
        "serviceAccountSecret" : "Admin@123#",
        "authServerURL" : "https://etlerp.arcapps.org",
        "serviceAccountApiKey" : "403dfa271ac2021",
        "serviceAccountApiSecret" : "4ee812fbd1f8acd",
        "profileURL" : "https://etlerp.arcapps.org/api/method/frappe.integrations.oauth2.openid_profile",
        "authorizationURL" : "https://etlerp.arcapps.org/api/method/frappe.integrations.oauth2.authorize",
        "tokenURL" : "https://etlerp.arcapps.org/api/method/frappe.integrations.oauth2.get_token",
        "revocationURL" : "https://etlerp.arcapps.org/api/method/frappe.integrations.oauth2.revoke_token",
        "scope" : [
                "all",
                "openid"
        ],
        "webhookApiKey" : "b9fcc4c8c1be9b6681c6ec20e2c9935ef2f8c450b9728f7c7e00bbfc38076a00f3740cdbad884c97d64db24389e15dae0156590ee840ffa54d3348fd827d79d0",
        "timeZone" : "Asia/Dhaka",
        "defaultCompany" : "Excel Technologies Ltd.",
        "sellingPriceList" : "Standard Selling",
        "debtorAccount" : "10203 - Accounts Receivable - ETL",
        "transferWarehouse" : "Work In Progress - ETL",
        "validateStock" : true,
        "footerImageURL" : "https://etlerp.arcapps.org/files/SINV-Footer.jpg",
        "footerWidth" : 530,
        "headerImageURL" : "https://etlerp.arcapps.org/files/ETL-header.jpg",
        "headerWidth" : 500,
        "brand" : {
                "faviconURL" : "https://etlerp.arcapps.org/files/Fav-Icon.png"
        },
        "backdatedInvoices" : true
	}
})
```

**Setup Webhooks**

Set the all Webhook which we deleted previously from ERP With Send a `POST` request by RESTER/Postman 

POST `url : https://etlportal.arcapps.org/api/settings/v1/setup_webhooks` Headers

```
{
  "Authorization": "Bearer <system manager token>"
}

```

This will create Webhooks in ERPNext synchronously. If it fails, delete the created webhooks from ERPNext and make this request again.


**Congratulations!!! You're good to go.... Everything completed successfully**

