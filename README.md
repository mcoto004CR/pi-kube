# Install Kubernetes on Raspberry Pi3B+

# Master Node Setup
Setup K8 on raspberry
Note: you need to be familiar with Linux/Debian and Raspberry.

Get a copy Raspbian Lite of https://www.raspberrypi.org/downloads/raspbian/
Flash the a 64Gb min SD Card using fletcher or other image tool

As a reminder first time you enter into the raspberry the user is "pi" and the password "raspberry"
I highly recommend once in the shell, enter "passwd" to change your password.

--> Configure you wifi (I highly recommend Ethernet if you can)
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
    interface wlan0 (Note if you are using ethernet as I recommended, use eth0)
    static ip_address=192.168.XX.XXX/24   (XX.XX is the IP you copied in the previous point)
    static routers=192.168.X.X  (this is your router IP gateway)
    static domain_name_servers=8.8.8.8  (setup this to you prefered DNS or Google DNS)
 
  
  Change Hostname
  
      sudo raspi-config
      Go to (Network Options -> Hostnames) to change the hostname to k8s-master-1 or similar , all cluster must have a diferent hostname
      sudo reboot 
  
  After reboot , use ifconfig to make sure you get the static IP correctly
  
  --> Enable SSH on Pi
  
      sudo raspi-config - interfacion options - SSH
  
  Apply these changes for DNS to work
         
         sudo nano /etc/hostnames
         should be blank

      sudo nano /etc/hosts
      Add your IP to you cluster host name, by default will be showing 127.0.0.1
      192.168.56.x    XXX-node
  
      NOTE, check if nameserver is correct
      
      sudo nano /etc/resolv.conf
      make sure it's showing
         nameserver 8.8.8.8
         
      If it's no matching above run the following commands
            sudo rm -f /etc/resolv.conf  # Delete the symbolic link
            sudo nano /etc/resolv.conf   # Create static file
               # Content of static resolv.conf
                 nameserver 8.8.8.8
      
     sudo reboot
     if you modify the resolv.conf, after reboot., check : sudo nano /etc/resolv.conf and make sure is showing google DNS

     Note if you are setting up this in ubuntu, this extra steps are required
     sudo nano /etc/NetworkManager/NetworkManager.conf
        [main]
        plugins=ifupdown,keyfile,ofono
        #dns=dnsmasq ==> MASK
     sudo service network-manager restart
  
  To check if changea worked
    
      ifconfig
      iwconfig
      or ping google.com

Starting at this point, you can SSH into the Pi, it's easy to copy/paste next commands, I recommend to use Putty

--> enable sudo
      
      sudo su
      
--> Setup Docker ,  command installs docker and sets the right permission.
  
      curl -sSL get.docker.com | sh && \
      sudo usermod pi -aG docker && \
      newgrp docker 

--> Disable swap - it's mandatory for Kubernetes to work on Pi
      
      sudo dphys-swapfile swapoff && \
      sudo dphys-swapfile uninstall && \
      sudo update-rc.d dphys-swapfile remove &&\
      sudo apt purge dphys-swapfile
      
      When asked enter Y

--> Change CGROUP 

      sudo nano /boot/cmdline.txt
      Add at the endof the line add: cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
      keep only one line
      and sudo reboot


--> run this to setup Kubernetes 
     
     sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - 
     Response should be OK
     
     sudo nano /etc/apt/sources.list.d/kubernetes.list
     Add this to the file: deb http://apt.kubernetes.io/ kubernetes-xenial main
     
     Then run 
     sudo apt-get update -q 
     sudo apt-get install -qy kubeadm
    
     sudo kubeadm config images pull -v3 (this will take some minutes)

-->  NOW lets setup the -- MASTER -- node, (ONLY FOR MASTER - for worker nodes read below)
      
      ---------- AGAIN THIS COMMAND IS ONLY FOR THE MASTER, STOP HERE AND LOOK BELOW FOR WORKER SETUP --------------------
         sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.X.X (XX your Master ip)
      --------------------------------------------------------------------------------------------------------------------
  
  Note we will use Flannel, you can use any other but please check Kube official documentation)  
  Users of weave-wrok had reported issues with ARM
  
 When this complete , check at the instructions , you need to run this to enable the master, see below

    Your Kubernetes control-plane has initialized successfully!
    To start using your cluster, you need to run the following as a regular user:
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config

    You should now deploy a pod network to the cluster.
    
    --> In my case I 'm using flannel, runbelow command
         kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
    
    This is from kube init, no need to do it --> Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      https://kubernetes.io/docs/concepts/cluster-administration/addons/

    Then you can join any number of worker nodes by running the following on each as root:
    kubeadm join 192.168.0.40:6443 --token cq3wji.jc8oe98m0lp4wyba \--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxx

Important Note: This last statement, you need to copy it to join workers nodes to the master
    if by any chance you forgot it, run this command on the master: kubeadm token list
    If you need to create a new token use: kubeadm token create --print-join-command



    
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

  For some reasons if the Pi that holds the master reboots, you will get an error like host port connection not allowed when get back from the reboot and try to run kubectl, apply the following fix
  Error: The connection to the server localhost:8080 was refused - did you specify the right host or port?
 
      sudo -i
      swapoff -a
      exit
      strace -eopenat kubectl version

  # Worker Node Setup
  1.- Adding Worker Nodes
   For the worker nodes on others raspberries, repeat all steps above except the kubeadm init command, for the workers use the join command
   
    Remember to Change hostname
      Use the raspi-config utility to change the hostname to k8s-worker-1 or similar and then reboot.
    
    Join the cluster
    Replace the token / IP for the output you got from the master node, for example:
      Remember this is coming from step 10
      sudo kubeadm join --token <token> <master-node-ip>:6443 --discovery-token-ca-cert-hash sha256:<sha256>
  
        Important Note: This last statement, you need to copy it to join workers nodes to the master, run this on the master
        if by any chance you forgot it, run this command on the master: kubeadm token list
        If you need to create a new token use: kubeadm token create --print-join-command
  
    

At the end should look something like this, names may change based on your nodes names

      pi@pi-k8-master:~ $ kubectl get nodes
      NAME            STATUS   ROLES    AGE   VERSION
      pi-k8-master    Ready    master   97m   v1.16.3
      pi-k8-node1     Ready    <none>   52m   v1.16.3
      pi-k8-worker2   Ready    <none>   16m   v1.16.3


      pi@pi-k8-master:~ $ kubectl get pods --all-namespaces
      NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE
      kube-system   coredns-5644d7b6d9-qz7rb               1/1     Running   0          97m
      kube-system   coredns-5644d7b6d9-sxqfp               1/1     Running   0          97m
      kube-system   etcd-pi-k8-master                      1/1     Running   0          96m
      kube-system   kube-apiserver-pi-k8-master            1/1     Running   0          96m
      kube-system   kube-controller-manager-pi-k8-master   1/1     Running   0          96m
      kube-system   kube-flannel-ds-arm-kjrbz              1/1     Running   0          93m
      kube-system   kube-flannel-ds-arm-lzwzg              1/1     Running   0          16m
      kube-system   kube-flannel-ds-arm-szwc6              1/1     Running   0          52m
      kube-system   kube-proxy-gk2p6                       1/1     Running   0          52m
      kube-system   kube-proxy-lc8lb                       1/1     Running   0          16m
      kube-system   kube-proxy-xr6f9                       1/1     Running   0          97m
      kube-system   kube-scheduler-pi-k8-master            1/1     Running   0          96m



Usefule commands
Reset kube clkuster: 
        
         kubeadmin reset

Delete docker

      apt remove docker -y
      sudo dpkg --purge docker-ce
 
 Kubectl useful commands
       
        kubectl describe pod kube-controller-manager-XXXX -n kube-system (XXX is the name of your master controller, check the get --all-namespace output)
        
        kubectl get pods -o wide  --> list all pods deployments and in which node they are running
        NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE            NOMINATED NODE   READINESS GATES
        nginx-86c57db685-qtzfj   1/1     Running   0          12m   10.244.2.3   pi-k8-worker2   <none>   


         kubectl get deployments
         NAME    READY   UP-TO-DATE   AVAILABLE   AGE
         nginx   1/1     1            1           26m

         kubectl get all -A
         
         Use to delete dashboard
         kubectl --namespace kube-system delete deployment kubernetes-dashboard
 
Pi commands

      check CPU temp:  /opt/vc/bin/vcgencmd measure_temp

# Deploying on Kubernetes
https://dev.to/anton2079/kubernetes-k8s-private-cloud-with-raspberry-pi-4s-k0d
https://www.infralovers.com/en/articles/2017/04/22/kubernetes-and-traefik-on-raspberry/
https://www.youtube.com/watch?v=XvlkYL1dGbw
https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/

If you have issue with reaching between PODs run this command
    
    sudo iptables -P FORWARD ACCEPT
    https://github.com/coreos/flannel/issues/799
    You can add this to startup
    sudo nano /etc/rc.local
 


