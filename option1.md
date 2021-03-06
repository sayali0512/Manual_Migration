
## Moodle Manual  Migration
This document explains how to migrate Moodle from an on-premises deployment to Azure. For each of the steps, you have two approaches provided: one that lets you use Azure Portal and one that lets you accomplish the same tasks on a command line using Azure CLI.

### Option 1: Migrating Moodle with an ARM templates 
-   Migration of Moodle with an ARM template:  creates the infrastructure in Azure.
-  Once the infrastructure is created, the Moodle software stack and associated dependencies are migrated.

## Prerequisites

-   If the versions of the software stack deployed on-premises are lagging with respect to the versions supported in this guide, the expectation is that the on-premises
    versions will be updated/patched to the versions listed in this guide.
-   Must have access to the on-premise infrastructure to take backup of Moodle deployment and configurations (including DB configurations).
-   Need an Azure subscription and the Azure Blob storage created before migration.
-   Have [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) and [AzCopy](https://docs.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-v10) handy.
-   This migration guide supports the following software versions:
    -   Ubuntu 16.04 LTS
    -   Nginx 1.10.3 or Apache 2.4
    -   MySQL 5.6, 5.7 or 8.0 database server (This guide uses [Azure Database for MYSQL](https://azure.microsoft.com/en-us/services/mysql/))
    -   PHP 7.2, 7.3, or 7.4
    -   Moodle 3.8 & 3.9

## Migration Approach

-   Migration of Moodle application to Azure is broken down in the following three stages:
    -   Pre-migration tasks
    -   Actual migration of the application
    -   Post-migration tasks


-   **Pre Migration**
    
    -   Data Export from on-premises to Azure involves the following tasks:
        -   Install [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
        -   Have an [Azure subscription](https://azure.microsoft.com/en-us/) handy
        -   Create a Resource Group inside Azure
        -   Create a Storage Account inside Azure
        -   Backup all relevant data from on-premises infrastructure
        -   Ensure the on-premises database instance has mysql-client installed
        -   Copy backup archive to Blob storage on Azure

-   **Migration**
    
    -   Actual migration tasks that involve the application and all data:
        - Deploy infrastructure on Azure using [Moodle ARM template](https://github.com/Azure/Moodle) 
        - Copy over the backup archive (Moodle data) to the Moodle controller instance from the ARM deployment 
        - Setup Moodle controller instance and worker nodes
        - Data migration tasks
       
-   **Post Migration**
    
    -   Post migration tasks that include application configuration and certificate installs:
        -   Update general configuration (e.g. log file destinations)
        -   Update any cron jobs / scheduled tasks
        -   Configuring certificates
        -   Restarting servers

## Pre Migration

-   **Data Export from on-premises to Azure Cloud:**
    -   **Install Azure CLI**
        -   Install Azure CLI on a host inside the on-premises infrastructure for all Azure related tasks
            ```
            curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
            ```
        -   Now login into your Azure account
            ```
                az login
            ```
        -   Azure CLI will quite likely launch an instance or a tab inside your default web-browser and prompt you to login to Azure using your Microsoft Account.
        -   If the above browser launch does not happen, open a browser page at  [https://aka.ms/devicelogin](https://aka.ms/devicelogin)  and enter the authorization code displayed in your terminal.
        -   Sign in with your account credentials in the browser.
        -   Sign in with credentials on the command line
            ```
                az login -u <username> -p <password>
            ```
    -   **Create Subscription:**
        -   If you do not have a subscription already present, you can choose to [create one within the Azure Portal]((https://ms.portal.azure.com/#blade/Microsoft_Azure_Billing/SubscriptionsBlade) or opt for a [Pay-As-You-Go](https://azure.microsoft.com/en-us/offers/ms-azr-0003p/).


    -   **Create Resource Group:**
        -   Once you have a subscription handy, you will need to [create a Resource Group](https://ms.portal.azure.com/#create/Microsoft.ResourceGroup).
            ```
                # Alternatively you can use the Azure CLI command to create a resource group
                az deployment group create --resource-group <resource-group-name> --template-file <path-to-template>
            
            ```
   
    -   **Create Storage Account:**

        -   The next step would be to [create a Storage Account]((https://ms.portal.azure.com/#create/Microsoft.StorageAccount) in the Resource Group you've just created
        -   Create a  [storage account](https://ms.portal.azure.com/#create/Microsoft.StorageAccount) and set the Account 
            ```
                az storage account create -n storageAccountName -g resourceGroupName --sku Standard_LRS --kind StorageV2 -l eastus2euap -t Account
            ```
        -   Once the storage account is created, this can be used as the interim destination for the on-premises backup
    
    -   **Backup of on-premises data:**
        -   Take backup of on-premises data such as moodle, moodledata, configurations and database backup file to a a single directory. The following illustration should give you a good idea: https://github.com/asift91/Manual_Migration/blob/master/images/folderstructure.png
        -   moodle and moodledata
            -  The moodle directory consists of site HTML content and moodledata contains Moodle site data
        -   configuration
            -   Copy the php configuration files such as php-fpm.conf, php.ini, pool.d and conf.d folder to phpconfig folder under the configuration directory.
            -   Copy the ngnix configuration such as nginx.conf, sites-enabled/dns.conf to the nginxconfig directory under the configuration directory.
            -   If the web-server used is Apache instead, copy all the relevant configuration for Apache to the configuration directory.
        -   create a backup of database
            -   If you do not have mysql-client installed on the database instance, now would be a good time to do that.
                
                ```
                    sudo -s
                    sudo apt install mysql-client
                    mysql -u dbUserName -p
                    # After the above command user will prompted for database password
                    mysqldump -h dbServerName -u dbUserId -pdbPassword dbName > /path/to/location/database.sql
                    # Replace dbServerName, dbUserId, dbPassword and bdName with on-premises database details
                
                ```
        -   Create an archive (tar.gz format) of the backup folder
            ```
                tar -zcvf moodle.tar.gz <source/folder/name>
            ```
            
    -   **Copy Archive file to Blob storage**
        -   Copy the on-premises archive file to blob storage using AzCopy
            -   To use AzCopy user should a generate SAS Token first.
            -   Go to the created Storage Account Resource and navigate to Shared access signature in the left panel.
            -   Select the Container checkbox and set the start, expiry date of the SAS token. Click on "Generate SAS and Connection String".
            -   copy the SAS token for further use.
            
                ```
                    az storage container create --account-name <storageAccontName> --name <containerName> --sas-token <SAS_token>
                    sudo azcopy copy '/path/to/location/moodle.tar.gz' 'https://<storageAccountName>.blob.core.windows.net/<containerName>/<dns>/<SAStoken>
                ```
            -   Now, you should have a copy of your archive inside the Azure blob storage account

## Migration

### Deploying Azure infrastructure using ARM templates
- When using an ARM template to deploy infrastructure on Azure, you have a couple of available options
    - A pre-defined deployment size using one of the four pre-defined Moodle sizes. 
    - A fully configurable deployment that provides gives more flexibility and choice[around deployments](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2FMoodle%2Fmaster%2Fazuredeploy.json)
- The four predefined options are Minimal, Short-to-Mid, Large and Maximal. These are available on [GitHub repository](https://github.com/Azure/moodle).
    - [Minimal](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2FMoodle%2Fmaster%2Fazuredeploy-minimal.json): This deployment will use NFS, MySQL, and smaller auto scale web frontend VM sku (1 core) that will give faster deployment time (less than 30 minutes) and requires only 2 VM cores currently that will fit even in a free trial Azure subscription.  
    - [Small to Mid](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2FMoodle%2Fmaster%2Fazuredeploy-small2mid-noha.json): Supporting up to 1000 concurrent users. This deployment will use NFS (no high availability) and MySQL (8 vCores), without other options like elastic search or Redis cache.  
    - [Large (high availability)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2FMoodle%2Fmaster%2Fazuredeploy-large-ha.json): Supporting more than 2000 concurrent users. This deployment will use Azure Files, MySQL (16 vCores) and Redis cache, without other options like elastic search.  
    - [Maximum](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2FMoodle%2Fmaster%2Fazuredeploy-maximal.json): This maximal deployment will use Azure Files, MySQL with highest SKU, Redis cache, elastic search (3 VMs), and pretty large storage sizes (both data disks and DB).
- To deploy any of the predefined size template click on the launch option. 
    -   For Moodle architecture diagram [click here](https://github.com/asift91/Manual_Migration/blob/master/images/stack_diagram.png).
- It will redirect to Azure Portal where user need to fill mandatory fields such as Subscription, Resource Group, SSH key, Region. 
![custom_deployment](images/customdeployment.png)
- Click on purchase to start the deployment of Moodle on Azure. Link for pricing [calculator]( https://azure.microsoft.com/en-us/pricing/calculator/ )
- The deployment will install  supported Infrastructure and Moodle
    - Moodle version: 3.8, 3.9 and 3.5  
    - Webserver: nginx or apache2 
    - Nginx version: 1.10.0
    - PHP version: 7.4, 7.3 or 7.2 
    - Database server type: MySQL  
    - MySQL version: 5.6, 5.7 and 8.0 
    - Ubuntu version: 16.04-LTS  
 
 <details>
  <summary>When the ARM template is used, the following resources are created within Azure (click to expand!)</summary>
  

- **Network Template:** Network Template will create virtual network,Network Security Group, Network Interface, subnet, Public IP, Load Balancer/App gateway and Redis cache etc. 
     - Creates a virtual network with string as name , apiVersion, Location and DNS server name.
     - The AddressSpace that contains an array of IP address ranges that can be used by subnets
   
    - **Virtual network:** An Azure Virtual Network is a representation of your own network in the cloud. It is a logical isolation of the Azure cloud dedicated to your subscription. When you create a VNet, your services and VMs within your VNet can communicate directly and securely with each other in the cloud. More information on Virtual Network [click here](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview).
    - **Network Security Group:** A network security group (NSG) is a networking filter (firewall) containing a list of security rules allowing or denying network traffic to resources connected to Azure VNets. For more information [click here](https://docs.microsoft.com/en-us/azure/virtual-network/security-overview).
    -   **Network Interface:** A network interface enables an Azure Virtual Machine to communicate with internet, Azure and on-premises resources.[click here](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-netwAork-interface)
    - **Subnet:** A subnet or subnetwork is a smaller network inside a large network. By default, an IP in a subnet can communicate with any other IP inside the VNET. More information on Subnet [click here](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-manage-subnet). 
    - **Public IP:** Public IP addresses are used to communicate Azure resources to the Internet. The address is dedicated to the Azure resource. More information on Public IP [click here](https://docs.microsoft.com/en-us/azure/virtual-network/public-ip-addresses#:~:text=Public%20IP%20addresses%20enable%20Azure,IP%20assigned%20can%20communicate%20outbound). 
    - **Load Balancer:** It is an efficient distribution of network or application traffic across multiple servers in a server farm. Ensures high availability and reliability by sending requests only to servers that are online. More information on Load balancer  [click here](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/tutorial-load-balancer#:~:text=An%20Azure%20load%20balancer%20is,traffic%20to%20an%20operational%20VM). 
    - ***Note***: Any of the 4 pre-defined template will deploy Azure Load Balancer,only in Fully Configurable deployment user has choice to choose App Gateway instead of Load Balancer,
    -  **Azure Application Gateway**: It is a web traffic load balancer that enables you to manage traffic to your web applications. Application Gateway can make routing decisions based on additional attributes of an HTTP request, for example URI path or host headers. More information on App Gateway [click here](https://docs.microsoft.com/en-us/azure/application-gateway/overview). 
    - **Redis Cache:** Azure Cache for Redis provides an in-memory data store based on the open-source software Redis. Redis improves the performance and scalability of an application that uses on backend data stores heavily. It is able to process large volumes of application request by keeping frequently accessed data in the server memory that can be written to and read from quickly. [click here](https://docs.microsoft.com/en-us/azure/azure-cache-for-redis/cache-overview). 

- **Storage Template:**  
    -  storage account  template will create a storage account  with FileStorage Kind and Premium LRS replication, Size of 1TB. For more example[click here](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-overview). 
    -   As per the predefined template storage account with Azure files creates File Share.
    -   An Azure storage account contains all of your Azure Storage data objects: blobs, files, queues, tables, and disks. The storage account provides a unique namespace for your Azure Storage data that is accessible from anywhere in the world over HTTP or HTTPS
    - The types of storage accounts are General-purpose V2, General-purpose V1, BlockBlobStorage, File Storage, BlobStorage accounts.
    - Types of Replication are Locally-redundant storage (LRS), Zone-redundant storage (ZRS), Geo redundant storage (GRS)
    - Types of  Performance are Standard, Premium
    - Size(sku):  A single storage account can store up to 500 TB of data and like any other Azure service
    - Below are types of storage account types ARM template support. 
        - NFS: A Network File System (NFS) allows remote hosts to mount file systems over a network and interact with those file systems as though they are mounted locally. This enables system administrators to consolidate resources onto centralized servers on the network. More information on NFS [click here](https://docs.microsoft.com/en-us/windows-server/storage/nfs/nfs-overview). 
        - GluserFS: It is an open source distributed file system that can scale out in building-block fashion to store multiple petabytes of data. More information on Gluster FS [click here](https://docs.microsoft.com/en-us/azure/virtual-machines/workloads/sap/high-availability-guide-rhel-glusterfs). 
        - Azure Files: is the only public cloud file storage that delivers secure, Server Message Block (SMB) based, fully managed cloud file shares that can also be cached on-premises for performance and compatibility. More information on Azure Files [click here](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-introduction). 
            - NFS and glusterFS:  
                - Replication is standard Locally-redundant storage (LRS)  
                - Type is Storage (general purpose v1) 
            - Azure Files:  
                - Replication is Premium Locally-redundant storage (LRS)  
                - Type is File Storage  
            - These storage mechanisms will differ according to deployment selected such as  
                - NFS and glusterFS will create a container  
                - Azure files will create a file share. 
    -  For Minimal and short2mid the template will support nfs and for Large and Maximal the template will support AzureFiles
    - To access the containers and file share etc. navigate to storage account in resource group in the portal. 
    ![storage_account](images/storage-account.png)
- **Database Template:** 
    - This Database template will creates an Azure Database for MySQL server. [Click-here](https://docs.microsoft.com/en-in/azure/mysql/) 
    - Azure Database for MySQL is easy to set up, manage and scale. It automates the management and maintenance of your infrastructure and database server,including routine updates,backups and security. Build with the latest community edition of MySQL, including versions 5.6, 5.7 and 8.0 
    - To access the database server created navigate to the resource group provided while deployment and go to Azure Database for MySQL server  
    - The database server will have a server name, server admin login name, MySQL version, and Performance Configuration 
    - Ways to connect to database server 
        - Use MySQL client or tools such as MySQL Workbench. 
        - For workbench give the connection name, hostname (server name), username (server admin login name) 
    ![mysqlworkbench](images/mysql-workbench.png)
        - After giving the details test connection. If the connection is successful it will prompt for password .Provide the password to get connected. 
        
- **Virtual Machine Template:** This template will create a  Virtual Machine
    - Controller VM: 
        - The OS used at this time is Uubntu 16.04
    - VM extension: 
        - Extension can be small applications that provide post-deployment configuration and automation tasks on Azure VMs.[Click here](https://docs.microsoft.com/en-us/azure/virtual-machines/extensions/overview) 
        - VM extension will executes a shell script file which installs Moodle on the Virtual Machine and captures the log files. 
        - Log files stderr and stdout are created at the /var/lib/waagent/custom-script/download/0/  
        - User can view the log files as a root user. 

- **Scale Set Template**: 
    -   This template will create a  [Virtual Machine Scale Set.](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/overview) 
    - A virtual machine scale set allows you to deploy and manage a set of auto-scaling virtual machines. You can scale the number of VMs in the scale set manually or define rules to auto scale based on resource usage like CPU, memory demand, or network traffic.
    - Autoscaling of VM Instances depends on [CPU utilization.](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/autoscale-overview)  
    - While scaling up an instance a VM is deployed and a shell script is executed to install the Moodle prerequisites and setting up cron jobs. 
    - VM instances have Private IP. 
    - For connecting to VMs on Scale Set with private IP, follow the steps written in the [document](https://github.com/asift91/Manual_Migration/blob/master/vpngateway.md). 
    
</details>

-   ### Manual migration of Moodle after Azure infrastructure deployment 
    -   After completion of deployment go to the resource group.  
    -   Update the Moodle folders and configuration in both the controller virtual machine and virtual machine scale set instance.
    -   **Virtual Machine**
    - Login into this controller machine using any of the free open-source terminal emulator or serial console tools 
        - Copy the public IP of controller VM to use as the hostname
        - Expand SSH in navigation panel and click on Auth and browse the same SSH key file given while deploying the Azure infrastructure using the ARM template
        - Click on Open and it will prompt to give the username. Give it as azureadmin as it is hard coded in template
        ![puttyloginpage](images/puttyloginpage.PNG)
        ![puttykey](images/puttykeybrowse.PNG)
    - After the login, run the following set of commands to migrate 
        - Download the on-premise backup from Azure Blob storage to VM such as moodle, moodledata, configuration folders with database backup file to /home/azureadmin location. 
        -   Download the compressed backup file to Controller VM at /home/azureadmin/ location.
            ```
                sudo -s
                cd /home/azuredamin/
                azcopy copy 'https://storageaccount.blob.core.windows.net/container/BlobDirectory/*' 'Path/to/folder'
            ```
    - Extract the compressed content to a folder.
        ```
            tar -zxvf moodle.tar.gz
        ```
    -   A backup folder is extracted as storage/ at /home/azureadmin/.
    -   This storage folder contains moodle, moodledata and configuration folders along with database backup file. These will be copied to desired locations.
    - Create a backup folder
        ```
            cd /home/azureadmin/
            mkdir -p backup
            mkdir -p backup/moodle
        ```
    - Replace the moodle folder  
        - Copy and replace this moodle folder with existing folder 
        - Before accessing the moodle folder switch to root user and copy the moodle folder to existing path 
            ```
                mv /moodle/html /home/azureadmin/backup/moodle/html
                cp /home/azureadmin/moodle /moodle/html
            ```
    - Replace the moodledata folder 
        - Copy and replace this moodledata (/moodle/moodledata) folder with existing folder 
        - As a root user and copy the moodledata folder existing path 
            ```
                mv /moodle/moodledata /home/azureadmin/backup/moodle/moodledata
                cp /home/azureadmin/moodledata /moodle/moodledata
            ``` 
    - Importing the Moodle database to the Azure Moodle DB    
        -   Import the on-premises database to Azure Database for MySQL.
        - Create a database to import on-premises database
            ```
                mysql -h $server_name -u $ server_admin_login_name -p$admin_password -e "CREATE DATABASE ${moodledbname} CHARACTER SET utf8;"
            ```
        - Assign right permissions to database
            ```
                mysql -h $ server_name -u $ server_admin_login_name -p${admin_password } -e "GRANT ALL ON ${moodledbname}.* TO ${moodledbuser} IDENTIFIED BY '${moodledbpass}';"
            ``` 
        - Import the database
            ```
                mysql -h db_server_name -u db_login_name -pdb_pass dbname >/path/to/.sql file
            ```
    - Configure folder permissions
        - Set 755 and www-data owner:group permissions to Moodle folder 
            ```
                sudo chmod 755 /moodle
                sudo chown -R www-data:www-data /moodle
            ```
        - Set 770 and www-data owner:group permissions to MoodleData folder 
            ```
                sudo chmod 755 /moodle/moodledata
                sudo chown -R www-data:www-data /moodle/moodledata
            ``` 
        - Change the database details in moodle configuration file (/moodle/config.php).
            - Update the following parameters in config.php
                - dbhost, dbname, dbuser, dbpass, dataroot and wwwroot
            ```
                cd /moodle/html/moodle/
                vi config.php
                # update the database details and save the file.
            ```
    - Configuring PHP and web server
        - Update the nginx conf file
            ```
                sudo mv /etc/nginx/sites-enabled/<dns>.conf  /home/azureadmin/backup/<dns>.conf 
                cd /home/azureadmin/storage/configuration/
                sudo cp <dns>.conf  /etc/nginx/sites-enabled/
            ```
        - Update the php config file
            ```
                sudo mv /etc/php/<phpVersion>/fpm/pool.d/www.conf /home/azureadmin/backup/www.conf 
                sudo  cp /home/azureadmin/storage/configuration/www.conf /etc/php/<phpVersion>/fpm/pool.d/ 
                
            ```
        -   Install Missing PHP extensions
                - ARM template install the following PHP extensions.
                    - fpm, cli, curl, zip, pear, mbstring, dev, mcrypt, soap, json, redis, bcmath, gd, mysql, xmlrpc, intl, xml and bz2
            Note: If on-premise has any additional PHP extensions those will be installed by the user.
            ```
                sudo apt-get install -y php-<extensionName>
            ```
        - Restart the web server
            ```
                sudo systemctl restart nginx 
                sudo systemctl restart php(phpVersion)-fpm  
                ex: sudo systemctl restart php7.4-fpm  
            ``` 
    -   **Virtual Machine Scaleset**
        -   Login to Scale Set VM instance and execute the following sequence of steps
        - Download the on-premise compressed data from Azure Blob storage to VM such as moodle, moodledata, configuration folders with database backup file to /home/azureadmin location. 
        -   Download the compressed backup file to Controller VM at /home/azureadmin/ location.
            ```
                sudo -s
                cd /home/azuredamin/
                azcopy copy 'https://storageaccount.blob.core.windows.net/container/BlobDirectory/*' 'Path/to/folder'
            ```
        - Extract the compressed content to a folder.
            ```
                tar -zxvf moodle.tar.gz
            ```
    -   A backup folder is extracted as storage/ at /home/azureadmin/.
        -   This Storage folder contains moodle, moodledata and configuration folders along with database backup file. These will be copied to desired locations.
        - Create a backup folder
            ```
                cd /home/azureadmin/
                mkdir -p backup
                mkdir -p backup/moodle
            ```
        
        - **Configuring PHP & web server**
            - Update the nginx conf file
                ```
                    sudo mv /etc/nginx/sites-enabled/<dns>.conf  /home/azureadmin/backup/<dns>.conf 
                    cd /home/azureadmin/storage/configuration/
                    sudo cp <dns>.conf  /etc/nginx/sites-enabled/
                ```
            - Update the php config file
                ```
                    sudo mv /etc/php/<phpVersion>/fpm/pool.d/www.conf /home/azureadmin/backup/www.conf 
                    sudo  cp /home/azureadmin/storage/configuration/www.conf /etc/php/<phpVersion>/fpm/pool.d/ 
                    
                ```
            -   Install Missing PHP extensions
                    - ARM template install the following PHP extensions.
                        - fpm, cli, curl, zip, pear, mbstring, dev, mcrypt, soap, json, redis, bcmath, gd, mysql, xmlrpc, intl, xml and bz2
                Note: If on-premises has any additional PHP extensions those will be installed by the user.
                ```
                    sudo apt-get install -y php-<extensionName>
                ```
            
        -   **Log Paths**
            -   The logging destination will need to be standardized...
            -   Log path are defaulted to /var/log/nginx.
                -   access.log and error.log are created
        -   **Restart servers**
        
            -   Update the time-stamp to update the local copy in VMSS instance.
                ```
                    /usr/local/bin/update_last_modified_time.azlamp.sh
                ```
            -   Restart nginx and php-fpm
                ```
                    sudo systemctl restart nginx
                    sudo systemctl restart php<phpVersion>-fpm
                ```
            -   If apache is installed as a webserver then restart apache
                ```
                    sudo systemctl restart apache
                ```
         

## Post Migration

-   Post migration tasks are around final application configuration that include setup of logging destinations, SSL certificates and scheduled tasks / cron jobs.

-   **Virtual Machine:**
    
    -   **Log Paths**
        
        -   on-premise might be having different log path location and those paths need to be updated with Azure log paths.
    -   **Certs:**
        -   _SSL Certs_: The certificates for your Moodle application reside in /moodle/certs/
        -   Copy over the .crt and .key files over to /moodle/certs/. The file names should be changed to nginx.crt and nginx.key in order to be recognized by the configured nginx servers. Depending on your local environment, you may choose to use the utility scp or a tool like WinSCP to copy these files over to the cluster controller virtual machine.
        -   You can also generate a self-signed certificate, useful for testing only:
            
            ```
                openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
                -keyout /moodle/certs/nginx.key \
                -out /moodle/certs/nginx.crt \
                -subj "/C=US/ST=WA/L=Redmond/O=IT/CN=mydomain.com"
            
            ```
        -   It's recommended that the certificate files be read-only to owner and that these files are owned by www-data:
            ```
                chown www-data:www-data /moodle/certs/nginx.*
                chmod 400 /moodle/certs/nginx.*
            ```
    -   **Update Time Stamp:**
        -   A cron job that runs in the VMSS instance(s) which will check the updates in timestamp for every minute. If there is an update in timestamp then local copy of VMSS is updated in web root directory.
        -   In Virtual Machine scaleset a local copy of Moodle site data (/moodle/html/moodle) is copied to its root directory (/var/www/html/).
        -   Update the time stamp to update the local copy in VMSS instance.
            ```
                /usr/local/bin/update_last_modified_time.azlamp.sh
            ```
    -   **Restart servers**
        
        -   Restart the nginx and php-fpm servers
            ```
                sudo systemctl restart nginx
                sudo systemctl restart php<phpVersion>-fpm
            ```
        -   If apache is installed as a webserver then restart apache server
            ```
                sudo systemctl restart apache
            ```
    -   **Mapping IP:**
        -   Map the load balancer IP with the DNS name.
    - Hit the load balancer DNS name to get the migrated moodle web page.   
