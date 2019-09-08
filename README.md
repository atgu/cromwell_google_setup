# CROMWELL - Google cloud setup

Notes for setting up a cromwell server for google cloud

## Installing Cromwell server to Google Cloud
 - Create a Google VM (Ubuntu) for running the cromwell server

(create_vm.png)
### Architecture requirements 
 - 1 CPU seems to work fine, but we need ~7-8G of memory. `n1-standard-2` is ok, but `n1-standard-1` is not due to too little memory (job dispatch will fail for larger submissions).
 - **REMEMBER TO GIVE SUFFICIENT API ACCESS SCOPE: "Allow full access to all Cloud APIs"**
 - Change `Boot disk` to OS of choice (tutorial using Ubuntu 18.04). Size depends on use, but 100 Gb suffices for managing many jobs. Running out of space will cause problems.
 - _Optional: Can attach separate disk to VM instance to hold the data_

(vm_cpu_type.png)

## Set up Cromwell on VM
0. Log in to VM: `gcloud compute ssh [instancename] --zone [selectedzone]`

### Install needed tools
1. Update system: 
``` sudo apt update ```
2. Install git if your OS doesn't have git by default (Ubuntu already should): 
``` sudo apt-get install git ```
3. Install docker by following instructions at [https://docs.docker.com/install/linux/docker-ce/ubuntu/](https://docs.docker.com/install/linux/docker-ce/ubuntu/). (Specifically sections "Install using the repository" and "INSTALL DOCKER ENGINE - COMMUNITY". Ubuntu = amd64)
4. Install java: 
``` sudo apt-get install default-jre ```
5. Install docker-compose: 
```
sudo curl -L https://github.com/docker/compose/releases/download/1.24.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Install cromwell server

6. Clone cromwell
```
git clone https://github.com/broadinstitute/cromwell.git
```

7. Create a directory to hold Google service account credentials which are used to run the cromwell jobs and change ownership to your account.

```
sudo mkdir [credentialdir]
sudo chown [username] [credentialdir]
```

8. Create credential json file called `default_credentials.json`

```
gcloud iam service-accounts keys create [credentialdir]/default_credentials.json --iam-account [service account name]
```

9. Change directories via: `cd cromwell/scripts/docker-compose-mysql`

10. Copy configuration template from git config/application.conf to `compose/cromwell/app-config/`

Modify following rows to appropriate values.
**Bucket should be a regional bucket in the same region so the cromwell server is a) the most efficient and b) does not incur data transfer charges!**. Zones control the default zone of the VM's cromwell is spinning up. Zones should be the same zone where the cromwell root is to avoid egress charges and for fast data transfer. These can be changed in wdl runtime attributes per task.

These will be specific to your tasks. Examples as follows:
```
application-name = "neurogap-cromwell"

call-caching {
  enabled = true
}
project = "neurogap-analysis"
root = "gs://neurogap-cromwell/"

zones: ["us-central1-b"]
```

Add yourself to docker group to be able to issue docker commands
```
sudo usermod -a -G docker $USER
```


**NOTE: Check which cromwell version you are installing!!!! Default often points to development snapshot docker!!**

Check the version in Cromwell dockerfile in compose/cromwell/Dockerfile and modify the from statement to Appropriate release version. (https://hub.docker.com/r/broadinstitute/cromwell/tags/)
E.g. using version 41: broadinstitute/cromwell:41

**ADD more memory to Cromwell process**

In some large workflows Cromwell can run out of memory and slowdown to a crawl.
Modify command part in docker-compose-mysql file after java add e.g. ```-Xmx12g``` to **allocate 12 gigs of memory** and **modify configuration to point to the copied configuration file**.

Full line would be e.g. ``` command: ["/wait-for-it/wait-for-it.sh mysql-db:3306 -t 120 -- java ```**-Xmx12g**``` -Dconfig.file=```**/app-config/application.conf**``` -jar /app/cromwell.jar server"] ```

**Choose how call caching data localization happens in Google Cloud**

By default all old result files are copied to new workflows input bucket. This is often not desired if generating lots of data.
We have set FINNGEN cromwell to only reference the data. See ``` duplication-strategy = "reference" ``` part in config/application.conf


## Start up Cromwell

While logged into the Cromwell VM, Build the docker compose image:

``` docker-compose build ```

(If docker isn't running, start it first:)
``` service docker start ```

Start the Cromwell server docker services

``` docker-compose up -d ```

_Optional_: automatically start cromwell server on server startup. Add following line to /etc/rc.local

```
docker-compose -f [path_to_docker_compose_file]/docker-compose.yml up -d
```

Get Cromwell logs with

``` docker-compose logs ```

## Access CROMWELL server and submit workflows

Create a socks proxy to the created vm

``` gcloud compute ssh  [vmname] --zone [zone] -- -N -p 22 -D localhost:5000 ```

Set your browsers proxy settings to use socks proxy on localhost:5000 and then browse to `0.0.0.0` (e.g. in Chrome)

Open 'Submit a workflow for execution', click 'Try it out', input your wdl and json configuration files, and click submit.
Record the workflow ID returned in response. You can query the status of workflow using the ID in 'Get workflow and call-level metadata for a specified workflow'


## Using VMs with private IP addresses

The number of external IP addresses is our quota bottleneck if running low-cpu instances with Cromwell with public IPs. However, in the runtime block of a WDL task, it's possible to give a ```noAddress: true``` option, which will result in a private IP for that task's VM so a public IP will not be used.

Example runtime:
```
    runtime {
        docker: "gcr.io/finngen-refinery-dev/saige:finngen-15"
        cpu: "1"
        memory: "3 GB"
        disks: "local-disk 10 HDD"
        zones: "europe-west1-b"
        preemptible: 0
        noAddress: true
    }
```

To connect to a VM with a private IP, first SSH to a VM in the same network to use as a bastian host (e.g. the Cromwell server). Then create an SSH key pair by e.g. ```ssh-keygen```. In the Cloud Console page of the VM you want to connect to, copy-paste the public key (~/.ssh/id_rsa.pub) to the SSH keys. Then connect to the machine from the bastian host without the gcloud command: ```ssh ggp-xxx -i ~/.ssh/id_rsa```, where ggp-xxx is the name of the machine you want to connect to.

## Troubleshooting

In case of errors / slowdowns etc. when running workflows pull workflow metadata from Cromwell server.

```
curl -X GET "http://0.0.0.0/api/workflows/v1/{WORKFLOW_ID}/metadata?expandSubWorkflows=false" -H  "accept: application/json" --socks5 localhost:5000 > meta.json
```

On some large workflows Cromwell runs out of memory when requesting the meta-data. In that case you can subset the data returned to only to bits you are interested in. E.g. when debugging issue with callcaching you can use:

```
curl -X GET "http://0.0.0.0/api/workflows/v1/{WORKFLOW_ID}/metadata?expandSubWorkflows=false&includeKey=callCaching" -H  "accept: application/json" --socks5 localhost:5000 > meta.json
```

## Mounting with external disk _(Optional)_

When on `Install cromwell server` step, do the following first:

Switch to mounted disks: ` cd [mountpoint] `

### Format the external disk (WARNING ONLY IF NEW DISK!!!!!! ALL DATA WILL BE DELETED)
Check the name of available disks (block devices)
```sudo lsblk```

Format the disk:
```
sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard [DEVNAME]
```

### Mount the external disks
Create mountpoint
``` sudo mkdir -p /mnt/disks/[MNT_DIR]```
Mount the drive:
``` sudo mount -o discard,defaults,noatime,nodiratime [devname] [mountpoint] ```

Automatically mount on startup
Edit /etc/fstab and add a line

[devicename]        [mountpoint]        ext4    defaults,noatime,nodiratime 0      0

## If using CloudSQL instead of local MySQL _(Optional)_

The docker-compose setup described in this README sets up MySQL locally for Cromwell to use. Alternatively CloudSQL can be used - the advantages are that then the DB is completely separate from Cromwell installation and that MySQL won't take up any resources on the Cromwell VM.

#### Set up CloudSQL

First, make the IP address of your Cromwell VM static so we can restrict MySQL connections to coming from that address. (In the Cloud console, you can make the address static on VPC Network -> External IP addresses.) Then go to [https://console.cloud.google.com/sql](https://console.cloud.google.com/sql) and create a MySQL instance, let's call it "cromwell". Create a user e.g. "cromwell" for that instance, allowing connections from the IP address you just made static. Create a database called "cromwell". In the Connections tab, use "Add network" and insert that same IP address to authorize connections from it.

#### Set up installation using CloudSQL and no local MySQL

In application.conf, replace the database config in the end with:

```
database {
  db.url = "jdbc:mysql://[IP-ADDRESS-OF-CLOUDSQL]/cromwell?useSSL=false&rewriteBatchedStatements=true&cloudSqlInstance=finngen-refinery-dev:europe-west1:cromwell"
  db.user = "cromwell"
  db.password = "[PW]"
  db.driver = "com.mysql.cj.jdbc.Driver"
  profile = "slick.jdbc.MySQLProfile$"
}
```

replacing `[IP-ADDRESS-OF-CLOUDSQL]` with the IP address of your CloudSQL instance, `finngen-refinery-dev:europe-west1:cromwell` with the instance connection name of your CloudSQL instance (visible on the Overview tab in the cloud console page of the instance), and `[PW]` with the password of the DB user.

Replace `scripts/docker-compose-mysql/compose/cromwell/Dockerfile` with `cloudsql/Dockerfile` from this repo and replace `scripts/docker-compose-mysql/docker-compose.yml` with `cloudsql/docker-compose.yml` from this repo. This takes out local MySQL.
