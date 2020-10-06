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


