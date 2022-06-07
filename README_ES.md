# Deploying MicroShift on an OSTree based system
by Diego Alvarez

As edge computing evolves and increases every time, new requirements and capabilities are needed in order to deploy and manage workloads on the edge. Microshift arises from the necessity of having a solution capable of providing the same experience that we could have by running OpenShift/Kubernetes on traditional infrastructures while decreasing the resource footprint. Therefore, MicroShift allows deploying Openshift solutions at scale for field-deployed edge computing devices.

To ensure basic security standards, edge computing should run on optimized operating systems. OSTree based systems provide immutability and are based in transactions for rollbacks and upgrades. In addtion to providing a safe environtment, these systems also allow avoiding workload disruptions. These capabilities make OSTree based systems the ideal environment to run MicroShift on. 

There are currently several Operating Systems which fall under this category (OSTree based systems). Among the most known options we can find OS such as Fedora IoT or RHEL for Edge. In this blog, we'll be focusing on how to deploy MicroShift on Fedora IoT.

## Installing Fedora IoT
The first steep will be installing the choosen Operating System in our machine. We can download the Fedora IoT image from the [official releases page](https://getfedora.org/en/iot/download/). In my case, I'm going to select the **Fedora 36: Installer ISO for x86_64** (the latest version at the time i'm writting this blogpost).

Once downloaded the installer, we need to set up the machine to reboot and initialize from that ISO image. Right after restarting the machine, we'll see the next screen. Select **Install Fedora-IoT 36**:

<img src="https://github.com/dialvare/MicroShift-OSTreeSystems-blog/blob/main/Fedora%20IoT%20Installer.png" width="500" height="250">

Then, a new window with the Fedora IoT graphical interface will appear. You'll need to choose the preferred **Language** and **Keyboard Layout**. After that, we'll jump to the *Installation Summary* section:

<img src="https://github.com/dialvare/MicroShift-OSTreeSystems-blog/blob/main/Installation%20Summary.png" width="700" height="250">

As shown in the image, some parts will be marked with a warning icon, meaning that we'll need to check and configure those sections before proceeding with the installation. Click on the following sections and configure:
- **Installation Destination** section:
  - Verify that the *Disk device* is selected.
- **Network & Host Name** section:
  - Enable *Ethernet connection*.
  - Give your preferred *hostname*.
- **Root Account** section:
  - Select *Enable root account*.
  - Set the *Root Password*.
  - Select *Allow root SSH login with password*.
- **User Creation** section:
  - Enter the *Full name* for the new user.
  - Change the *User name* if needed.
  - Set the *Password*.

When finished, click on the **Begin Installation** button. Once completed, reboot the machine. Our node is currently running Fedora IoT as its operating system. We can use *SSH* to connect to the machine. Log in as *root* (we'll need full privilegies to deploy MicroShift) and enter the *password* provided during the installation:

````
ssh -p 3022 root@127.0.0.1
````

## Deploying MicroShift
Now, we're ready to proceed with the MicroShift installation. We're going to embed MicroShift as part of the base Fedora IoT *rpm-ostree*. Let's start by including all needed repositories to embed MicroShift in our Fedora IoT system:

````
curl -L -o /etc/yum.repos.d/fedora-modular.repo https://src.fedoraproject.org/rpms/fedora-repos/raw/rawhide/f/fedora-modular.repo
curl -L -o /etc/yum.repos.d/fedora-updates-modular.repo https://src.fedoraproject.org/rpms/fedora-repos/raw/rawhide/f/fedora-updates-modular.repo
curl -L -o /etc/yum.repos.d/group_redhat-et-microshift-fedora-36.repo https://copr.fedorainfracloud.org/coprs/g/redhat-et/microshift/repo/fedora-36/group_redhat-et-microshift-fedora-36.repo
````

Then, we can enable the *crio-o* module. This step allows us to select the needed packages to be installed in our base *rpm-ostree*:

````
rpm-ostree ex module enable cri-o:1.21
````

It's important to keep in mind that, when trying to install some dependencies in an *rpm-ostree*, we must perform an update to ensure that all newer verisons are installed. In OSTree based systems, the base layer is an atomic entity, so when installing a local package, older dependency versions will not be updated. Once this is known, we can continue installing the packages and dependeces for MicroShift. Then reboot the node:

```
rpm-ostree upgrade
rpm-ostree install cri-o cri-tools microshift
systemctl reboot
```

Let's continue by obtaining the MicroShift service. Run the following command:

````
sudo curl -o /etc/systemd/system/microshift.service \
https://raw.githubusercontent.com/redhat-et/microshift/main/packaging/systemd/microshift-containerized.service
````

To provide more security in our deployment, we're going configure a firewall to allow only the ports and connections that MicroShift needs to usee. Below are listed the ones to be considered:

| Port | Protocol | Description |
|---|---|---|
| 80 | TCP | HTTP port to serve applicactions |
| 443 | TCP | HTTPS port to serve applications |
| 5353 | UDP | Port to expose the mDNS service |




## Sample application 

## What's next?
