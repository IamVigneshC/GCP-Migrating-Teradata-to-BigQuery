# Migrating-Teradata-to-BigQuery
Create a Teradata source system running on a Compute Engine instance  Prepare the Teradata source system for the schema and data transfer  Configure the schema and data transfer service  Migrate the schema and data from Teradata to BigQuery  Translate Teradata SQL queries into compliant BigQuery Standard SQL

## Creating the Teradata VM and downloading Teradata Express

Enable the required Cloud APIs:

The following APIs are required for the demo: Compute Engine, Cloud Storage, Cloud Storage JSON, Pub/Sub, BigQuery, BigQuery Data Transfer Service, and Cloud Build. 

### Check which APIs are enabled for the project
`gcloud services list --enabled`


### Enable the following APIs:


### Required for the data transfer service

gcloud services enable bigquery-json.googleapis.com

gcloud services enable storage-api.googleapis.com

gcloud services enable storage-component.googleapis.com

gcloud services enable pubsub.googleapis.com

gcloud services enable bigquerydatatransfer.googleapis.com



### Required to run Teradata on a GCE instance
`gcloud services enable compute.googleapis.com`

Create a service account for the BigQuery Data Transfer Service
This service account will be used to configure the transfer service and also to run the Teradata Compute Engine instance.

Using Cloud Shell:

### Set the PROJECT variable
`export PROJECT=$(gcloud config get-value project)`

### Create a service account
`gcloud iam service-accounts create td2bq-transfer`

You now grant BigQuery Google Cloud Storage administration permissions to the Teradata to BigQuery service account so that it can access the services that are required for the migration tasks.

To create an environment variable to hold the Teradata to BigQuery migration Service Account enter the following in Cloud Shell:

### Set the TD2BQ_SVC_ACCOUNT = service account email

export TD2BQ_SVC_ACCOUNT=`gcloud iam service-accounts list \
  --filter td2bq-transfer --format json | jq -r '.[].email'`
  
To bind the migration service account to the BigQuery admin role enter the following in Cloud Shell:

### Bind the service account to the BigQuery Admin role
gcloud projects add-iam-policy-binding ${PROJECT} \
  --member serviceAccount:${TD2BQ_SVC_ACCOUNT} \
  --role roles/bigquery.admin
  
  
To bind the migration service account to the Cloud Storage admin role enter the following in Cloud Shell:

### Bind the service account to the Storage Admin role
gcloud projects add-iam-policy-binding ${PROJECT} \
  --member serviceAccount:${TD2BQ_SVC_ACCOUNT} \
  --role roles/storage.admin
  
  
To bind the migration service account to the PubSub admin role enter the following in Cloud Shell:

### Bind the service account to the Pub/Sub Admin role
gcloud projects add-iam-policy-binding ${PROJECT} \
  --member serviceAccount:${TD2BQ_SVC_ACCOUNT} \
  --role roles/pubsub.admin

Create a Compute Instance virtual machine (VM) to run Teradata
You will be now need to create your Teradata Compute Engine VM instance. The Compute Engine VM image will be populated with:

Teradata Express running as a nested QEMU VVM

The Teradata BTEQ query utility

TPC-H data - This is a publicly available OLAP schema primarily used to benchmark data warehousing queries

The BigQuery Data Transfer Service agent

Creating the Teradata Compute Engine VM instance
The Compute Engine VM instance is the host for the Teradata virtual machine. Because there will be a nested VM running inside a Compute VM, the Compute Engine VM instance must have virtualization support enabled.

Nested virtualization can only be enabled for Compute Engine VMs running on Haswell processors or later. Review the Regions and Zones page to determine which zones support Haswell or later processors. In this guide we will always use the us-central1 region.

To create an Ubuntu boot disk enter the following in Cloud Shell:

### Create UBUNTU boot disk
gcloud compute disks create ubuntu-disk \
   --type=pd-ssd \
   --size=300GB \
   --zone=us-central1-c \
   --image=ubuntu-1604-xenial-v20200521 \
   --image-project=ubuntu-os-cloud

When the command completes it will tell you that the new disk is ready.

In order to run a nested hypervisor you must create an image from the boot disk and enable support for Intel VT-x processor virtualization instructions. The CPU flag for VT-x capability is VMX, which stands for Virtual Machine eXtensions.

To create an image with nested hypervisor support enter the following in Cloud Shell:

### Create image with VMX enabled
gcloud compute images create vmx-enabled-image \
  --source-disk ubuntu-disk --source-disk-zone us-central1-c \
  --licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx"


To create the Compute Engine VM instance from the image enter the following in Cloud Shell:

### Create Compute Engine virtual machine from image
gcloud compute instances create teradata \
  --zone=us-central1-c \
  --machine-type=n1-standard-4 \
  --image=vmx-enabled-image \
  --min-cpu-platform "Intel Haswell" \
  --service-account=td2bq-transfer@${PROJECT}.iam.gserviceaccount.com \
  --scopes=https://www.googleapis.com/auth/cloud-platform \
  --metadata startup-script-url=gs://cloud-training/CBL046/load-td-installer.sh
  
  
  ## Configure the Teradata Compute Engine VM
  
Accept the licensing to allow you to download the Teradata Express VM image, Teradata tools, and JDBC drivers from Teradata's support site. You download these directly to the Teradata VM once you have accepted the licenses as this allows the downloads to be substantially faster.

Navigate to Compute Engine > VM Instances using the Navigation menu in the top left of the console page. In the list of VM instances you will see the teradata instance.
Click the SSH link in the Connect column for the teradata instance.
A new window will open with an SSH terminal connected to the Compute Engine VM.

Note:
The following sequence of instructions require you to log in to the Teradata downloads site and accept the licensing for the three component downloads that are required to build the Teradata Express image that you will use during the lab.

When you accept the license a download will be triggered - you then copy the URL for that specific download and use that with the curl command inside your Teradata VM to download the files directly to that VM.

When all the files have been downloaded you will launch an automated script that will complete the rest of the Teradata Data Warehouse deployment and configuration for you.

Download the Teradata client
The teradata client is called Basic Teradata Query (BTEQ). It is used to communicate with one or more Teradata Database servers and to run SQL queries on those systems.

Download the Teradata Tools and Utilities package to your local machine.

Open the Teradata Tools and Utilities page.
Login or create an account. Make sure you read and accept the terms of service before proceeding.
Download the tar.gz file corresponding to the desired version of Teradata to a directory in your local machine. In this guide you will use version 16.10, therefore the tools file is linked under TTU 16.10.26.00 Ubuntu - Base.
In the License Agreement pop-up window page down and click I Agree.
When the download begins open your Downloads page. In Google Chrome you can open this directly by pressing CTRL+J.
Click Pause on your download. You don't want to download it to your machine, you want to copy the URL as you will use that inside your teradata VM.
Copy the download URL.

Switch to the SSH window for the Teradata compute instance.

Enter the following command, replacing DOWNLOAD-URL with the full URL you copied in step 7, in the SSH window:

` curl -o ~/TeradataToolsAndUtilitiesBase__ubuntu_indep.tar.gz "DOWNLOAD-URL" `

repeat the exercise for the VM image and the Java drivers for the migration agent. Those files are somewhat bigger but all of the downloads should complete in a minute or two at most.

Return to your Teradata download tab in your browser and cancel the download.

Download the Teradata Express VM image
Teradata Express is distributed as a VMware Player VM image. You now download this image and in a later step you convert it so that it can be used by the QEMU hypervisor that can run inside a Compuite Engine Ubuntu instance.

Download Teradata Express to your local machine.

Open this Teradata download page.

Log in or create an account, read and accept the terms of service before proceeding.

Note: You must be logged in to the Teradata support and downloads site in order to see the file you require.
Click TDExpress16.10.00.03_Sles11_40GB.7z to start the download the 4 GB file compressed with 7z, to a directory in your local machine.

In the License Agreement pop-up window page down and click I Agree.

When the download begins open your Downloads page. In Google Chrome you can open this directly by pressing CTRL+J.

Click Pause on your download. You don't want to download it to your machine, you want to copy the URL as you will use that inside your Teradata VM.

Copy the download URL.

Switch to the SSH window for the Teradata compute instance.

Enter the following command, replacing DOWNLOAD-URL with the full URL you copied in step 7, in the SSH window:

` curl -o ~/TDExpress16.10_Sles11_40GB.7z "DOWNLOAD-URL" `

Wait for the download to complete. As this is a 40GB file it will take longer than the first file but it should still complete in under two minutes.

Return to your Teradata download tab in your browser and cancel the download.

Download the Teradata JDBC drivers
Navigate to this Teradata Downloads page.

Log in or create an account, read and accept the terms of service before proceeding.

Download the zip file corresponding to the version of Teradata that you have configured to a directory in your local machine. If you have followed this guide it is version 16.10, so the file is TeraJDBC__indep_indep.16.10.00.07.zip.

In the License Agreement pop-up window page down and click I Agree.

When the download begins open your Downloads page. In Google Chrome you can open this directly by pressing CTRL+J.

Click Pause on your download. You don't want to download it to your machine, you want to copy the URL as you will use that inside your Teradata VM.

Copy the download URL.

Switch to the SSH window for the Teradata compute instance.

Enter the following command, replacing DOWNLOAD-URL with the full URL you copied in step 7, in the SSH window:

` curl -o ~/TeraJDBC__indep_indep.16.10.00.07.zip "DOWNLOAD-URL" `

Wait for the download to complete. As this is a 4 GB file it will take longer than the first file but it should still complete in under two minutes.

Return to your Teradata download tab in your browser and cancel the download.

Start the Automated Teradata Installation script
Now that you have downloaded the three files directly to your Teradata VM you can start the automated installation script that will unpack the VM and convert it from VMware Player format to libvirt/QEMU format. Start the VM and then import the tpch Data Warehouse benchmark data so that you have a sample Data Warehouse to migrate.

Open the SSH window for your Teradata VM instance.

To launch the automated Teradata nested VM installation script enter the following command in the SSH window for the Teradata compute instance:

cd ~
sudo cp /root/configure-teradata-vm.sh .
./configure-teradata-vm.sh

Note:
This process will take at least 30 minutes to complete. You must leave this SSH session open with the script running until it completes.


When the script has completed you will see the final logoff notice and the message Fastload Terminated in the SSH window.


## Configure the Google Cloud environment and start the data Migration from Teradata

You must prepare the Google Cloud environment by creating a storage bucket for staging, a target BigQuery dataset, and the BigQuery Data Transfer Service Teradata migration resource.

Once the Google Cloud components have been configured you can initialize the BigQuery Teradata Transfer Agent running in the Teradata environment and then start a data transfer run.

Create the Cloud Storage bucket for staging
This bucket will be used by the BigQuery Data Transfer Service as a staging area for data files to be ingested into BigQuery.

To make sure the environment variables you need are still defined enter the following commands in Cloud Shell.

### Set the PROJECT variable to the Project ID
 ` export PROJECT=$(gcloud config get-value project) `

### Set TD2BQ_SVC_ACCOUNT to the migration service account
export TD2BQ_SVC_ACCOUNT=`gcloud iam service-accounts list \
  --filter td2bq-transfer --format json | jq -r '.[].email'`

To create a cloud storage bucket enter the following in Cloud Shell:

### Use gsutil to create the bucket
` gsutil mb -c regional -l us-central1 gs://${PROJECT}-td2bq `

Create the BigQuery dataset
This is the dataset that will be used for the migrated data from Teradata.

Note: The dataset's multi-region location should contain the Cloud Storage bucket's region. In this case it is: US.
To create the BigQuery dataset enter the following in the Cloud Shell:

### Use the bq utility to create the dataset
` bq mk --location=US td2bq `

You can also check the schema on your Teradata instance.

Switch back to the the Teradata compute instance SSH Window.

To save the nested Teradata QEMU VM IP address in an environment variable enter the following in the SSH Window:

### Get the Teradata VM IP
` TERA_IP=$(sudo virsh net-dhcp-leases default | sed -n 3p | awk '{print $5}' | cut -f1 -d/) `

This puts the address of the nested QEMU virtual machine where the Teradata data warehouse is actually running into an environment variable called TERA_IP.

To launch bteq and log on to Teradata, enter the following in the SSH Window:

### Use the BTEQ query utility
` bteq .LOGON $TERA_IP/dbc,dbc `

Warning: You may see the following error depending on how quickly you have run through the lab up to this point. You can proceed with the subsequent query even if you see this:

*** Warning: RDBMS CRASHED OR SESSIONS RESET. RECOVERY IN PROGRESS.

However, if this error appears to persist, and the query does not work, you are probably not logging in using correct IP address for the QEMU VM. Recheck the last few steps to retry.

Enter the following SQL:

` SELECT * FROM DBC.Tables WHERE DatabaseName ='tpch'; `

Note: All Teradata SQL commands must end with a semicolon.


Enter the following SQL:

` SELECT TOP 100 * FROM tpch.supplier ORDER BY s_suppkey; `


To exit the BTEQ utility:

.QUIT


Leave the SSH Window open.


## Configure the BigQuery Data Transfer Service
You configure a BigQuery Data Transfer Service resource that the transfer agent will connect to. The BigQuery Data Transfer Service resource will then import the database schema and data into the BigQuery dataset that you select for that transfer.

Enter the following commands in the Cloud Shell to display the configuration values you will use when configuring the BigQuery Data Transfer Service:

` export PROJECT=$(gcloud config get-value project) `

### Print the project, bucket, dataset and service account names
` echo Project ID: $PROJECT `

### Storage Bucket
` gsutil ls | sed -nr 's/^gs:\/\/(.*-td2bq)\/$/\1/p' `

### BigQuery dataset
` bq ls | grep "td2bq" `

### Teradata Migration Service Account
` gcloud iam service-accounts list --format="value(email)" | grep "td2bq" `

To create the BigQuery Data Transfer Service config, navigate to BigQuery in the Google Cloud Console and click Transfers:

This will take you to the BigQuery Data Transfer Service page.

Choose Create a Transfer and then "Migration: Teradata" as the source.
You should see the Transfer service configuration dialog.

Complete the transfer configuration form using the following values:

Enter any Display name, for instance: td2bq-transfer
Schedule options: Leave as is.
Destination dataset: Choose td2bq
Database type: Should be Teradata
Cloud Storage bucket: PROJECT_ID-td2bq . Replace PROJECT_ID with your project ID. You can also click Browse to select the bucket.
Database name Change to tpch
Table name patterns: Change to supplier;orders;part
See note below explaining these patterns.

Service account email: Set to td2bq-transfer@PROJECT_ID.iam.gserviceaccount.com . Replace PROJECT_ID with the output of the command above.
Schema file path: Leave empty.
Select a Pub/Sub topic: Select Create a topic, name it td2bq-topic and then click the Create topic button to go back to the form.

Finally, click on Save in the main form.
You will be prompted to Choose an account to continue to the BigQuery Data Transfer Service. You must accept this as the lab student to allow the transfer to proceed.
Note: If you do not see the popup the first time you create a transfer double check that the authentication pop-up is not hidden behind another window.
You will then see the transfer run in progress.

Click the Configuration tab.

Make a copy of the Resource Name as you will need it in a later step.

Table name patterns
The Table name patterns field expects an expression that indicates which tables to include in the transfer. In this demo you include only three of the tables from the tpch database. To see a list of tables, refer to the Schema and Data Overview section in this document.

Pipe delimited list of tables: lineitem|part|orders

Semicolon delimited list of qualified table names: lineitem;orders

Use of a wildcard: *

## Initialize the BigQuery Data Transfer Service agent

In this section you will download and then initialize the transfer agent that will be used to automate the movement of data between Teradata and BigQuery. The agent is part of the BigQuery Data Transfer Service.

In order to migrate the schema and data to BigQuery, you must create a credentials file and then initialize the migration agent installed on the Teradata machine.

Open the Teradata SSH window.

To install the Java Runtime Environment (JRE) enter the following in the SSH window:

### Install Java
` sudo apt-get install -y openjdk-8-jre-headless `

The automated setup script created a directory, called /home/agentbase and saved the Teradata JDBC drivers in there as part of the preparation process. In your own environment you would download and unpack these files on the machine where you plan to install the transfer agent yourself.

To download the BigQuery Data Transfer Service agent enter the following in the SSH window:

### Download transfer agent
sudo wget -P /home/agentbase/ \
https://storage.googleapis.com/data_transfer_agent/latest/mirroring-agent.jar


Copy the transfer agent *.jar and JDBC driver files from the staging directory to your home directory. This copies the transfer agent you just downloaded and the extracted files from the Teradata JDBC driver files that you saved at the start of the lab.

### Copy the *.jar files to your home directory
` cp /home/agentbase/*.jar ~ `

Make a note of the Teradata VM IP address. You will need this to identify the Teradata instance during the Agent configuration.

### Take note of the Teradata VM IP
` sudo virsh net-dhcp-leases default | sed -n 3p | awk '{print $5}' | cut -f1 -d/ `


Create a new credentials file for the transfer agent using the nano editor.

nano credentials

Copy and paste the credentials details into the credentials file.
username=dbc
password=dbc

Press CTRL+X then Y to save the credentials file.

Initialize the transfer agent.

### Initialise the agent
` java -cp ~/mirroring-agent.jar com.google.cloud.bigquery.dms.Agent --initialize `


For the subsequent prompts enter the following values:
Prompt	Value to enter
Would you like to write the default TPT template to disk? (yes/no):	no
Enter directory path for locally extracted files:	/tmp
Enter Database hostname:	Use the Teradata VM IP address obtained in a previous step in this section
Enter Database connection port: (press enter to use default port):	Press enter
Would you like to use Teradata Parallel Transporter (TPT) to unload data? (yes/no):	no
Enter database credentials file path:	credentials
Enter BigQuery Data Transfer Service config name:	Use the Resource Name saved in the previous section.
Enter configuration file path:	config

Check the configuration file details:

### cat the config file
` cat config `


Migrate the schema and data
Now that the transfer agent has been installed and initialized you must run it to start transferring the schema and data.

To start the data transfer agent enter the following the the SSH window.

### Run the agent in the background
` java -cp mirroring-agent.jar:tdgssconfig.jar:terajdbc4.jar com.google.cloud.bigquery.dms.Agent --configuration-file=config `

Note: If you get config file errors here then manually edit the config file and check that the transfer config ID is correct.
Note: Within a minute or two you should see a message from the Agent stating that databases have been listed in the database and you should see corresponding event log data appear in the BigQuery UI in the Run History tab for the transfer indicating that tables have been matched.

The schema and data transfer can take up to 30 minutes to complete.

## Teradata to BigQuery SQL query translation

you will compare Teradata SQL query syntax to equivalant BigQuery queries running against the data that has been migrated from the Teradata source data warehouse.

Confirm that data has been migrated successfully
You start by checking that the transfer has completed successfully.

Check the status in the BigQuery Transfers UI by selecting td2bq-transfer:

Wait for the transfer to complete.
You will know that the transfer is done:

When the td2bq-transfer in the BigQuery Transfers UI shows a green checkmark, and
The SSH window where you run the agent shows all three TPC-H tables, tpch.supplier,tpch.orders, and tpch.part are marked as DONE.

In the SSH window press CTRL+C to exit the transfer run. The transfer agent will not exit automatically.
You can now verify that the schema has been migrated to BigQuery:

Open the BigQuery UI in the Cloud Console.
On the left hand side, click on the lab project ID.
Click on the down arrow next to the td2bq dataset.
You should see the Teradata tables that were migrated to BigQuery.

Check that the data that was migrated to BigQuery:
In the BigQuery query editor enter the following query:

SELECT *
FROM `td2bq.supplier`
ORDER BY s_suppkey
LIMIT 100

Click Run.

Compare with the previous results from the same query in Teradata:

## Teradata to BigQuery SQL query translation
After data has been migrated, customers will begin to migrate existing queries and scripts to BigQuery. This stage typically takes up the bulk of the migration because of the effort needed to convert scripts between SQL dialects and the customer's usage of Teradata proprietary SQL functions.

Value: BigQuery is fully compliant with the SQL 2011 standard, which makes it easy for SQL practitioners to apply their skills when migrating legacy queries.

In this section, you will migrate Teradata queries to standard SQL, making some common adjustments along the way. These are examples meant to show the mechanics of query translation. They are not an exhaustive set of differences between Teradata and Standard SQL.

For a more detailed set of differences, please refer to the Teradata to BigQuery SQL Query Translation guide.

You may also use this guide in your future customer engagements.

In order to compare the results between Teradata and BigQuery, you will utilise Teradata's BTEQ command-line tool to execute queries against the source environment for result verification.

### The QUALIFY clause
Teradata's QUALIFY is a conditional clause in the SELECT statement that filters results of a previously computed ordered analytical function according to user‑specified search conditions. Teradata customers very commonly use this function as a shorthand way to RANK and return results without the need for an additional subquery.

## Run the query in Teradata
Follow these steps to use the query shown below to return information about the last purchase made by a customer.

Switch back to the Teradata compute instance SSH window.

Store the IP address of the nested Teradata QEMU VM in an environment variable. This is the QEMU virtual machine where the Teradata data warehouse is actually running.

### Get the Teradata VM IP
` TERA_IP=$(sudo virsh net-dhcp-leases default | sed -n 3p | awk '{print $5}' | cut -f1 -d/) `


In the ssh session on Teradata machine launch BTEQ and logon to Teradata:

### Use the BTEQ query utility
` bteq .LOGON $TERA_IP/dbc,dbc `


Then enter the following SQL:

SELECT
  orders.O_CUSTKEY,
  orders.O_TOTALPRICE,
  orders.O_ORDERDATE
FROM
  (
    SELECT
      O_CUSTKEY,
      O_TOTALPRICE,
      O_ORDERDATE,
      ROW_NUMBER() OVER (
        PARTITION BY O_CUSTKEY
        ORDER BY
          O_ORDERDATE DESC,
          O_ORDERKEY DESC
      ) AS row_id
    FROM
      tpch.orders
    QUALIFY row_id = 1
  ) AS orders
WHERE
  orders.O_ORDERDATE = '1998-07-06'
ORDER BY
  orders.O_CUSTKEY;
  
 ### Translate the query to BigQuery
There are two changes applied to this query to translate it:

The QUALIFY clause is translated by adding a WHERE condition containing the analytics value to the enclosing query.
Optionally, the query is simplified by removing the alias for the nested select. BigQuery does not require aliases for nested select statements.
This is the translated SQL which works in BigQuery:

SELECT
  O_CUSTKEY,
  O_TOTALPRICE,
  O_ORDERDATE
FROM (
  SELECT
    O_CUSTKEY,
    O_TOTALPRICE,
    O_ORDERDATE,
    ROW_NUMBER() OVER (
      PARTITION BY O_CUSTKEY
      ORDER BY
        O_ORDERDATE DESC,
        O_ORDERKEY DESC
    ) AS row_id
  FROM `td2bq.orders`
)
WHERE row_id = 1
  AND O_ORDERDATE = '1998-07-06'

Switch back to the BigQuery console, enter the above query in the Query editor, and then click Run.
The results are the same as the results obtained from Teradata.

Value: The translated query uses only Standard SQL, and therefore is easier to understand and maintain by SQL practitioners.

Filtering and Concatenation
Teradata SQL syntax for filtering and concatenation is slightly different than that of ANSI Standard SQL.

The query below filters on P_NAMEs starting with a|h|w, concatenates the P_PARTKEY and P_NAME together, and finally returns the top 100 records ordered by the concatenated part key and name.

Run the query in Teradata
Switch back to the BTEQ session in the Teradata SSH Window.
If you aren't already connected to BTEQ, follow the instructions in the previous section, The QUALIFY clause, to reconnect BTEQ to the Teradata data warehouse.

### Enter the following SQL in the BTEQ session:

SELECT TOP 100 TRIM(LEADING FROM P_PARTKEY||' - '||P_NAME) AS part
FROM tpch.part
WHERE P_NAME LIKE ANY ('a%','w%','h%')
ORDER BY part;

## Translate the query in BigQuery
There are three changes applied to this query to translate it:

The string concatenation operator in Teradata is ||. In standard SQL, this is expressed with the CONCAT() function.

TRIM / LEADING is replaced by CAST. CAST is necessary because the P_PARTKEY is integer.

LIKE ANY is translated to separate OR conditions in the WHERE clause.

SELECT
 CONCAT(CAST(P_PARTKEY AS STRING), ' - ', P_NAME) AS part
FROM `td2bq.part`
WHERE
 P_NAME LIKE 'a%'
 OR P_NAME LIKE 'w%'
 OR P_NAME LIKE 'h%'
ORDER BY part
LIMIT 100

Switch back to the BigQuery console, enter the above query in the Query editor, and then click Run.
The results are the same as those obtained from Teradata.

