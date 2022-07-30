# microk8s
Instructions for creating a microk8s cluster on bare-metal

## Install Ubuntu Server On Each Node

## Update hosts files to include Node IP addresses

- ubuntu1 
  ```
  sudo nano /etc/hosts
  ```
    - 10.0.0.187 ubuntu2
    - 10.0.0.163 ubuntulaptop

- ubuntu2 
  ```
  sudo nano /etc/hosts
  ```
    - 10.0.0.86 ubuntu1
    - 10.0.0.163 ubuntulaptop

- ubuntulaptop 
  ```
  sudo nano /etc/hosts
  ```
    - 10.0.0.86 ubuntu1
    - 10.0.0.187 ubuntu2
 
## Mayastor Prep On Each Node
Mayastor is a k8s distributed storage class

- Enable huge pages on each node:
     
    ```
    sudo echo vm.nr_hugepages = 1024 | sudo tee -a /etc/sysctl.d/20-microk8s-hugepages.conf
    ```

- Enable Kernel module 'nvme_tcp'
     
    ```
    sudo apt-get install linux-modules-extra-$(uname -r)
    ```
     
    ```
    sudo modprobe nvme-tcp
    ```
     
    ```
    echo 'nvme-tcp' | sudo tee -a /etc/modules-load.d/microk8s-mayastor.conf
    ```
     
    ```
    sudo reboot
    ```
    
## Laptop Close Lid
Need to change laptop log in configuration file so that when the lid is closed the laptop stays active

- Edit /etc/systemd/logind.conf    
    ```
    sudo nano /etc/systemd/logind.conf
    ```
- Find and uncomment the key HandleLidSwitch and set key-value to ignore:  HandleLidSwitch=ignore
    ```
    sudo service systemd-logind restart
    ```

## For <span style="color:blue">Each</span> Node With GPU

- Install NVIDIA Drivers
    ```
    sudo apt install nvidia-driver-515-server nvidia-utils-515-server
    ```
    ```
    sudo reboot
    ```
- Containerd Setup
    - Follow to the letter: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#containerd

- NVIDIA Container Runtime
    ```
    sudo apt-get install nvidia-container-runtime
    ```

## Install microk8s
    
```
sudo snap install microk8s --classic
```
- Configure Containerd
    ```
    echo '
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
            BinaryName = "/usr/bin/nvidia-container-runtime"
    ' | sudo tee -a /var/snap/microk8s/current/args/containerd-template.toml
    ```
    ```
    sudo snap restart microk8s.daemon-containerd
    ```
    
    
## For <span style="color:blue">Each</span> Node With GPU
This ensures all drivers and settings are correct for each node before setting up the cluster
   
- Helm Install GPU Operator
    ```
    sudo microk8s helm3 repo add nvidia https://nvidia.github.io/gpu-operator
    ```
    ```
    sudo microk8s helm3 install gpu-operator nvidia/gpu-operator \
    --create-namespace -n gpu-operator-resources \
    --set driver.enabled=false,toolkit.enabled=false
    ```
    ```
    watch -n 1 'microk8s kubectl get pods --all-namespaces'
    ```

- When all the pods are up and running without failures, reset Microk8s
    ```
    sudo microk8s reset
    ```

## Set Up the Cluster

- From a Single Node Add Each Node
  ```
  microk8s add-node
  ```
- Enable Micrkok8s add-ons
    ```
    microk8s enable dns
    ```
    ```
    microk8s enable helm3
    ```

- Helm Install GPU Operator
  Note: Don't use microk8s enable gpu use Helm3
    ```
    sudo microk8s helm3 repo add nvidia https://nvidia.github.io/gpu-operator
    ```
    ```
    sudo microk8s helm3 install gpu-operator nvidia/gpu-operator \
    --create-namespace -n gpu-operator-resources \
    --set driver.enabled=false,toolkit.enabled=false
    ```
- If Everything Green-lights  
  ```
  microk8s enable ingress
  ```
  ```
  microk8s enable metallb with 10.0.0.50-10.0.0.100
  ```
  ```
  microk8s enable mayastor
  ```
