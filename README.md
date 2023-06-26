# Configure a 4-node Kubernetes Cluster with Vagrant and virtualbox VM Provider

This guide provides instructions for setting up a 4-node Kubernetes cluster using k3s. It also includes guidelines for attaching and detaching virtual block devices.


## Pre-requisites

Before proceeding with the setup, ensure that you have the following:

- Ubuntu OS installed
- Basic familiarity with the command line

## Set up virtualbox, kubectl on Ubuntu


- ### Check the PATH Variable and Add Snap

  To ensure proper installation, verify the PATH variable and add Snap by executing the following commands:

  ```bash
  echo 'export PATH=$PATH:/snap/bin' >> ~/.bashrc
  ```

  ```bash
  source ~/.bashrc
  ```

- ### Install kubectl using Snap

  ```bash
  sudo snap install kubectl --classic
  ```

- ### Install VirtualBox
  
    ```bash
    sudo apt install virtualbox
    ```

## Check the Available IP Addresses on Your Machine and Modify the Vagrantfile Accordingly


### Install Vagrant

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
```

```bash
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
```

```bash
sudo apt update && sudo apt install vagrant
```

## Spin Up VMs Using Vagrant

Clone the Repository

Clone the repository containing the Vagrant configuration files:

```bash
git clone <repository_url>
```

Navigate to the vagrant directory:

```bash
cd vagrant
```

Start the virtual machines by running the following command:

```bash
vagrant up
```

Ensure that the master and the 4 nodes are in the running state.

```bash
vagrant global-status
```

Example Output:

```bash
Copy code
id       name   provider state   directory                                     
-------------------------------------------------------------------------------
6507133  master     virtualbox running /home/abdulrahman/vagrant                                          
7cb8bf7  node1      virtualbox running /home/abdulrahman/vagrant                                          
bad2ca5  node2      virtualbox running /home/abdulrahman/vagrant                                          
f236523  node3      virtualbox running /home/abdulrahman/vagrant                                          
90db6e7  node4      virtualbox running /home/abdulrahman/vagrant  
```

## Setup Kubernetes (K3s) on the Master and Nodes

- ### Log in to the Master Node

  To set up K3s on the master node, follow these steps:

  SSH into the Master Node SSH into the master node using the following command:

  ```bash
  vagrant ssh master
  ```

- ### Setup K3s on the Master Node

  Execute the command below to set up K3s on the master node:

  ```bash
  curl -sfL https://get.k3s.io | sh -
  ```

- ### Save the Kubeconfig file to access the Kubernetes cluster:

  ```bash
  cat /etc/rancher/k3s/k3s.yaml > k3s.yaml
  ```

  Modify k3s permissions to be able to access the cluster:

  ```bash
  sudo chmod +r /etc/rancher/k3s/k3s.yaml
  ```

- ### Copy the K3s token from the master node as it will be used later:

  ```bash
  cat /var/lib/rancher/k3s/server/node-token
  ```

- ### Make a Note of the Master's IP

  You can check the VagrantFile to get the IP address of the master node. In this example, the IP address is 


- ### Setup K3s on Worker Nodes (Execute on Each Worker Node)

  To set up K3s on each worker node, follow these steps:

  ```bash
  curl -sfL https://get.k3s.io | K3S_URL=https://<ip_of_the_master_node>:6443 K3S_TOKEN=<copied_token_from_master> sh -
  ```
  
  Replace the <ip_of_the_master_node> with the IP address of the master node and <copied_token_from_master> with the token copied from the master node.

- ### Add the Following Host Entries in /etc/hosts

  Ensure that the following host entries are present in the /etc/hosts file on each node:

  Run the following command to open the /etc/hosts file:

  ```bash
  sudo vim /etc/hosts
  ```

  Add the following host entries:

  ```bash
  Copy code
  192.168.56.1 master.k8s.com master
  192.168.56.2 node1.k8s.com node1
  192.168.56.3 node2.k8s.com node2
  192.168.56.4 node3.k8s.com node3
  192.168.56.5 node4.k8s.com node4
  ```

  Note: The master node can also update the host entries accordingly.

## Check the Cluster's Health

Configure kubectl in your client/host machine to point to the cluster:

```bash
export KUBECONFIG=k3s.yaml
```

Check the Status of the Nodes

To verify the status of the nodes, run the following command:

```bash
kubectl get nodes
```

This command will display the status of the nodes in the Kubernetes cluster.


## Configure LVs on the Nodes:

```bash
sudo truncate --size=1G /tmp/disk-{1..4}.img        
for disk in /tmp/disk-{1..4}.img; do sudo losetup --find $disk; done
devices=( $(for disk in /tmp/disk-{1..4}.img; do sudo losetup --noheadings --output NAME --associated $disk; done) )
sudo pvcreate "${devices[@]}"
vgname="vg0"
sudo vgcreate "$vgname" "${devices[@]}"
for lvname in lv-{0..3}; do sudo lvcreate --name="$lvname" --size=800MiB "$vgname"; done
```

These commands create virtual disk images, associate them with loop devices, initialize physical volumes, create a volume group, and create logical volumes within the volume group.