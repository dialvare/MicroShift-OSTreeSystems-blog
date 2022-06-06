# Deploy MicroShift on an OSTree based system
by Diego Alvarez

MicroShift is a research project which aims to reduce Kubernetes and OKD (the Kubernetes distribution by the OpenShift community) footprints, in order to be deployed on edge computing devices. 

## Install an OSTree based system
According to security standards, an edge device deployed in the field should run an operating system optimizd to be inmutable, and based in transactions for rollbacks and upgrades. Those transactions should be done in a securely and safely way, in addition to be speedly, but always avoiding workloads disrruptions. All those requirements metioned before could be addressed by running MicroShift in a OSTree based system.

We're going to install and run MicroShift in the machine with an OSTree based as its operating system. One of the main systems used for Edge computing, based in OSTree is Fedora IoT. We can obtain the Installer ISO image from the [official page](https://getfedora.org/en/iot/download/). In this case, I'll be installing the image in a virtual machine. To run MicroShift the minimum requirements will be:
- 2048 MB of RAM
- 2 CPU cores
- 1 GB of free storage space

Once downloaded the image, we neet to set up the machine to reboot and initialize from that image. Right after starting the machine, we'll see the Installer menu. We're going to select **Install Fedora-IoT 36**:

<img src="https://github.com/dialvare/MicroShift-OSTreeSystems-blog/blob/main/Fedora%20IoT%20Installer.png" width="500" height="250">

The Fedora-IOT graphical interface we'll apper. We're going to need to select the **Language** and the **Keyboard layout**. After that, we'll junp to the *Installation Summary* section: 

<img src="https://github.com/dialvare/MicroShift-OSTreeSystems-blog/blob/main/Installation%20Summary.png" width="700" height="250">

As shown in the image, some parts will be marked with a warning icon, meaning that we'll need to check and configure those sections before proceeding with the installation. Click on the following sections and configure:
- **Installation Destination** section:
  - Verify that the Disk device is selected
- **Network & Host Name** section:
  - Enable Ethernet connection
  - Give your preferred hostname
- **Root Account** section:
  - Select *Enable root account*
  - Set the *Root Password*
  - Select *Allow root SSH login with password*

When finished, click on the **Begin Installation** button. Once the installation is complete, reboot the machine and log in as *root* user and the *password* provided during the installation.

## Deploy MicroShift
Our node is currently running Fedora IoT as operating system. The next step will be the MicroShift deployment. In order to connect to the node, we can use SSH into the virtual machine:
```
ssh -p 3022 root@127.0.0.1
```
Excelent! We have succesfully connected. Now, we're ready to proceed with the MicroShift installation. 

Let's start by including all needed repositories to embed MicroShift in our Fedora IoT system:
```
curl -L -o /etc/yum.repos.d/fedora-modular.repo https://src.fedoraproject.org/rpms/fedora-repos/raw/rawhide/f/fedora-modular.repo
curl -L -o /etc/yum.repos.d/fedora-updates-modular.repo https://src.fedoraproject.org/rpms/fedora-repos/raw/rawhide/f/fedora-updates-modular.repo
curl -L -o /etc/yum.repos.d/group_redhat-et-microshift-fedora-36.repo https://copr.fedorainfracloud.org/coprs/g/redhat-et/microshift/repo/fedora-36/group_redhat-et-microshift-fedora-36.repo
```

Then, we can enable the crio-o module, to pick some packages to install in our base *rpm-ostree*:
```
rpm-ostree ex module enable cri-o:1.21
```

Before continue, it's importatant to know that when using an *rpm-ostree*, we'll need to perform an upgrade before installing packages. This is because, the base layer is an atomic entity, so when we need to install a package that is already part of the *rpm-ostree*, this won't be updated. 
Once we mentioned that, we can install the packages and dependeces for MicroShift. Then reboot the node:
```
rpm-ostree upgrade
rpm-ostree install cri-o cri-tools microshift

systemctl reboot
```

Now, we can deploy the MicroShift service:
```
sudo curl -o /etc/systemd/system/microshift.service \
https://raw.githubusercontent.com/redhat-et/microshift/main/packaging/systemd/microshift-containerized.service
```

MicroShift nRegardnig security requeriments, we're going to enable the firewall to allow only the ports that MicroShift is going to use:
| Port | Protocol | Description |
|---|---|---|
| 80 | TCP | HTTP port to serve applicactions |
| 443 | TCP | HTTPS port to serve applications |
| 5353 | UDP | Port to expose the mDNS service |

Enable the firewall and add the ports mentioned above. Additionally, add the PodIP range (*10.42.0.0/16 in my case*) to allow the pods to contact the internal coreDNS server:
```
sudo firewall-cmd --zone=trusted --add-source=10.42.0.0/16 --permanent
sudo firewall-cmd --zone=public --add-port=80/tcp --permanent
sudo firewall-cmd --zone=public --add-port=443/tcp --permanent
sudo firewall-cmd --zone=public --add-port=5353/udp --permanent
sudo firewall-cmd --reload
```

Once configured, enable MicroShift:
```
sudo systemctl enable microshift --now
```

Install the OpenShift client to access the cluster:
```
curl -O https://mirror.openshift.com/pub/openshift-v4/$(uname -m)/clients/ocp/stable/openshift-client-linux.tar.gz
sudo tar -xf openshift-client-linux.tar.gz -C /usr/local/bin oc kubectl
```

Let's move the *kubeconfig* file to the default location. Note that we don't need to install Podman because it's already part of the *rpm-ostree* entity:
```
mkdir ~/.kube
sudo podman cp microshift:/var/lib/microshift/resources/kubeadmin/kubeconfig ~/.kube/config
sudo chown `whoami`: ~/.kube/config
```

Good news: MicroShift is now installed and running, and we can use *oc* commands to verify that MicroShift is working:
```
oc get pods -A

NAMESPACE                       NAME                                  READY   STATUS    RESTARTS   AGE
kube-system                     kube-flannel-ds-4g8mq                 1/1     Running   1          16h
kubevirt-hostpath-provisioner   kubevirt-hostpath-provisioner-gmc7l   1/1     Running   1          16h
openshift-dns                   dns-default-kz7g7                     2/2     Running   2          16h
openshift-dns                   node-resolver-9wvnl                   1/1     Running   1          16h
openshift-ingress               router-default-6c96f6bc66-5dckf       1/1     Running   1          16h
openshift-service-ca            service-ca-7bffb6f6bf-w584c           1/1     Running   1          16h
```

To end this blog, we're going to test it with a sample application. Let's deploy a Metal LB to route traffic.

Firstly, create a new namespace and the deployment:
```
oc apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml
oc apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml
```

Once the components are available, we need to createa a *ConfigMap* to define the address pool taht the LB will be using:
```
oc create -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.240-192.168.1.250    
EOF
```

To see if everithing is working as expected, we are going to create a test applicaton:
```
oc create ns test
oc create deployment nginx -n test --image nginx
```

Then, deploy a service:
```
oc create -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: test
  annotations:
    metallb.universe.tf/address-pool: default
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
EOF
```

Check if an external IP has been assigned to the service:
```
oc get svc -n test

NAME    TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
nginx   LoadBalancer   10.43.228.39   192.168.1.240   80:30643/TCP   8s
```

Finally, if everything is configured correctly (it should be at this point), it will be possible to access the application using a browser. Before installing the browser, we need to allow the machine to show the graphical interface in a window. Then, we can install our preferred browser. In my case, I'll use Firefox:
```
rpm-ostree upgrade

rpm-ostree install xauth xhost
rpm-ostree install firefox

systemctl reboot
```

Connect again to the machine, but this time, adding the *-Y* command to allow graphical windows:
```
ssh -Y -p 3022 root@127.0.0.1
```

Now, access the NGINX application throught the External IP in the Firefox browser:
```
firefox http://192.168.1.240
```

A new window will pop up and you'll see the page shown below:

<img src="https://github.com/dialvare/MicroShift-OSTreeSystems-blog/blob/main/Nginx.png" width="660" height="270">

