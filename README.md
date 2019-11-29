# Install Kubernetes on Raspberry Pi3B+

# Master Node Setup
Setup K8 on raspberry
Note: you need to be familiar with Linux/Debian and Raspberry.

Get a copy of https://www.raspberrypi.org/downloads/raspbian/
Flash the a 64Gb min SD Card using fletcher or other image tool

As a reminder first time you enter into the raspberry the user is "pi" and the password "raspberry"
I highly recommend once in the shell, enter "passwd" to change your password.

--> Configure you wifi (I recommend Ethernet if you can)
   Option 1:
   sudo raspi-config
   Go to Network options
   Wi-fi option
   and enter your wifi details
   
   Option 2 add this to wpa_supplicant.conf file:
      country = XX (your country code, ie US)
      sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
      network={
          ssid="your SSID"
          psk="your WiFi Password"
        }

   then: sudo reboot
  
  Use ifconfig, to get your current ip
  Copy your IP , or you can setup any IP you want, just make sure is free on your router

  Below will enable a static IP so you can easy SSH
  sudo nano /etc/dhcpcd.conf and enter the following data
    interface wlan0
    static ip_address=192.168.XX.XXX/24   (XX.XX is the IP you copied in the previous point)
    static routers=192.168.X.X  (this is your router IP gateway)
    static domain_name_servers=8.8.8.8  (setup this to you prefered DNS or Google DNS)
 
  
  Use the raspi-config utility to change the hostname to k8s-master-1 or similar and then reboot (Network Options -> Hostnames).
  
  Apply these changes for DNS to work
      sudo nano /etc/hostnames
      - should be blank

      sudo nano /etc/hosts
      Add your IP to you cluster host name, by default will be showing 127.0.0.1
      192.168.56.x    XXX-node
      
      sudo rm -f /etc/resolv.conf  # Delete the symbolic link
      sudo nano /etc/resolv.conf   # Create static file
          # Content of static resolv.conf
          nameserver 8.8.4.4
          nameserver 8.8.8.8
      
     sudo reboot
     After reboot., check : sudo nano /etc/resolv.conf and make sure is showing google DNS

     Note if you are setting up this in ubuntu, this extra steps are required
     sudo nano /etc/NetworkManager/NetworkManager.conf
        [main]
        plugins=ifupdown,keyfile,ofono
        #dns=dnsmasq ==> MASK
     sudo service network-manager restart
  
  Reboot
  To check if it's working
    ifconfig
    iwconfig
    or ping google.com

--> Enable SSH on Pi
  sudo raspi-config - interfacion options - SSH
  
--> Setup Docker ,  command installs docker and sets the right permission.
  curl -sSL get.docker.com | sh && \
  sudo usermod pi -aG docker && \
  newgrp docker 

--> Disable swap - it's mandatory for Kubernetes to work on Pi
  sudo dphys-swapfile swapoff && \
  sudo dphys-swapfile uninstall && \
  sudo update-rc.d dphys-swapfile remove &&\
  sudo apt purge dphys-swapfile

--> Add cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory to sudo nano /boot/cmdline.txt, keep only one line
    and sudo reboot


--> run this to setup Kubernetes 
     curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - 
     
     sudo nano /etc/apt/sources.list.d/kubernetes.list
     Add this to the file: deb http://apt.kubernetes.io/ kubernetes-xenial main
     
     Then run 
     sudo apt-get update -q && \
     sudo apt-get install -qy kubeadm
    
-->  NOW lets setup the master node
   sudo kubeadm config images pull -v3 (this will take some minutes)
  (ONLY FOR MASTER - for nodes read below)
     sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.X.X (XX your Master ip)
  
  Note we will use Flannel, you can use any other but please check Kube official documentation)  
  Users of weave-wrok had reported issues with ARM
  
 When this complete , check at the instructions , you need to run this to enable the master, see below

    Your Kubernetes control-plane has initialized successfully!
    To start using your cluster, you need to run the following as a regular user:
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      https://kubernetes.io/docs/concepts/cluster-administration/addons/

    Then you can join any number of worker nodes by running the following on each as root:
    kubeadm join 192.168.0.40:6443 --token cq3wji.jc8oe98m0lp4wyba \--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxx

Important Note: This last statement, you need to copy it to join workers nodes to the master
    if by any chance you forgot it, run this command on the master: kubeadm token list
    If you need to create a new token use: kubeadm token create --print-join-command


--> Apply network driver
   kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
    
--> On the master and all the workers run the following command
    sudo sysctl net.bridge.bridge-nf-call-iptables=1

--> Run this command - note it will take some time to initialize the flannel-pod and core-DNS
    kubectl get pods --all-namespaces
    Everything should be running
   
          NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE
      kube-system   coredns-5644d7b6d9-8smqz               1/1     Running   0          9m4s
      kube-system   coredns-5644d7b6d9-knrj6               1/1     Running   0          9m4s
      kube-system   etcd-k8s-master-1                      1/1     Running   0          8m46s
      kube-system   kube-apiserver-k8s-master-1            1/1     Running   0          8m35s
      kube-system   kube-controller-manager-k8s-master-1   1/1     Running   0          8m39s
      kube-system   kube-flannel-ds-arm-w252m              1/1     Running   0          6m49s
      kube-system   kube-proxy-mpwgf                       1/1     Running   0          9m4s
      kube-system   kube-scheduler-k8s-master-1            1/1     Running   0          8m48s

  # Worker Node Setup
  1.- Adding Worker Nodes
   For the worker nodes on others raspberries, repeat all steps above except the kubeadm init command, for the workers use the join command
   
    Remember to Change hostname
      Use the raspi-config utility to change the hostname to k8s-worker-1 or similar and then reboot.
    
    Join the cluster
    Replace the token / IP for the output you got from the master node, for example:
      Remember this is coming from step 10
      sudo kubeadm join --token <token> <master-node-ip>:6443 --discovery-token-ca-cert-hash sha256:<sha256>
  
  remmber to apply the network driver after join
  kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
  
      Important Note: This last statement, you need to copy it to join workers nodes to the master, run this on the master
        if by any chance you forgot it, run this command on the master: kubeadm token list
        If you need to create a new token use: kubeadm token create --print-join-command
  
  
  

Usefule commands
Reset kube clkuster: kubeadmin reset

Delete docker
apt remove docker -y
sudo dpkg --purge docker-ce
    


    


