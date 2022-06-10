# Deploying MicroShift on an OSTree based system
by Diego Alvarez

<img src="https://github.com/dialvare/MicroShift-OSTreeSystems-blog/blob/main/MicroShift%20architecture.png" width="800" height="380">

As edge computing evolves and increases every time, new requirements and capabilities are needed in order to deploy and manage workloads on the edge. MicroShift arises from the necessity of having a solution capable of providing the same experience that we could have by running OpenShift/Kubernetes on traditional infrastructures while decreasing the resource footprint. Therefore, MicroShift allows deploying Openshift solutions at scale for field-deployed edge computing devices.

To ensure basic security standards, edge computing should run on optimized operating systems. OSTree based systems provide immutability and are based on transactions for rollbacks and upgrades. In addition to providing a safe environment, these systems also allow avoiding workload disruptions. These capabilities make OSTree based systems the ideal environment to run MicroShift on. 

There are currently several Operating Systems which fall under this category (OSTree based systems). Among the most known options we can find OS such as Fedora IoT or RHEL for Edge. In this blog, we'll be focusing on how to deploy MicroShift on Fedora IoT.

## Installing Fedora IoT
The first step will be installing the chosen Operating System in our machine. We can download the Fedora IoT image from the [official releases page](https://getfedora.org/en/iot/download/). In my case, I'm going to select the **Fedora 36: Installer ISO for x86_64** (the latest version at the time I'm writing this blogpost).

Once the installer is downloaded we need to set up the machine to reboot and initialize from that ISO image. Right after restarting the machine, we'll see the next screen. Select **Install Fedora-IoT 36**:

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

When finished, click on the **Begin Installation** button. Once completed, reboot the machine. If completed correctly, our node should be running Fedora IoT as its operating system. 

For the next steps, we can use *ssh* to connect to the machine. Log in as *root* (we'll need full privileges to deploy MicroShift) and enter the *password* provided during the installation:

````
ssh -p 3022 root@127.0.0.1
````

## Deploying MicroShift
Now, we're ready to proceed with the MicroShift installation. We're going to embed MicroShift on the Operating System to be part of the base *rpm-ostree*. Let's start by including all the required repositories to embed MicroShift in our Fedora IoT system:

````
curl -L -o /etc/yum.repos.d/fedora-modular.repo https://src.fedoraproject.org/rpms/fedora-repos/raw/rawhide/f/fedora-modular.repo
curl -L -o /etc/yum.repos.d/fedora-updates-modular.repo https://src.fedoraproject.org/rpms/fedora-repos/raw/rawhide/f/fedora-updates-modular.repo
curl -L -o /etc/yum.repos.d/group_redhat-et-microshift-fedora-36.repo https://copr.fedorainfracloud.org/coprs/g/redhat-et/microshift/repo/fedora-36/group_redhat-et-microshift-fedora-36.repo
````

Then, we can enable the *crio-o* module. This step allows us to select the packages to be installed in our base *rpm-ostree*:

````
rpm-ostree ex module enable cri-o:1.21
````

It's important to note that when trying to install some dependencies in an *rpm-ostree*, we must perform an update to ensure that all newer versions are installed. In OSTree based systems, the base layer is an atomic entity, so when installing a local package, older dependency versions will not be updated. Keeping this in mind, we can continue installing the packages and dependencies for MicroShift. When done, reboot the node:

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

To further secure our deployment, we're going to configure the system firewall to only allow connections through the ports that MicroShift needs to use. The ones to be considered are listed below:

| Port | Protocol | Description |
|---|---|---|
| 80 | TCP | HTTP port to serve applications |
| 443 | TCP | HTTPS port to serve applications |
| 5353 | UDP | Port to expose the mDNS service |

More networking information regarding firewall connections can be found in the *Firewall section* of [MicroShift documentation](https://microshift.io/docs/user-documentation/networking/firewall/).

We should configure the firewall by enabling connections through the ports previously stated. Additionally, we will add the PodIP range (*10.42.0.0/16 in my case*) to allow the pods to contact the internal coreDNS server:

````
sudo firewall-cmd --zone=trusted --add-source=10.42.0.0/16 --permanent
sudo firewall-cmd --zone=public --add-port=80/tcp --permanent
sudo firewall-cmd --zone=public --add-port=443/tcp --permanent
sudo firewall-cmd --zone=public --add-port=5353/udp --permanent
sudo firewall-cmd --reload
````

At this point, we can finally enable MicroShift by running the following command:

````
sudo systemctl enable microshift --now
````

Next, we'll install the OpenShift client to access the cluster:

````
curl -O https://mirror.openshift.com/pub/openshift-v4/$(uname -m)/clients/ocp/stable/openshift-client-linux.tar.gz
sudo tar -xf openshift-client-linux.tar.gz -C /usr/local/bin oc kubectl
````

Finally, let's move the *kubeconfig* file to the default location to avoid scaling permissions. Note that we don't need to install *podman* because it's already part of the *rpm-ostree* entity:

```
mkdir ~/.kube
sudo podman cp microshift:/var/lib/microshift/resources/kubeadmin/kubeconfig ~/.kube/config
sudo chown `whoami`: ~/.kube/config
```

That's all! Now we have MicroShift deployed and running on an OSTree based system! With it, you can run typical *oc* commands, as shown in the example:

````
oc get pods -A

NAMESPACE                       NAME                                  READY   STATUS    RESTARTS   AGE
kube-system                     kube-flannel-ds-4g8mq                 1/1     Running   1          16h
kubevirt-hostpath-provisioner   kubevirt-hostpath-provisioner-gmc7l   1/1     Running   1          16h
openshift-dns                   dns-default-kz7g7                     2/2     Running   2          16h
openshift-dns                   node-resolver-9wvnl                   1/1     Running   1          16h
openshift-ingress               router-default-6c96f6bc66-5dckf       1/1     Running   1          16h
openshift-service-ca            service-ca-7bffb6f6bf-w584c           1/1     Running   1          16h
````

## Sample application 
Before finishing this blog post, and with the objective of validating the installation, we're going to deploy a basic application. For this example, we can deploy a **Metal Load Balancer** to manage and route traffic. 

Firstly, create the *namespace* and *deployment* for the load balancer:

````
oc apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml
oc apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml
````

Then, we need to create a *ConfigMap* to define the address pool that the LB will be using:

````
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
````

Deploy an *nginx* test application to ensure that everything was well configured and is working as expected:

````
oc create ns test
oc create deployment nginx -n test --image nginx
````

Create the service, by generating the following file:

````
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
````

Verify the *External-IP* automatically assigned to our service. It might take some time, so we can add the *watch* statement to refresh the command every 2 seconds:

````
watch oc get svc -n test

NAME    TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
nginx   LoadBalancer   10.43.228.39   192.168.1.240   80:30643/TCP   8s
````

At this point, we should reach the *nginx* application through that External-IP. To do so, we're going to install *Firefox* web browser in our *rpm-ostree*, as well as some other packages to allow the usage of GUIs. Then, reboot the machine:

````
rpm-ostree upgrade

rpm-ostree install xauth xhost
rpm-ostree install firefox

systemctl reboot
````

Once the machine is up and running again, we can *ssh* into it, but this time, using the *-Y* command to take advantage of the graphical window:

````
ssh -Y -p 3022 root@127.0.0.1
````

To conclude, run the following command to access the *nginx* application using the web browser:

````
firefox http://192.168.1.240
````

A new window will pop up and you'll see the page shown below:

<img src="https://github.com/dialvare/MicroShift-OSTreeSystems-blog/blob/main/Nginx.png" width="660" height="270">

## What's next?
Now you know how to embed and configure MicroShift in an OSTree based system, by previously installing the Fedora IoT Operating System. In addition, we have validated the installation by deploying and running a sample application.

For further information, please refer to the [MicroShift project site](https://microshift.io/docs/home/), where you'll find all the documentation, release notes and demos. You can also check the source code on this [GitHub repository](https://github.com/openshift/microshift).

Do you want to [get involved](https://microshift.io/docs/community/) in our community? Join our Slack channel or attend one of our bi-weekly calls!
