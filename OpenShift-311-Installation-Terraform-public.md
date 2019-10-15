# Installation of OpenShift



In this tutorial, you provision IBM Cloud classic infrastructure for the Red Hat OpenShift Container Platform by using Terraform. Before you can start the classic infrastructure provisioning process, you must ensure that you set up Terraform, the IBM Cloud Provider plug-in, and the Terraform OpenShift project.

**All passwords, userIDs, keys, certificates are fake in this document.**

Documentation:

<https://docs.openshift.com/container-platform/3.11/welcome/index.html>

<https://cloud.ibm.com/docs/terraform/tutorials?topic=terraform-redhat>

# 1. Configure your environment

On you local machine, Create a Docker container that installs Terraform and the IBM Cloud Provider plug-in. To execute Terraform commands, you must be logged in to the container. You can also [install Terraform and the IBM Cloud Provider plug-in](https://cloud.ibm.com/docs/terraform?topic=terraform-setup_cli#setup_cli) on your local machine to run Terraform commands without using a Docker container.

Command:

```
docker pull ibmterraform/terraform-provider-ibm-docker
```

Results:

```shell
Using default tag: latest
latest: Pulling from ibmterraform/terraform-provider-ibm-docker
911c6d0c7995: Pull complete 
fed331e93a76: Pull complete 
82a1ea1a0cd7: Pull complete 
a4b4f00ab356: Pull complete 
78858415d97d: Pull complete 
515c9be5f236: Pull complete 
94021f117e26: Pull complete 
a50b454f6bba: Pull complete 
dd63d43987e3: Pull complete 
a098dba94337: Pull complete  
Digest: sha256:df316f5ed26cbec1bc1ad7a6f6d2c978f767408080a4a4db954c94c91e8271e5
Status: Downloaded newer image for ibmterraform/terraform-provider-ibm-docker:latest
```

Create a container from your image and log in to your container. When the container is created, Terraform and the IBM Cloud Provider plug-in are automatically installed and you are automatically logged in to the container. The working directory is set to `/go/bin`.

Command:

```shell
docker run -it ibmterraform/terraform-provider-ibm-docker:latest
```

Results:

```shell
 docker run -it ibmterraform/terraform-provider-ibm-docker:latest
/go/bin # 
```

From **within** your container, set up the IBM Terraform OpenShift Project.

Command:

```shell
apk add --no-cache openssh
```

Results:

```shell
apk add --no-cache openssh
fetch http://dl-cdn.alpinelinux.org/alpine/v3.7/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.7/community/x86_64/APKINDEX.tar.gz
(1/6) Installing openssh-keygen (7.5_p1-r10)
(2/6) Installing openssh-client (7.5_p1-r10)
(3/6) Installing openssh-sftp-server (7.5_p1-r10)
(4/6) Installing openssh-server-common (7.5_p1-r10)
(5/6) Installing openssh-server (7.5_p1-r10)
(6/6) Installing openssh (7.5_p1-r10)
Executing busybox-1.27.2-r11.trigger
OK: 326 MiB in 50 packages
```

Download the Terraform configuration files to deploy the Red Hat OpenShift Container Platform.

```
git clone https://github.com/IBM-Cloud/terraform-ibm-openshift.git
```

Results:

```shell
 git clone https://github.com/IBM-Cloud/terraform-ibm-openshift.git
Cloning into 'terraform-ibm-openshift'...
remote: Enumerating objects: 35, done.
remote: Counting objects: 100% (35/35), done.
remote: Compressing objects: 100% (28/28), done.
remote: Total 601 (delta 12), reused 15 (delta 7), pack-reused 566
Receiving objects: 100% (601/601), 878.94 KiB | 583.00 KiB/s, done.
Resolving deltas: 100% (311/311), done.
```

Navigate into the installation directory.

```shell
cd terraform-ibm-openshift
```

Results:

```shell
/go/bin/terraform-ibm-openshift # ls -l
total 56
-rw-r--r--    1 root     root          6550 Sep 28 09:54 README.md
drwxr-xr-x    5 root     root          4096 Sep 28 09:54 docs
drwxr-xr-x    2 root     root          4096 Sep 28 09:54 inventory_repo
-rw-r--r--    1 root     root          9076 Sep 28 09:54 main.tf
-rw-r--r--    1 root     root          2958 Sep 28 09:54 makefile
drwxr-xr-x    7 root     root          4096 Sep 28 09:54 modules
-rw-r--r--    1 root     root          1609 Sep 28 09:54 output.tf
-rw-r--r--    1 root     root           143 Sep 28 09:54 provider.tf
drwxr-xr-x    2 root     root          4096 Sep 28 09:54 scripts
drwxr-xr-x    2 root     root          4096 Sep 28 09:54 templates
-rw-r--r--    1 root     root          1592 Sep 28 09:54 variables.tf
```

Generate an SSH key. The SSH key is used to access IBM Cloud classic infrastructure resources during provisioning.

Create an SSH key inside the container that you created earlier. Enter the email address that you want to associate with your SSH key. Make sure to accept the default file name, file location, and missing passphrase by pressing **Enter**.

Command:

```
ssh-keygen -t rsa -b 4096 -C "<email_address>"
```

Results:

```shell
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:67LDt8zjbPoX+uKFGrVs2CrsyNk1izzBOkDf8tBb3Xc myemail@example.com
The key's randomart image is:
+---[RSA 4096]----+
|                 |
|                 |
|                 |
| .               |
|. ..o   S .      |
|.  +oo * =.. . E |
| . o+oB B.... .  |
| .o=+=+%*..      |
|  +o=o*@X*.      |
+----[SHA256]-----+
```



Verify that the SSH key is created successfully. The creation is successful if you can see one **id_rsa** and one **id_rsa.pub** file.

Command:

``` shell
cd /root/.ssh && ls
```

Results:

```
id_rsa      id_rsa.pub
```

Navigate back to your OpenShift directory:

```shell
cd /go/bin/terraform-ibm-openshift
```

Open the Terraform `variables.tf` file and review the default values that are set in the file. The `variables.tf` file specifies all information that you want to pass on to Terraform during the provisioning of your infrastructure resources. You can change the default values, but do not add sensitive data, such as your infrastructure user name and API key, to this file. The `variables.tf` file is usually stored under version control and shared across users.

Command:

```shell
vi variables.tf
```

List:

| Variable name     | Description                                                  | Default value       |
| ----------------- | ------------------------------------------------------------ | ------------------- |
| `app_count`       | Enter the number of app nodes that you want to create. App nodes are used to run your app pods. | 1                   |
| `bastion_flavor`  | Enter the flavor that you want to use for your Bastion virtual machine. The Bastion host is the only ingress point for SSH in the OpenShift cluster from external entities. When you connect to the OpenShift Container Platform infrastructure, the Bastion host forwards the request to the infrastructure or app server. For more information, see [Bastion instance ![External link icon](https://cloud.ibm.com/docs-content/v1/content/6d69c89aece95412281779ad9e9aee96fcb73b84/icons/launch-glyph.svg)](https://access.redhat.com/documentation/en-us/reference_architectures/2018/html/deploying_and_managing_openshift_3.9_on_google_cloud_platform/components_and_considerations#bastion_instance). | B1_4X16X100         |
| `datacenter`      | Enter the zone where you want to provision your IBM Cloud classic infrastructure. To find existing zones, run `ibmcloud ks zones`. | dal12               |
| `ibm_sl_api_key`  | The IBM Cloud classic infrastructure API key to access classic infrastructure resources. Do not enter this information in this file. Instead, you are prompted to enter this information when you create the classic infrastructure resources. To retrieve your API key, see [Managing classic infrastructure API keys](https://cloud.ibm.com/docs/iam?topic=iam-classic_keys). | n/a                 |
| `ibm_sl_username` | The IBM Cloud classic infrastructure user name to access classic infrastructure resources. Do not enter this information in this file. Instead, you are prompted to enter this information when you create the classic infrastructure resources. To retrieve your user name, see [Managing classic infrastructure API keys](https://cloud.ibm.com/docs/iam?topic=iam-classic_keys). | n/a                 |
| `infra_count`     | Enter the number of infrastructure nodes that you want to create. Infrastructure nodes are used to run infrastructure-related pods, such as router or registry pods. | 1                   |
| `master_count`    | Enter the number of master nodes that you want to create. The master runs the API server, controller manager server, and etcd database instance. | n/a                 |
| `pool_id`         | The Red Hat pool ID that is linked to the subscription that you set up with Red Hat. Do not enter this information here. Instead, follow the steps in [Lesson 3](https://cloud.ibm.com/docs/terraform/tutorials?topic=terraform-redhat#deploy_openshift) to retrieve your pool ID and provide the pool ID during the OpenShift installation. | n/a                 |
| `rhn_password`    | The [Red Hat Network password ![External link icon](https://cloud.ibm.com/docs-content/v1/content/6d69c89aece95412281779ad9e9aee96fcb73b84/icons/launch-glyph.svg)](https://www.redhat.com/apps/register/connect/) to access the OpenShift project. You can enter this information here or provide it as part of your OpenShift deployment in [Lesson 2](https://cloud.ibm.com/docs/terraform/tutorials?topic=terraform-redhat#provision_infrastructure). | n/a                 |
| `rhn_username`    | The [Red Hat Network user name with OpenShift subscription ![External link icon](https://cloud.ibm.com/docs-content/v1/content/6d69c89aece95412281779ad9e9aee96fcb73b84/icons/launch-glyph.svg)](https://www.redhat.com/apps/register/connect/) to access the OpenShift project. You can enter this information here or provide it as part of your OpenShift deployment in [Lesson 2](https://cloud.ibm.com/docs/terraform/tutorials?topic=terraform-redhat#provision_infrastructure). | n/a                 |
| `private_vlanid`  | Enter the VLAN ID of your existing private VLAN that you want to use. To find existing VLAN IDs, run `ibmcloud sl vlan list` and review the **ID** column. | n/a                 |
| `public_vlanid`   | Enter the VLAN ID of your existing public VLAN that you want to use. To find existing VLAN IDs, run `ibmcloud sl vlan list` and review the **ID** column. | n/a                 |
| `ssh_label`       | Enter a label to assign to your SSH key.                     | ssh_key_openshift   |
| `ssh_private_key` | Enter the path to the SSH private key that you created earlier. | `~/.ssh/id_rsa`     |
| `ssh_public_key`  | Enter the path to the SSH public key that you created earlier. | `~/.ssh/id_rsa.pub` |
| `storage_count`   | Decide whether you want to configure your OpenShift cluster with GlusterFS. Enter 0 to configure your OpenShift cluster without GlusterFS, and 3 or more to set up your OpenShift cluster with GlusterFS. | 0                   |
| `storage_flavor`  | If you configure your OpenShift cluster with GlusterFS, enter the flavor that you want to use for your storage virtual machine. Each storage node mounts three block storage devices that host the Red Hat Gluster Storage. You can use the combination of compute capacity and local Gluster storage to run a hyper-converged deployment where your apps are placed on the same node as the app's persistent storage. | B1_4X16X100         |
| `subnet_size`     | Enter the number of subnets that you want to be able to create with your public and private VLAN. This value is required only if you decide to create a new private and public VLAN pair. | 64                  |
| `vlan_count`      | Enter `1` to automatically create a new private and public VLAN, or `0` if you want to use existing VLANs. To find existing VLANs, run `ibmcloud sl vlan list`. The zone where your existing VLAN routers are provisioned is included in the **primary_router** column of your CLI output. | 1                   |
| `vm_domain`       | Enter the domain name that you want to use for your virtual machine nodes. | `ibm.com`           |

> **Important Note** : Generally we change the following variables:
>
> - the **hostname_prefix** to something easy to remember
> - the **app_count** to 2 or more
> - the **master_flavor** to B1_8X16X100 or more



# 2. Provision your cluster

Retreive your infrastructure username and apikey

From the OpenShift installation directory `/go/bin/terraform-ibm-openshift` inside your container, create the IBM Cloud classic infrastructure components for your Red Hat OpenShift cluster. When you run the command, Terraform evaluates what components must be provisioned and presents an execution plan. You must confirm that you want to provision the classic infrastructure resources by entering **yes**. During the provisioning, Terraform creates another execution plan that you must approve to continue. When prompted, enter the classic infrastructure user name and API key that you retrieved earlier. The provisioning of your resources takes about 40 minutes.

You may have to enter your **ibm_sl_api_key** and **your ibm_sl_username** two times during the procedure.

```
make rhn_username=<rhn_username> rhn_password=<rhn_password> infrastructure
```

Results:

```shell
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Outputs:

app_hostname = [
    niceocp-4665d30461-app-0,
    niceocp-4665d30461-app-1
]
app_private_ip = [
    10.72.23.134,
    10.72.23.172
]
bastion_hostname = niceocp-4665d30461-bastion
bastion_private_ip = 10.72.23.132
bastion_public_ip = 158.176.125.46
domain = IBM-OpenShift.cloud
infra_hostname = [
    niceocp-4665d30461-infra-0
]
infra_private_ip = [
    10.72.23.180
]
master_hostname = [
    niceocp-4665d30461-master-0
]
master_private_ip = [
    10.72.23.183
]
master_public_ip = [
    158.176.125.45
]
storage_hostname = []
storage_private_ip = []
```



Normally this is what have been created for you:

| Resource            | Flavor      | Description                                                  |
| ------------------- | ----------- | ------------------------------------------------------------ |
| Master node         | B1_4X16X100 | Three disks that are arranged as SAN with a total capacity of 100 GBDisk 1: 50 GBDisk 2: 25 GBDisk 3: 25 GB |
| Infrastructure node | B1_2X4X100  | Three disks that are arranged as SAN with a total capacity of 100 GBDisk 1: 50 GBDisk 2: 25 GBDisk 3: 25 GB |
| App nodes           | B1_2X4X100  | Three disks that are arranged as SAN with a total capacity of 100 GBDisk 1: 50 GBDisk 2: 25 GBDisk 3: 25 GB |
| Bastion node        | B1_2X2X100  | Two disks with the following capacity:Disk 1: 100 GBDisk 2: 50 GB |
| Storage nodes       | B1_4X16X100 | Two disks with the following capacity:Disk 1: 100 GBDisk 2: 50 GB |

Validate your deployment:

```shell
terraform show
```



# 3. Deploy Red Hat OpenShift Container Platform (OCP)

Deploy Red Hat OpenShift Container Platform on the IBM Cloud classic infrastructure resources that you created earlier.

During the deployment the following cluster components are set up and configured:

- 1 OpenShift Container Platform master node
- 1 OpenShift Container Platform infrastructure nodes
- 2 OpenShift Container Platform application nodes
- 1 OpenShift Container Platform Bastion node

For more information about Red Hat OpenShift Container Platform components, 

Retrieve the pool ID for your Red Hat account.



### Login to Bastion

```shell
ssh root@$(terraform output bastion_public_ip)
```

Results:

```shell
 # ssh root@$(terraform output bastion_public_ip)
The authenticity of host '158.176.125.46 (158.176.125.46)' can't be established.
ECDSA key fingerprint is SHA256:BBYZzuGwLShcxSwpt3Ht7Y4rJsVw3VISDG15c2NCfA0.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '158.176.125.46' (ECDSA) to the list of known hosts.
Last login: Sat Sep 28 15:40:43 2019 from i16-les01-ix2-62-35-133-241.sfr.lns.abo.bbox.fr

```



### Subscription

Remove any previous registration of the Bastion node.

```shell
subscription-manager unregister
```

Results:

```shell
# subscription-manager unregister
Unregistering from: rhncaplon0601.service.networklayer.com:8443/rhsm
System has been unregistered.

```

Import the `gpg` public key for Red Hat by using the Red Hat Package Manager:

```shell
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```

Register your Bastion node with the Red Hat Network. Enter the user name and password for your Red Hat account:

```shell
subscription-manager register --serverurl subscription.rhsm.redhat.com:443/subscription --baseurl cdn.redhat.com
```

When requested, enter your Red Hat username (shortname) and password:

```shell
# subscription-manager register --serverurl subscription.rhsm.redhat.com:443/subscription --baseurl cdn.redhat.com
Registering to: subscription.rhsm.redhat.com:443/subscription
Username: philippe
Password: 
The system has been registered with ID: 4fa632ba-6c4f-40bf-bf9a-af0feacd04bc
The registered system name is: niceocp-4665d30564-bastion.IBM-OpenShift.cloud
```

To get the **pool ID**, use this command and locate the first Pool ID (9a85f99a6cbfea02016d0679de30261d)

```
subscription-manager list --available --matches '*OpenShift Container Platform*'
```

Results:

```
# subscription-manager list --available --matches '*OpenShift Container Platform*'
+-------------------------------------------+
    Available Subscriptions
+-------------------------------------------+
Subscription Name:   OpenShift Business Partner Self-Supported NFR
Provides:            Red Hat CodeReady Linux Builder for x86_64
                     Red Hat CodeReady Linux Builder for Power, little endian
                     Red Hat OpenShift Enterprise Infrastructure
                     Red Hat Enterprise Linux Fast Datapath
                     Red Hat JBoss Core Services
                     JBoss Enterprise Application Platform
                     Red Hat CloudForms
                     Red Hat CodeReady Workspaces for OpenShift
                     Red Hat OpenShift Enterprise JBoss EAP add-on
                     Red Hat OpenShift Container Platform
                     Red Hat CoreOS
                     Red Hat OpenShift Enterprise Client Tools
                     Red Hat CoreOS Beta
                     Red Hat Openshift Serverless
                     Red Hat CloudForms Beta
                     Red Hat Software Collections (for RHEL Server)
                     Red Hat OpenShift Enterprise Application Node
                     Red Hat Enterprise Linux Atomic Host
                     Red Hat JBoss AMQ Clients
                     Red Hat Enterprise Linux Fast Datapath Beta for x86_64
                     Red Hat Software Collections Beta (for RHEL Server)
                     Red Hat Enterprise Linux Server
                     Red Hat Enterprise Linux for x86_64
                     JBoss Enterprise Web Server
                     Red Hat Container Native Virtualization
SKU:                 SER0480
Contract:            12002799
Pool ID:             9a85f99a6cbfea06616d0679de30261d
Provides Management: Yes
Available:           Unlimited
Suggested:           1
Service Level:       Self-Support
Service Type:        L1-L3
Subscription Type:   Stackable
Starts:              09/06/2019
Ends:                09/06/2020
System Type:         Virtual

```



Exit the secure shell to return to your OpenShift installation directory **inside your container**.

```shell
exit
```

Results:

```shell
logout
Connection to 169.47.XXX.XX closed.
/go/bin/terraform-ibm-openshift #
```



### Register

Finish setting up and registering the nodes with the Red Hat Network (You may have to enter your **ibm_sl_api_key** and **your ibm_sl_username** two times during the procedure):

```shell
make rhn_username=<rhn_username> rhn_password=<rhn_password> pool_id=<pool_ID> rhnregister
```

When you create the nodes, the following software and Red Hat subscriptions are automatically downloaded and installed on the nodes for you:

**Software packages:**



| Software                                               | Version                    |
| ------------------------------------------------------ | -------------------------- |
| Red Hat® Enterprise Linux 7.7 x86_64                   | 3.10.0-957.21.3.el7.x86_64 |
| Atomic-OpenShift (`master/clients/node/sdn-ovs/utils`) | 3.11.x.x                   |
| Docker                                                 | 1.13.1                     |
| Ansible                                                | 2.6.1                      |

**Red Hat subscription channels and rpm packages:**



| Channel                                                   | Repository Name             |
| --------------------------------------------------------- | --------------------------- |
| Red Hat® Enterprise Linux 7 Server (RPMs)                 | rhel-7-server-rpms          |
| Red Hat® OpenShift Enterprise 3.11 (RPMs)                 | rhel-7-server-ose-3.11-rpms |
| Red Hat® Enterprise Linux 7 Server - Extras (RPMs)        | rhel-7-server-extras-rpms   |
| Red Hat® Enterprise Linux 7 Server - Fast Datapath (RPMs) | rhel-7-fast-datapath-rpms   |



###Deploy OpenShift

Prepare the master, infrastructure, and application nodes for the OpenShift installation.

```shell
make openshift
```

Results after 27 minutes:

```shell
module.openshift.null_resource.deploy_cluster (remote-exec): TASK [openshift_cluster_autoscaler : Ensure the cluster-autoscaler is present] ***
module.openshift.null_resource.deploy_cluster (remote-exec): skipping: [nice-cluster-3f022ac5fd-master-0.IBM-OpenShift.cloud]

module.openshift.null_resource.deploy_cluster (remote-exec): PLAY [Cluster Auto Scaler Install Checkpoint End] ******************************

module.openshift.null_resource.deploy_cluster (remote-exec): TASK [Set Cluster Auto Scaler install 'Complete'] ******************************
module.openshift.null_resource.deploy_cluster (remote-exec): skipping: [nice-cluster-3f022ac5fd-app-0.IBM-OpenShift.cloud]

module.openshift.null_resource.deploy_cluster (remote-exec): PLAY RECAP *********************************************************************
module.openshift.null_resource.deploy_cluster (remote-exec): localhost                  : ok=11   changed=0    unreachable=0    failed=0
module.openshift.null_resource.deploy_cluster (remote-exec): nice-cluster-3f022ac5fd-app-0.IBM-OpenShift.cloud : ok=134  changed=15   unreachable=0    failed=0
module.openshift.null_resource.deploy_cluster (remote-exec): nice-cluster-3f022ac5fd-app-1.IBM-OpenShift.cloud : ok=122  changed=15   unreachable=0    failed=0
module.openshift.null_resource.deploy_cluster (remote-exec): nice-cluster-3f022ac5fd-infra-0.IBM-OpenShift.cloud : ok=122  changed=15   unreachable=0    failed=0
module.openshift.null_resource.deploy_cluster (remote-exec): nice-cluster-3f022ac5fd-master-0.IBM-OpenShift.cloud : ok=672  changed=170  unreachable=0    failed=0


module.openshift.null_resource.deploy_cluster (remote-exec): INSTALLER STATUS ***************************************************************
module.openshift.null_resource.deploy_cluster (remote-exec): Initialization              : Complete (0:00:36)
module.openshift.null_resource.deploy_cluster (remote-exec): Health Check                : Complete (0:00:02)
module.openshift.null_resource.deploy_cluster (remote-exec): Node Bootstrap Preparation  : Complete (0:07:41)
module.openshift.null_resource.deploy_cluster (remote-exec): etcd Install                : Complete (0:00:49)
module.openshift.null_resource.deploy_cluster (remote-exec): Master Install              : Complete (0:04:12)
module.openshift.null_resource.deploy_cluster (remote-exec): Master Additional Install   : Complete (0:05:13)
module.openshift.null_resource.deploy_cluster (remote-exec): Node Join                   : Complete (0:01:30)
module.openshift.null_resource.deploy_cluster (remote-exec): Hosted Install              : Complete (0:01:08)
module.openshift.null_resource.deploy_cluster (remote-exec): Web Console Install         : Complete (0:00:37)
module.openshift.null_resource.deploy_cluster (remote-exec): Console Install             : Complete (0:00:26)
module.openshift.null_resource.deploy_cluster (remote-exec): metrics-server Install      : Complete (0:00:01)
module.openshift.null_resource.deploy_cluster (remote-exec): Service Catalog Install     : Complete (0:03:30)
module.openshift.null_resource.deploy_cluster: Creation complete after 27m49s (ID: 5669036932986517579)
```

Tips:

If the installation fails with the error `module.post_install.null_resource.post_install: error executing "/tmp/terraform_1700732344.sh": wait: remote command exited without exit status or exit signal`, go to the [IBM Cloud classic infrastructure console](https://cloud.ibm.com/classic), and click **Devices** > **Device List**. Then, find the affected virtual server and from the actions menu, perform a soft reboot.

Errors:

If you get any **errors** concerning **skopeo** or **cockpit-ws** (time-out for instance). 

```shell
# On all VMs including the master
yum install skopeo
# On the master only 
yum install cockpit-ws
# and then restart the installation on Bastion
make openshift
```



# 4. Post Installation

Follow instruction on :

<https://cloud.ibm.com/docs/terraform/tutorials?topic=terraform-redhat>

After installation, you need to perform a few tasks to get access to the console. 

<https://docs.openshift.com/container-platform/3.11/admin_guide/manage_users.html>

### Update /etc/hosts on you laptop

Add the following line to /etc/hosts on your laptop (don't forget the domain at the end). You can get the master name and domain name in the /etc/hosts of the bastion or the master node.

<master_ip>   <master_name>.<domain>

Example:

```
158.176.105.4 nice-cluster-3f022ac5fd-master-0.IBM-OpenShift.cloud
```

To login, use the following user/password : admin / test123



### OpenShift Web Console

```http
https://master-name.domain-name:8443
```

Example

[https://nice-cluster-3f022ac5fd-master-0.IBM-OpenShift.cloud:8443](https://nice-cluster-3f022ac5fd-master-0.ibm-openshift.cloud:8443/)



###Change admin password : 

(<https://docs.openshift.com/container-platform/3.6/getting_started/configure_openshift.html>)

```shell
#ssh to the master node
ssh root@<master_ip>
cd /etc/origin/master
# Change the admin password
htpasswd htpasswd admin
new password: Admin1!
Re-type new password: Admin1!
```



### Change to full cluster role

```
oc adm policy add-cluster-role-to-user cluster-admin admin
```



###Add a basic user

```shell
oc login -u system:admin
cd /etc/origin/master
# create a new user philippe
oc create user philippe
# create identity to user philippe
oc create identity htpasswd_auth:philippe
# create a userid entity mapping to philippe
oc create useridentitymapping htpasswd_auth:philippe philippe
# Create a password to user philippe in the /etc/origin/master/htpasswd file
htpasswd htpasswd philippe
new password: philippe1!
Re-type new password: philippe1!
# login to philippe
oc login -u philippe
```









# Appendix

Find below an example of all commands to be used : 

**In a Local terminal:**

``` 
docker pull ibmterraform/terraform-provider-ibm-docker
docker run -it ibmterraform/terraform-provider-ibm-docker:latest

```

**Inside the container:**

```
apk add --no-cache openssh
git clone https://github.com/IBM-Cloud/terraform-ibm-openshift.git
cd terraform-ibm-openshift
ssh-keygen -t rsa -b 4096 -C "philippe@fr.ibm.com"
cd /go/bin/terraform-ibm-openshift
vi variables.tf
```

**Inside variables.tf**

```
variable "hostname_prefix"{
default = "IBM-OCP" with "niceocp-alpha"

variable "app_count" {
  Change default = 1  with 2

variable master_flavor {
   Change default = "B1_4X16X25"  with   "B1_8X16X100"

```

**Save the file and provision the VMs**

```
make rhn_username=phthom1 rhn_password=3fv_LjduwZt!HMuq7xrm infrastructure

  softlayer_api_key  = c96ea0b807aa8273aa1e27729b849666a99815f0cdffe1bd69f3c9decf3790ed
  softlayer_username = IBM808531

terraform show
```

**Register the subscriptions in the bastion**

```
ssh root@$(terraform output bastion_public_ip)
subscription-manager unregister
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
subscription-manager register --serverurl subscription.rhsm.redhat.com:443/subscription --baseurl cdn.redhat.com

phthom1 
3fv_Luazfhauhfauzhfq7xrm 

subscription-manager list --available --matches '*OpenShift Container Platform*'

Locate the Pool ID for the first occurence the 10 cores: 
8a85f99a6cbfgduya16d0679e0fe2623

exit (from bastion)
```

**Register and Deploy the cluster**

``` 
make rhn_username=phthom1 rhn_password=3fv_Ljduquytyutjhqfuq7xrm pool_id=8a85f99auygygu6dqsdkh016d0679e0fe2623 rhnregister

On all VMs:
yum -y install skopeo
On the master: 
yum -y install cockpit-ws

make openshift

```





**End of Installation**















