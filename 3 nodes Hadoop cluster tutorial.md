<body style:"margin-left:150px,">
<style>
body {
margin : 50px 150px 50px 150px;
font-size:1.1em;
}
.indented-section {
margin-left : 50px;
}
</style>

# Setting up a 3 nodes Hadoop cluster : complete tutorial

<br><br>

- [Setting up a 3 nodes Hadoop cluster : complete tutorial](#setting-up-a-3-nodes-hadoop-cluster--complete-tutorial)
  - [1. Choosing your compute instances (servers)](#1-choosing-your-compute-instances-servers)
      - [1st instance : master node](#1st-instance--master-node)
      - [2nd and 3nd instances : slave nodes](#2nd-and-3nd-instances--slave-nodes)
  - [2. Setting up the servers so that they can communicate](#2-setting-up-the-servers-so-that-they-can-communicate)
    - [a. Set up hostnames on all 3 instances](#a-set-up-hostnames-on-all-3-instances)
    - [b. Set up ssh between master and slave instances](#b-set-up-ssh-between-master-and-slave-instances)
  - [3. Install Ambari on the master server](#3-install-ambari-on-the-master-server)
    - [a. Download the Ambari Repository](#a-download-the-ambari-repository)
    - [b. Install the Ambari Server](#b-install-the-ambari-server)
    - [c. Set Up the Ambari Server](#c-set-up-the-ambari-server)
  - [4. Open ports on your cloud provider (Azure, Google Cloud, AWS ...)](#4-open-ports-on-your-cloud-provider-azure-google-cloud-aws)
  - [5. Deploy your cluster](#5-deploy-your-cluster)
    - [a. Create the cluster](#a-create-the-cluster)
      - [Step 1 : Launch the Ambari Cluster Install Wizard](#step-1--launch-the-ambari-cluster-install-wizard)
      - [Step 2 : Name Your Cluster](#step-2--name-your-cluster)
      - [Step 3 : Select Version](#step-3--select-version)
      - [Step 4 : Install Options](#step-4--install-options)
      - [Step 5 : Confirm Hosts](#step-5--confirm-hosts)
      - [Step 6 : Choose Services](#step-6--choose-services)
      - [Step 7 : Assign Masters](#step-7--assign-masters)
      - [Step 8 : Assign Slaves and Clients](#step-8--assign-slaves-and-clients)
      - [Step 9 : Customize Services](#step-9--customize-services)
      - [Step 10 : Review](#step-10--review)
      - [Step 11 : Install](#step-11--install)
    - [b. Increase YARN capacity scheduler maximum resource percent](#b-increase-yarn-capacity-scheduler-maximum-resource-percent)

<br><br>

## 1. Choosing your compute instances (servers)

---

<div class="indented-section">

<br><br>

Pick Centos 7 instances (Red Hat 7 should also work).

Google Cloud platform provides 300\$ free credit and allows up to 8 cpu cores accross all instances. This isn't the case of all providers ; if you want to have solid compute instances for free, you should check google's cloud platform.

**Also remember to poweroff your instances when you don't use them, as they don't consume credit when they are turned off.**

<br>

#### 1st instance : master node

> At least 25 gb of ram

I personally used an instance with 32 gb of ram and 4 cores.

<br>

#### 2nd and 3nd instances : slave nodes

> At least 16 gb of ram

I used two instances with 16 gb of ram and 2 cores each.

<br><br>

</div>

## 2. Setting up the servers so that they can communicate

---
<br><br>

<div class="indented-section">

On all instances :

```
sudo su root
yum install nano -y
yum install wget -y
setenforce 0
umask 0022
echo umask 0022 >> /etc/profile
systemctl disable firewalld
service firewalld stop
```

You need to be root during the whole tutorial.

<br>

</div>

### a. Set up hostnames on all 3 instances

---
<br>

<div class="indented-section">

For our Twitter analysis application, I chose the following hostnames but you can choose any hostnames that you like. Please bear in mind that these hostnames are local but you don't need to buy real domain names.

master.twitter-multi.com <br/>
slave-1.twitter-multi.com<br/>
slave-2.twitter-multi.com

On master node :

```
hostname master.twitter-multi.com
nano /etc/hostname
```

Replace the current value with _master.twitter-multi.com_. Save and exit (ctrl + o and ctrl + x).

```
nano /etc/hosts
```

Your file should like this :

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.132.0.13 master.twitter-multi.com # this is the private IP of the server
34.76.236.54 slave-1.twitter-multi.com
35.195.162.0 slave-2.twitter-multi.com
169.254.169.254 metadata.google.internal  # Added by Google. You can ignore this line
```

On slave 1 :

```
hostname slave-1.twitter-multi.com
nano /etc/hostname
```

Replace the current value with _slave-1.twitter-multi.com_. Save and exit (ctrl + o and ctrl + x).

```
nano /etc/hosts
```

Your file should look like this :

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.132.0.14 slave-1.twitter-multi.com  # this is the private IP of the server
35.195.162.0 slave-2.twitter-multi.com
34.76.149.62 master.twitter-multi.com
169.254.169.254 metadata.google.internal  # Added by Google. You can ignore this line
```

On slave 2 :

```
hostname slave-2.twitter-multi.com
nano /etc/hostname
```

Replace the current value with _slave-2.twitter-multi.com_. Save and exit (ctrl + o and ctrl + x).

```
nano /etc/hosts
```

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.132.0.15 slave-2.twitter-multi.com # this is the private IP of the server
34.76.236.54 slave-1.twitter-multi.com
34.76.149.62 master.twitter-multi.com
169.254.169.254 metadata.google.internal  # Added by Google. You can ignore this line
```

</div>
<br>

### b. Set up ssh between master and slave instances

---
<div class="indented-section">
<br>

On all instances :

```
nano /etc/ssh/sshd_config
```

Change the corresponding lines to :

```

PermitRootLogin yes
PubkeyAuthentication yes
PasswordAuthentication yes
```

Save and exit (ctrl + o and ctrl + x).

```
service sshd restart
```

On master node :

```
cd
ssh-keygen # press enter to accept defaults
cd .ssh
cat id_rsa.pub >> authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
cat id_rsa.pub # shows the public key that will be copied an instances
```

On slave-1 and slave-2 :

```
mkdir .ssh
cd .ssh/
nano authorized_keys
```

Paste exactly the content from the _cat id_rsa.pub_ command on master node. Save and exit (ctrl + o and ctrl + x).

```
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

</div>
<br><br>

## 3. Install Ambari on the master server

---
<br>

<div class="indented-section">

We will now start the installation of the Hadoop cluster. Ambari is a useful tool to build the cluster automatically and manage it afterwards.<br/><br/>
The following steps follow the tutorial from the cloudera installation guide :
https://docs.cloudera.com/HDPDocuments/Ambari-2.7.4.0/bk_ambari-installation/content/ch_Installing_Ambari.html
<br/>
These steps are susceptible to change if you install another version of ambari.

Make sure you are still logged in as root (_sudo su root_).
Now let's get started !

</div>
<br>

### a. Download the Ambari Repository

---
<br>
<div class="indented-section">

Download the Ambari repository file to a directory on your master node.

```
wget -nv http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.7.4.0/ambari.repo -O /etc/yum.repos.d/ambari.repo
```

Confirm that the repository is configured by checking the repo list.

```
yum repolist
```

You should see values similar to the following for Ambari repositories in the list.

```
repo id                    repo name                                       status
ambari-2.7.4.0-118        ambari Version - ambari-2.7.4.0-118            12
epel/x86_64                Extra Packages for Enterprise Linux 7 - x86_64  11,387
ol7_UEKR4/x86_64           Latest Unbreakable Enterprise Kernel Release 4
                            for Oracle Linux 7Server (x86_64)   295
ol7_latest/x86_64          Oracle Linux 7Server Latest (x86_64)            18,642
puppetlabs-deps/x86_64     Puppet Labs Dependencies El 7 - x86_64          17
puppetlabs-products/x86_64 Puppet Labs Products El 7 - x86_64              225
 repolist: 30,578
```

</div>
<br>

### b. Install the Ambari Server

---

<br>
<div class="indented-section">

Install the Ambari bits. This also installs the default PostgreSQL Ambari database. The following steps are still only on master node.

```
yum install ambari-server
```

Enter y when prompted to confirm transaction and dependency checks.

A successful installation displays output similar to the following:

```
Installing : postgresql-libs-9.2.18-1.el7.x86_64         1/4
Installing : postgresql-9.2.18-1.el7.x86_64              2/4
Installing : postgresql-server-9.2.18-1.el7.x86_64       3/4
Installing : ambari-server-2.7.4.0-118.x86_64           4/4
Verifying  : ambari-server-2.7.4.0-118.x86_64           1/4
Verifying  : postgresql-9.2.18-1.el7.x86_64              2/4
Verifying  : postgresql-server-9.2.18-1.el7.x86_64       3/4
Verifying  : postgresql-libs-9.2.18-1.el7.x86_64         4/4

Installed:
  ambari-server.x86_64 0:2.7.4.0-118
Dependency Installed:
 postgresql.x86_64 0:9.2.18-1.el7
 postgresql-libs.x86_64 0:9.2.18-1.el7
 postgresql-server.x86_64 0:9.2.18-1.el7
Complete!
```

</div>
<br>

### c. Set Up the Ambari Server

---
<br>
<div class="indented-section">

On master node :

```
ambari-server setup
```

Respond to the setup prompt:

- If you have not temporarily disabled SELinux, you may get a warning. Accept the default (y), and continue.

- By default, Ambari Server runs under root. Accept the default (n) at the Customize user account for ambari-server daemon prompt, to proceed as root. If you want to create a different user to run the Ambari Server, or to assign a previously created user, select y at the Customize user account for ambari-server daemon prompt, then provide a user name.

- If you have not temporarily disabled iptables you may get a warning. Enter y to continue.

- Select a JDK version to download. Enter 1 to download Oracle JDK 1.8.

- Review the GPL license agreement when prompted. To explicitly enable Ambari to download and install LZO data compression libraries, you must answer y.

- Select n at Enter advanced database configuration to use the default, embedded PostgreSQL database for Ambari. The default PostgreSQL database name is ambari. The default user name and password are ambari/bigdata.

Our master node needs to be able to access the Hive MySql server. Run the following for that :

```
yum install mysql-connector-java*
ambari-server setup --jdbc-db=mysql --jdbc-driver=/usr/share/java/mysql-connector-java.jar
```

The path _/usr/share/java/mysql-connector-java.jar_ might change. Make sure it is indeed at this location

Restart ambari :

```
ambari-server restart
```

</div>
<br><br>

## 4. Open ports on your cloud provider (Azure, Google Cloud, AWS ...)

---
<div class="indented-section">

<br>

If you don't want to run into any port pronblem,
I recommend opening all ports **between the instances**, in other words opening all ports **only for your instances public IP addresses**.

In addition, you will need to open some ports publicly (i.e for all IP addresses) such as the ambari port (8080), or any other service's port that you might want to access inside your browser.

</div>

<br><br>

## 5. Deploy your cluster

---
<div class="indented-section">

<br>

Once your ports are opened, you can connect to ambari at the following address :

```
http://<your.server.public.ip.address>:8080
```

Username / Password : admin / admin

</div>

<br>

### a. Create the cluster

---
<br>
<div class="indented-section">

#### Step 1 : Launch the Ambari Cluster Install Wizard

> Launch the install wizard

#### Step 2 : Name Your Cluster

> Input a name for your cluster : don't use spaces or special characters

#### Step 3 : Select Version

> Leave the default version. <br/>
> Leave the repositeries matching your linux distribution and remove the others. CentOS repos are the redhat7 ones.
> <br/> Leave the other default options.

#### Step 4 : Install Options

> Input your hostnames. In our case it looks like this :

```
master.twitter-multi.com
slave-1.twitter-multi.com
slave-2.twitter-multi.com
```

> Copy/paste the private key generated in the master server. You can show the private key with these commands :

```
cd
cat .ssh/id_rsa
```

#### Step 5 : Confirm Hosts

> If everything goes well, you should see the following screen

![](2020-03-02-18-58-07.png)

#### Step 6 : Choose Services

> Clear the list of services

> Select the services you need. Only add services you will actually need because otherwise your servers will be overloaded and you might run into more configuration problems.

> Press next. **Ambari will ask you to accept the installation of some mandatory services and will suggest other ones.** Accept the mandatory ones and check if the suggested ones will be useful for your case or not. Unless you want extra security, I don't recommend installing RangerKMS since it requires additional configuration steps that are not detailed in this guide.

#### Step 7 : Assign Masters

> For the assign masters step, I recommend leaving the defaults unless you know what you're doing. Unfortunately, there wasn't a lot of documentation nor forums talk that helped me decide myself which services should be installed where so I just trusted ambari auto selection.

#### Step 8 : Assign Slaves and Clients

> For the assign slaves step, I recommend leaving the defaults unless you know what you're doing. Unfortunately, there wasn't a lot of documentation nor forums talk that helped me decide myself which services should be installed where so I just trusted ambari auto selection.

#### Step 9 : Customize Services

> Choose passwords for your services and note them down in case you might need them.

#### Step 10 : Review

> Review. You have the ability to save a blueprint of your server at this step. I never used a blueprint but I downloaded it in case something went wrong.
>
> **A blueprint is a file that stores your configuration and that can be reused to automate the cluster installation.** For more info on blueprints, please see the documentation :
> https://cwiki.apache.org/confluence/display/AMBARI/Blueprints

#### Step 11 : Install

> Install ! This step might take some time so go grab a coffee or netflix and chill in the meantime.

<br>

</div>

### b. Increase YARN capacity scheduler maximum resource percent

---
<br>
<div class="indented-section">

> Navigate to **Ambari Dashboard>YARN>Configs>Advanced>Scheduler**.
> Modify _yarn.scheduler.capacity.maximum-am-resource-percent_ value to something like 0.8~0.9 .
> Save and restart YARN service.

</div>

</body>
