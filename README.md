# pi-kube
Setup K8 on raspberry
Note: you need to be familiar with Linux/Debian and Raspberry.

Get a copy of https://www.raspberrypi.org/downloads/raspbian/
Flash the a 64Gb min SD Card using fletcher or other image tool

1.- Configure you wifi (I recommend Ethernet if you can)
  Use ifconfig, to get your current ip
  Copy your IP

  Below will enable a static IP so you can easy SSH
  sudo nano /etc/dhcpcd.conf and enter the following data
    interface wlan0
    static ip_address=192.168.XX.XXX/24   (XX.XX is the IP you copied in the previous point)
    static routers=192.168.X.X  (this is your router IP gateway)
    static domain_name_servers=8.8.8.8  (setup this to you prefered DNS or Google DNS)

   Below you can setup your WiFI SSID
   sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
    network={
        ssid="your SSID"
        psk="your WiFi Password"
    }
  
  Reboot
  To check if it's working
    ifconfig
    iwconfig
    or ping google.com

2.- Enable SSH on Pi
  sudo raspi-config
  
3.- Setup Docker ,  command installs docker and sets the right permission.
  curl -sSL get.docker.com | sh && \
  sudo usermod pi -aG docker && \
  newgrp docker 

4.- Disable swap - it's mandatory for Kubernetes to work on Pi
  sudo dphys-swapfile swapoff
  sudo dphys-swapfile uninstall
  sudo update-rc.d dphys-swapfile remove
  sudo apt purge dphys-swapfile

5.- Now edit the /boot/cmdline.txt file. Add the following in the end of the file, should be just one line
   cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
  
6.- reboot: sudo reboot

7.- Edit the following file /etc/apt/sources.list.d/kubernetes.list and add
    deb http://apt.kubernetes.io/ kubernetes-xenial main

8.- Run this command to add the key
    sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    if that fails , try step 4 again
    if it works, output will show OK
 
9.- Update system
    sudo apt-get update
    
10.- Install kubeadm and kubectl - the brain
    sudo apt-get install -qy kubeadm
    
NOW lets setup the master node
  - sudo kubeadm config images pull -v3
  - sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.X.X (XX your ip) ( we will use Flannel, you can use any other flannel, just check Kube official documentation)
  Note: this should not be done in production , as the token will never expire
  
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

11.- Apply these changes for DNS to work
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

12.- Apply network driver
   kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
    
12.- On the master and all the workers run the following command
    sudo sysctl net.bridge.bridge-nf-call-iptables=1

13.- Run this command - note it will take some time to initialize the flannel-pod and core-DNS
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

    


    


    


