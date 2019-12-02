

# OPENSHIFT 3.11 - AUTOMATED

---



#Installing OCP 3.11 and CP4A on IBM Cloud Infrastructure with Terraform

The purpose of this document is to explain how you can automate the installation of Cloud Pak for Application and the most important, how you can also automate the installation of OpenShift on the IBM Cloud Infrastructure. My idea was first to prepare workshop and to make possible some customizations and controls over **OpenShift 3.11**

Other information can be fount in the following web site: <https://cloud.ibm.com/docs/terraform?topic=terraform-redhat>

**All credentials in this document are fake.**





# Task #1 Credentials

Before starting anything, you have to prepare a quite long list of **credentials** that you will change later in the automated.sh script.

1. General **IBM Cloud account** with the capability to order infrastructure (especially shared VMs):
   - **apikey** : this is a json file (like for instance mykey.json) that you can download from the IBM Cloud console from your account - <https://cloud.ibm.com/docs/iam?topic=iam-userapikey#userapikey> 
   -  **account number** : you will need it when using ibmcloud command line. See <https://cloud.ibm.com/account/settings>
   - Here is an example in the script:

```
export IBMACCOUNT='--apikey @./mykey.json -c 8bb3642c049G4587t18bac0779dfb5d0'
```



2. **IBM Cloud Infrastructure UserID** and Password :

- API Classic infrastructure UserID and Password can be found in the IBM Cloud Console : <https://cloud.ibm.com/iam/apikeys>
- Here is an example in the script:

```
export SLU='philippe'
export SLP='c96ea0b807aa8273aa1e27uegfgefuya99815f0cdffe1bd69f3c9decf3790ed'
```



3. **Red Hat UserID and Password**
   - These are the UserID and Password used for subscriptions (in my case NFR but this can be any kind of subscriptions)
   - You should also know the Pool ID which is a number where you are going to get the subscriptions for OCP. You can find this number here : <https://access.redhat.com/management/products>. Search for [Red Hat OpenShift, Standard Support (10 Cores, NFR, Partner Only)](https://access.redhat.com/management/subscriptions/product/SER0423). Then under your subscription number, find the Pool IDs, and in the master Pool, you will get a long key like : 8a85f99a6cbfea02015d0679e1352629  for example.
   - Here is an example in the script:

```
export RHNU=thomas
export RHNP1='3fv_Ljdiua6zfuMuq7xrm'
export POLL='8a85f99a6cbfk67666d0679e0fe2623'
```



4. Cloud Pak for Applications **Entitlement Key** (only if you are installing CP4A)
   - Get the entitlement key that is assigned to your ID.
     1. Log in to [**MyIBM Container Software Library** ![External link icon](https://www.ibm.com/support/knowledgecenter/SSCSJL/images/icons/launch-glyph.svg)](https://myibm.ibm.com/products-services/containerlibrary) with the IBMid and password that are associated with the entitled software.
     2. In the **Entitlement keys** section, select **Copy key** to copy the entitlement key to the clipboard.

``` 
export ERK=GIlM9-1Y7IGAqKO5-NFuyegzry875Mnlxen957jHZp
```



5. Create a **ssh key** pair and register that key in IBM Cloud

   Now you will create an ssh key pair and register it with IBM Cloud (you can skip this section if you already get one ssh key pair )

   <https://cloud.ibm.com/docs/cli?topic=cloud-cli-sl-manage-security-keys>

   It is very important to use **one ssh key** when you want to manage **multiple virtual servers** and devices. 

   The name you give to the key in IBM Cloud has to be unique. This can be tricky if you are using a shared account. You can't just call it ssh-key.

   For now, use your name and some numbers  to make the key ID unique. For the following steps, replace 'shortname_ssh_key' with your own actual key ID. For this example, we choose **eagk**.

   > On windows, you may need to install OpenSSH separately (check `ssh` command in the CLI ).

   On the command line, enter the following commands :

   ```bash
   cd
   mkdir keys
   cd keys
   ssh-keygen -f eagk -P ""
   ```

   You will end up with two files in the current directory, **eagk** (the private key) and **eagk.pub** (the public key). Next, you'll register the public key with IBM Cloud by executing :

   `ibmcloud sl security sshkey-add eagk -f eagk.pub --note eagk`

   > Note: choose a name for your key - don't use eagk

   Then list all your ssh keys with the following command :

   `ibmcloud sl security sshkey-list`

   Results :

   ```bash
   ibmcloud sl security sshkey-list
   ID        label                fingerprint                                       note   
   920557    eagk                 ba:18:fc:54:82:ee:f3:47:a2:be:91:24:72:fa:d4:b4   -   
   1146085   Temp Public Key      37:cd:69:ef:08:7c:42:36:92:31:a3:33:e9:38:45:eb   -   
   842075    Temp Public Key      1a:83:27:96:d7:ab:8f:bf:af:21:eb:b7:28:cd:87:71   -   
   842077    CAM Public Key       94:4d:f0:8b:89:72:61:7f:a9:9a:87:b4:7a:a5:6c:b7   -   
   920445    CAM Public Key       b3:c4:47:8c:cb:dc:50:dc:27:11:d9:b8:15:15:57:55   -   
   921603    Temp Public Key      f1:63:cd:ff:a9:22:d1:b5:8b:61:c8:c4:41:0a:75:a4   -   
   1150051   Temp Public Key      02:b2:66:e4:52:e4:ef:ce:ab:1c:49:08:f5:ac:a9:35   -   
   1145561   Temp Public Key      06:ea:10:41:1d:53:c1:42:22:32:fd:4e:6d:41:ab:f9   -   
   
   ```

   Locate your **key** and take a note of the sshkey ID (**920557**).



   6. **Summary**

   At the end of this task, you should have set the following variables (all values are similar of what you can find for real)

   ```
   export IBMACCOUNT='--apikey @./SL017487K.json -c 8bb3642c049f5876788bac0779dfb5d0'
   export SLU='philippe'
   export SLP='c96ea0b807aa8273aa1e27729b846787877815f0cdffe1bd69f3c9decf3790ed'
   export RHNU=philou
   export RHNP1='3fv_LjdUYUYGjhMuq7xrm'
   export POLL='8a85f99a6cbfeb05016d0679e0fe2623'
   export EAGK='-o StrictHostKeyChecking=no -i ./eagk'
   export ERK=GIlM9-1Y7IGAqKO5-NFb01HHJTYGx5Mnlxen957jHZp
   ```

   In a directory, you should have **3 keys** : eagk, eagk.pub, mykey.json

   **All credentials in this document are fake.**



   # Task #2 Create a NFS-VM

   Now that you know all your credential, we are going to create a VM to be the **starting point** to create any OpenShift cluster in IBM Cloud. Also this VM will serve as **NFS** storage for all clusters if necessary. 

   We call that VM : the NFS-VM. You only need **one NFS-VM** if necessary.

   Choose and create a VM with the following characteristics in IBM Cloud :

   - On the IBM Cloud, on your account (the same that we identified above), go to the catalog and look for **Virtual Server** (multitenant is fine)

   - **B1_4X8X100** (4 vcpu, 8 GB ram, 100 GB disk) - more disk can be added if necessary.

   - Choose a **datacenter** that will be the same as your future OCP clusters.

   - Choose **RHEL 64 latest version** (7.7 is fine)

   - use the **ssh key** (eagk) that we created above

   - choose a **name** that is meaningful for your NFS-VM

   - the other characteristics are not relevant for what we want to do.


   Login to your new system (NFS-VM) with your ssh key:

   ```
   ssh -i ./eagk root@<ipaddress>
   ```



   After creation, Install the following software on this server:

   - git, figlet, wget, unzip:

     ```
     yum install git figlet wget unzip nano
     ```

   - Install **terraform** version 0.11.8 - please follow that link https://cloud.ibm.com/docs/terraform?topic=terraform-getting-started#install
   - install ibm cloud **plugin** for terraform (in the same section as just above)
   - install **ibmcloud CLI** latest version (see <https://github.com/IBM-Cloud/ibm-cloud-cli-release/releases/>)



   Finally you should **configure your NFS** with the following instructions:

   ```
   yum install -y nfs-utils
   systemctl enable rpcbind
   systemctl enable nfs-server
   systemctl start rpcbind
   systemctl start nfs-server
   mkdir /var/data
   chmod -R 755 /var/data
   ```

   To allow all clients put *

   ```
   # vi /etc/exports
   /var/data *(rw,sync,no_root_squash)
   ```

   Restart NFS

   ``` 
   # systemctl restart nfs-server
   ```

   You are now ready to install your first cluster.



   # Task #3 Customize the script

   Be sure you are still logged in to your NFS-VM as root.

   Copy or create the **automated.sh** script to the /root directory (see below the script)

   Create a new directory:

   ```
   cd
   mkdir keys
   cd keys 
   ```

   Copy  the 3 keys **eagk, eagk.pub, mykey.json** to the **keys** directory.

   Edit the script:

   ```
   cd
   nano automated.sh
   ```

   Change the following variables with yours (section Security in the script)

   You should have set the following variables (all values are similar of what you can find for real):

   ```
   export IBMACCOUNT='--apikey @./SL017487K.json -c 8bb3642c049f5876788bac0779dfb5d0'
   export SLU='philippe'
   export SLP='c96ea0b807aa8273aa1e27729b846787877815f0cdffe1bd69f3c9decf3790ed'
   export RHNU=thomas
   export RHNP1='3fv_LjdUYUYGjhMuq7xrm'
   export POLL='8a85f99a6cbfeb05016d0679e0fe2623'
   export EAGK='-o StrictHostKeyChecking=no -i ./eagk'
   export ERK=GIlM9-1Y7IGAqKO5-NFb01HHJTYGx5Mnlxen957jHZp
   ```

   Review also some other parameters that you may change like the datacenter (by default **lon06**). 

   Change automated.sh to be executable:

   ```
   chmod +x automated.sh
   ```





   #Task #4 Deploy a first cluster

   The automated.sh script will first create a new directory on the NFS-VM. The **directory** name will be also be the **prefix name for all your VM**s in the cluster. 

   The automated.sh script takes **3 required parameters**:

   - the name of the **directory** (and also the prefix for the VMs in your cluster)
   - the admin **password** (don't put any special caracters)
   - the **number** of **app** nodes (or compute nodes) greater or equal to 2

   ```
   ./automated.sh <mydirectoryname> <admin-password> <number-app-nodes>
   ```

   To run disconnected and to save the log, type the folowing commands (tail will show you the latest lines in the log):

   ```
   ./automated.sh <dir> <admin-password> <number> &> /tmp/<dir>.log &
   tail -f /tmp/<dir>.log
   ```

   The installation last **100 minutes** for a minimal cluster of **5 nodes** (1 master, 1 bastion, 1 Infra, 2 app). It could be more than 2 hours for 10 nodes. 

   The final result could be :

   ```bash
   Install successful *************************************************************
   
   Installation complete.
   
   Please see https://ibm-cp-applications.apps.158.176.72.172.xip.io to get started and learn more about IBM Cloud Pak for Applications.
   
   The Tekton Dashboard is available at: https://tekton-dashboard-kabanero.apps.158.176.72.172.xip.io.
   
   The IBM Transformation Advisor UI is available at: https://ta.apps.apps.158.176.72.172.xip.io.
   
    _____           _           ____                      
   | ____|_ __   __| |         |  _ \ ___  ___ __ _ _ __  
   |  _| | '_ \ / _` |  _____  | |_) / _ \/ __/ _` | '_ \ 
   | |___| | | | (_| | |_____| |  _ <  __/ (_| (_| | |_) |
   |_____|_| |_|\__,_|         |_| \_\___|\___\__,_| .__/ 
                                                   |_|    
   --------------------------------------------------------------------------------------
   https://nicejm0.ibm.ws:8443
   158.176.72.170 nicejm0 nicejm0.ibm.ws
   DynamicClass: managed-nfs-storage
    
   93574332   niceja0                                                  158.176.72.163       
   93574334   niceja1                                                  158.176.72.164       
   93574328   nicejb                                                   158.176.72.162       
   93574336   niceji0                                                  158.176.72.172       
   93574330   nicejm0                                                  158.176.72.170       
    
   Active PODS (more than 73 for CP4A) :
   75
   --------------------------------------------------------------------------------------
    _ _       _  _  ____              ____   ___      
   / | |__  _| || ||___ \ _ __ ___  _| ___| / _ \ ___ 
   | | '_ \(_) || |_ __) | '_ ` _ \(_)___ \| | | / __|
   | | | | |_|__   _/ __/| | | | | |_ ___) | |_| \__ \
   |_|_| |_(_)  |_||_____|_| |_| |_(_)____/ \___/|___/
                                                                
   
   ```

   The output log could be very big : more than 24k lines and depends on the number of app nodes.



   The script is composed of 6 steps:

   - step0: preparation
   - step1: infrastructure provisioning
   - step2: RH Subscriprions for all the VMs in the OpenShift Cluster
   - step3: OCP cluster deploy
   - step4: Security (admin)
   - step5: NFS 
   - step6: CP4A installation on this cluster



   >  The last step (step6) can be skipped if necessary to use OpenShift for another Cloud Pak. Just delete the section 6 in the script but keep the last echos.



   # Troubleshooting

   In case of problem, review the log. Identify the step in the log. 

   Search the internet, OpenShift documentations and IBM doco for solutions. 

   I will put more stuff in here in the next version of the document. 

   I didn't test with multiple masters or infra nodes. I didn't test to get a bigger Infra node storage with **more than 100 GB** for the registry.



   # Appendix

   Find below the **automated.sh** script that you can cut & past.

   ```bash
   #!/bin/bash
   #############################################################
   #
   # Installation using a NFS-VM as the starting point 
   # Automated installation of OpenShift
   #
   #############################################################
   # Prepare the following files in the /root/keys of the NFS-VM
   # -----------------------------------------------------------
   ## eagk :     private key to get access to NFS-VM 
   ## eagk.pub : public key to get access to NFS-VM
   ## mykey.json : your key to access your IBM Cloud account
   # -----------------------------------------------------------
   # NFS VM already exists with git, figlet, wget, unzip, 
   # and terraform and IBM Cloud plugin, ibmcloud cli installed
   # -----------------------------------------------------------
   #############################################################
   # copy this content to a file : automated.sh     in /root
   # chmod +x automated.sh
   # ./automated.sh cluster1 Password1 2 &> /tmp/cluster1.log &
   # tail -f /tmp/cluster1.log
   
   SECONDS=0  # Start Chrono
   
   if [ -z "$1" ] 
   then
     echo "1-Prefix VM names"
     echo "2-admin password"
     echo "3-Number of Nodes"
     exit
   fi
   if [ -z "$2" ] 
   then 
     echo "2-admin password"
     echo "3-Number of Nodes"
     exit  
   fi
   if [ -z "$3" ] 
   then 
     echo "3-Number of Nodes"
     exit  
   fi
   export MYDIR=$1
   export PASS=$2
   export NODES=$3
   
   # -----------------------------------------------------------
   
   figlet "STEP0 - Prepare"
   duration=`printf '%dh:%dm:%ds\n' $(($SECONDS/3600)) $(($SECONDS%3600/60)) $(($SECONDS%60))`
   figlet "$duration"
   
   # Variables
   export MYNFS=`ip -4 addr show eth1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}'`
   export MYPAT="/root/$MYDIR/terraform-ibm-openshift"
   export MYCLONE='https://github.com/IBM-Cloud/terraform-ibm-openshift.git'
   export DOM='ibm.ws'
   export DC='lon06'
   export B0=${MYDIR}b  # Bastion Hostname
   export M0=${MYDIR}m0 # Master Hostname
   export I0=${MYDIR}i0 
   export DURLOG="duration-${MYDIR}.log"
   
   # Security
   export IBMACCOUNT='--apikey @./mykey.json -c 8bb3642c049f4455c18bac0779dfb5d0'
   export SLU='philippe'
   export SLP='c96ea0b807aa8273aa1e27729b867808a99815f0cdffe1bd69f3c9decf3790ed'
   export PROJ='adminproj'
   export RHNU=thomas
   export RHNP1='3fv_Lj6T6!HMuq7xrm'
   export POLL='8a85f99a6cbfea67816d0679e0fe2623'
   export RHNP=\'$RHNP1\'
   export EAGK='-o StrictHostKeyChecking=no -i ./eagk'
   export ERK=GIlM9-1Y7IGAqKO5-NFb0GTHyJGx5Mnlxen957jHZp
   
   
   # Directory scafolding
   rm -rf $MYDIR
   mkdir $MYDIR
   cd $MYDIR
   git clone $MYCLONE
   cd $MYPAT
   
   # Copy the 3 necessary keys from /root/keys into the current directory
   cp ~/keys/* .
   
   # Login to IBM Cloud
   ibmcloud login $IBMACCOUNT
   
   # Change variables.tf with values provided in variables
   cat <<XXXXXX > ./variables.tf
   variable "hourly_billing" {
     default = "true"
   }
   
   variable "datacenter" {
     default = "XXXDC"
   }
   
   variable "hostname_prefix"{
     default = "XXXHOST"
   }
   
   variable "bastion_count" {
     default = 1
   }
   
   variable "master_count" {
     default = 1
   }
   
   variable "infra_count" {
     default = 1
   }
   
   variable "app_count" {
     default = XXXNODES
   }
   
   variable "storage_count" {
     description = "Set to 0 to configure openshift without glusterfs"
     default = 0
   }
   
   variable "ssh_public_key" {
     default     = "./eagk.pub"
   }
   
   variable "ssh-label" {
     default = "ssh_key_terraform"
   }
   
   variable "vm_domain" {
     default = "XXXDOM"
   }
   
   
   variable "ibm_sl_username"{
     default = "XXXSLU"
   }
   
   
   variable "ibm_sl_api_key"{
     default = "XXXSLP"
   }
   
   variable "rhn_username"{
     default = "XXXRHNU"
   }                           
   
   variable "rhn_password"{
     default = "XXXRHNP1"
   }
   
   variable "pool_id"{
      default = ""
   }
   
   variable "private_ssh_key"{
     default     = "./eagk"
   }
   
   variable vlan_count {
     description = "Set to 0 if using existing and 1 if deploying a new VLAN"
     default = "1"
   }
   variable private_vlanid {
     description = "ID of existing private VLAN to connect VSIs"
     default = "2543851"
   }
   
   variable public_vlanid {
     description = "ID of existing public VLAN to connect VSIs"
     default = "2543849"
   }
   
   ### Flavors to be changed to actual values in '#...'
   
   variable bastion_flavor {
     default = "B1_2X4X25"
   }
   
   variable master_flavor {
      default = "B1_8X16X100"
   }
   
   variable infra_flavor {
      default = "B1_4X8X100"
   }
   
   variable app_flavor {
      default = "B1_4X8X100"
   }
   
   variable storage_flavor {
      default = "B1_4X8X100"
   }
   XXXXXX
   
   # Modify variables.tf
   sed -i "s/XXXDC/$DC/g" variables.tf              # DataCenter 
   sed -i "s/XXXHOST/$MYDIR/g" variables.tf         # Hostname Prefix 
   sed -i "s/XXXDOM/$DOM/g" variables.tf            # Domain
   sed -i "s/XXXNODES/$NODES/g" variables.tf        # Number of nodes 
   sed -i "s/XXXSLU/$SLU/g" variables.tf            # SoftLayer Username
   sed -i "s/XXXSLP/$SLP/g" variables.tf            # SoftLayer Password
   sed -i "s/XXXRHNU/$RHNU/g" variables.tf          # RH Username for Subscriptions
   sed -i "s/XXXRHNP1/$RHNP1/g" variables.tf        # RH Password for Subscriptions
   
   # -----------------------------------------------------------
   
   figlet "STEP1 - INFRA"
   duration=`printf '%dh:%dm:%ds\n' $(($SECONDS/3600)) $(($SECONDS%3600/60)) $(($SECONDS%60))`
   figlet "$duration"
   echo "$duration : STEP0 - PREPA"  > $DURLOG
   
   cd $MYPAT
   sed -i 's/-${var.random_id}-${var.bastion_hostname}/b/g' ./modules/bastion/bastion_node/bastion.tf
   grep -rnw './' -e '${var.bastion_hostname_prefix}'
   sed -i 's/-${var.random_id}-${var.master_hostname}-/m/g' ./modules/infrastructure/master_node/masternode.tf
   grep -rnw './' -e '${var.master_hostname_prefix}'
   sed -i 's/-${var.random_id}-${var.infra_hostname}-/i/g' ./modules/infrastructure/infra_node/infranode.tf
   grep -rnw './' -e '${var.infra_hostname_prefix}'
   sed -i 's/-${var.random_id}-${var.app_hostname}-/a/g' ./modules/infrastructure/app_node/appnode.tf
   grep -rnw './' -e '${var.app_hostname_prefix}'
   
   # INFRASTRUCTURE PROVISIONING
   cd $MYPAT
   make rhn_username=$RHNU rhn_password=$RHNP infrastructure
   ibmcloud sl vs list --column id --column hostname --column public_ip --column action | grep $MYDIR
   
   # IP Collect
   export IB0=`ibmcloud sl vs list | grep $B0 | awk '{print $6}'`
   export IM0=`ibmcloud sl vs list | grep $M0 | awk '{print $6}'`
   export II0=`ibmcloud sl vs list | grep $I0 | awk '{print $6}'`
   for host in $(ibmcloud sl vs list --column hostname | grep ${MYDIR}a); do echo "--- $host"; done
   
   
   # -----------------------------------------------------------
   
   figlet "STEP2 - Subscription"
   duration=`printf '%dh:%dm:%ds\n' $(($SECONDS/3600)) $(($SECONDS%3600/60)) $(($SECONDS%60))`
   figlet "$duration"
   echo "$duration : STEP1 - INFRA"  >> $DURLOG
   
   # Bastion
   ssh $EAGK root@$IB0 'subscription-manager unregister'
   ssh $EAGK root@$IB0 'rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release'
   ssh $EAGK root@$IB0 "subscription-manager register --serverurl subscription.rhsm.redhat.com:443/subscription --baseurl cdn.redhat.com --username $RHNU --password $RHNP"
   
   # RHN
   cd $MYPAT
   make rhn_username=$RHNU rhn_password=$RHNP pool_id=8a85f99a6cbfea02016d0679e0fe2623 rhnregister
   
   
   # Complementary modules
   ssh $EAGK root@$IM0 'yum -y install unzip nano'
   
   # IMPORTANT : to be manage by DNS and NetworkManager
   ssh $EAGK root@$IM0 'sed -i "s/NM_CONTROLLED=no/NM_CONTROLLED=yes/g" /etc/sysconfig/network-scripts/ifcfg-eth1'
   ssh $EAGK root@$II0 'sed -i "s/NM_CONTROLLED=no/NM_CONTROLLED=yes/g" /etc/sysconfig/network-scripts/ifcfg-eth1'
   
   for iphost in $(ibmcloud sl vs list --column hostname --column public_ip | grep ${MYDIR}a | awk '{print $2}')
       do 
         ssh $EAGK root@$iphost 'sed -i "s/NM_CONTROLLED=no/NM_CONTROLLED=yes/g" /etc/sysconfig/network-scripts/ifcfg-eth1' 
       done
   
   # -----------------------------------------------------------
   
   figlet "STEP3 - Deploy"
   duration=`printf '%dh:%dm:%ds\n' $(($SECONDS/3600)) $(($SECONDS%3600/60)) $(($SECONDS%60))`
   figlet "$duration"
   echo "$duration : STEP2 - Subscription" >> $DURLOG
   # Modification inventory.cfg
   
   ssh $EAGK root@$IM0 'hostnamectl'
   
   cd $MYPAT
   sed -i "s/openshift_cluster_monitoring_operator_install=false/openshift_cluster_monitoring_operator_install=true/g" $MYPAT/inventory_repo/inventory.cfg
   sed -i "s/# openshift_hosted_metrics_deploy=true/openshift_metrics_install_metrics=true/g" $MYPAT/inventory_repo/inventory.cfg
   
   # Deploy Cluster
   cd $MYPAT
   make openshift
   
   
   # -----------------------------------------------------------
   
   figlet "STEP4 - Security"
   duration=`printf '%dh:%dm:%ds\n' $(($SECONDS/3600)) $(($SECONDS%3600/60)) $(($SECONDS%60))`
   figlet "$duration"
   echo "$duration : STEP3 - Deploy" >> $DURLOG
   
   sleep 60 # wait that all running parts have finished successfully
   ssh $EAGK root@$IM0 "cd /etc/origin/master;htpasswd -b htpasswd admin ${PASS};oc adm policy add-cluster-role-to-user cluster-admin admin"
   export LOGIN="https://$M0.ibm.ws:8443"
   export ETCH="$IM0 $M0 $M0.ibm.ws"
   
   # -----------------------------------------------------------
   
   figlet "STEP5 - NFS"
   duration=`printf '%dh:%dm:%ds\n' $(($SECONDS/3600)) $(($SECONDS%3600/60)) $(($SECONDS%60))`
   figlet "$duration"
   echo "$duration : STEP4 - Security" >> $DURLOG
   
   
   cd $MYPAT
   cat <<XXXXXX > filecmd.sh
   oc login -u admin -p $PASS
   curl -L -o kubernetes-incubator.zip https://github.com/kubernetes-incubator/external-storage/archive/master.zip
   unzip -o kubernetes-incubator.zip > null
   cd /root/external-storage-master/nfs-client/
   oc new-project $PROJ
   oc project $PROJ
   sed -i "s/namespace: default/namespace: $PROJ/g" deploy/rbac.yaml
   oc create -f deploy/rbac.yaml
   oc adm policy add-scc-to-user hostmount-anyuid system:serviceaccount:$PROJ:nfs-client-provisioner
   sed -i "s/namespace: default/namespace: $PROJ/g" /root/external-storage-master/nfs-client/deploy/deployment.yaml
   sed -i "s/value: fuseim.pri\/ifs/value: $PROJ\/nfs/g" /root/external-storage-master/nfs-client/deploy/deployment.yaml
   sed -i "s/value: 10.10.10.60/value: $MYNFS/g" /root/external-storage-master/nfs-client/deploy/deployment.yaml
   sed -i "s/value: \/ifs\/kubernetes/value: \/var\/data/g" /root/external-storage-master/nfs-client/deploy/deployment.yaml
   sed -i "s/server: 10.10.10.60/server: $MYNFS/g" /root/external-storage-master/nfs-client/deploy/deployment.yaml
   sed -i "s/path: \/ifs\/kubernetes/path: \/var\/data/g" /root/external-storage-master/nfs-client/deploy/deployment.yaml
   sed -i "s/provisioner: fuseim.pri\/ifs/provisioner: $PROJ\/nfs/g" /root/external-storage-master/nfs-client/deploy/class.yaml
   echo '-1-'
   oc create -f deploy/class.yaml
   oc create -f deploy/deployment.yaml
   sleep 10
   echo '-2-'
   oc get pods
   oc create -f deploy/test-claim.yaml
   sleep 10
   oc get pvc
   echo '-3-'
   XXXXXX
   
   scp $EAGK ./filecmd.sh root@$IM0:/root/filecmd.sh
   ssh $EAGK root@$IM0 "chmod +x /root/filecmd.sh"
   ssh $EAGK root@$IM0 "./filecmd.sh"
   
   
   # -----------------------------------------------------------
   
   figlet "STEP6 - CP4A"
   duration=`printf '%dh:%dm:%ds\n' $(($SECONDS/3600)) $(($SECONDS%3600/60)) $(($SECONDS%60))`
   figlet "$duration"
   echo "$duration : STEP5 - NFS" >> $DURLOG
   
   
   cat <<'XXXXXX' > filecmd.sh 
   export ENTITLED_REGISTRY_KEY=$1
   export ENTITLED_REGISTRY=cp.icr.io
   export ENTITLED_REGISTRY_USER=ekey
   docker login "$ENTITLED_REGISTRY" -u "$ENTITLED_REGISTRY_USER" -p "$ENTITLED_REGISTRY_KEY"
   mkdir data
   docker run -v $PWD/data:/data:z -u 0 -e LICENSE=accept "$ENTITLED_REGISTRY/cp/icpa/icpa-installer:3.0.0.0" cp -r data/* /data
   sed -i 's/storageClassName: ""/storageClassName: "managed-nfs-storage"/g' data/transadv.yaml
   docker run -v ~/.kube:/root/.kube:z -u 0 -t -v $PWD/data:/installer/data:z -e LICENSE=accept -e ENTITLED_REGISTRY -e ENTITLED_REGISTRY_USER -e ENTITLED_REGISTRY_KEY "$ENTITLED_REGISTRY/cp/icpa/icpa-installer:3.0.0.0" install
   XXXXXX
   
   scp $EAGK ./filecmd.sh root@$IM0:/root/filecmd.sh
   ssh $EAGK root@$IM0 "chmod +x /root/filecmd.sh"
   ssh $EAGK root@$IM0 "./filecmd.sh $ERK"
   
   figlet "End - Recap"
   
   echo "------------------------------------------------------------------------------------------------"
   echo "$LOGIN"
   echo "$ETCH"
   echo "DynamicClass: managed-nfs-storage"
   echo ' '
   ibmcloud sl vs list --column id --column hostname --column public_ip --column action | grep $MYDIR
   echo ' '
   echo 'Active PODS (more than 73 for CP4A) :'
   ssh $EAGK root@$IM0 'oc get pods --all-namespaces |wc -l'
   echo "------------------------------------------------------------------------------------------------"
   
   echo "$LOGIN" > ${MYDIR}-status.txt
   echo "$ETCH"  > ${MYDIR}-status.txt
   echo "DynamicClass: managed-nfs-storage" > ${MYDIR}-status.txt
   
   duration=`printf '%dh:%dm:%ds\n' $(($SECONDS/3600)) $(($SECONDS%3600/60)) $(($SECONDS%60))`
   figlet "$duration"
   echo "$duration : STEP6 - CP4A" >> $DURLOG
   
   
   
   
   ```





    # END OF DOCUMENT






