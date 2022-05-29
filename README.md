# Deploy MicroShift on a OSTree based system
MicroShift is a research project which aims to reduce Kubernetes and OKD (the Kubernetes distribution by the OpenShift community) footprints, in order to be deployed on edge computing devices. 

## OSTree based systems
According to security standards, an edge device deployed in the field should run an operating system optimizd to be inmutable, and based in transactions for rollbacks and upgrades. Those transactions should be done in a securely and safely way, in addition to be speedly, but always avoiding workloads disrruptions. All those requirements metioned before could be addressed by running MicroShift in a OSTree based system.

## Deployment
We're going to install and run MicroShift in the machine with an OSTree based as its operating system. One of the main systems used for Edge computing, based in OSTree is Fedora IoT. We can obtain the Installer ISO image from the [official page](https://getfedora.org/en/iot/download/). In this case, I'll be installing the image in a virtual machine. To run MicroShift the minimum requirements will be:
- 2048 MB of RAM
- 2 CPU cores
- 1 GB of free storage space

Once downloaded the image, we neet to set up the machine to reboot and initialize from that image. Right after starting the machine, we'll see the Installer menu. We're going to select **Install Fedora-IoT 36**:

<img src="https://github.com/dialvare/MicroShift-OSTreeSystems-blog/blob/main/Fedora%20IoT%20Installer.png" width="500" height="250">

The Fedora-IOT graphical interface we'll apper. We're going to need to select the **Language** and the **Keyboard layout**. After that, we'll junp to the *Installation Summary* section. As shown in the image, some parts will be marked with a warning icon, meaning that we'll need to check and configure those sections before proceeding with the installation:

<img src="https://github.com/dialvare/MicroShift-OSTreeSystems-blog/blob/main/Installation%20Summary.png" width="700" height="250">





