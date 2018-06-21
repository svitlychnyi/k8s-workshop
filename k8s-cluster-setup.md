### Setup Cluster:
1. Create new VM using Debian 9 image (http://cloud.proway-fs.local:9869/)
    * Set name: k8s-master
    * Set memory 4GB
    * Set CPU 0.05
    * Set HDD 80GB
    * Set network default
    * Set role default

2. Rename hostname
    * vim /etc/hostname
    * change to unique node name (k8s-master, k8s-node1, ...)
    * vim /etc/hosts
    * replace localhost in first line to node name
    * add node name to second line
    * Restart VM

3. Disable swap (https://unix.stackexchange.com/questions/224156/how-to-safely-turn-off-swap-permanently-and-reclaim-the-space-on-debian-jessie)  
    * swapoff -a
    * Debian:
        * Delete the swap partition
            * fdisk /dev/vdb
            * n -> ceate new primary partition with default settings
            * w
    * Ubuntu:
        * Comment out swap entries in:
            * /etc/fstab
            * /etc/initramfs-tools/conf.d/resume

4. Install Docker
    * Debian:
        * apt-get update
        * apt-get install -y \
              apt-transport-https \
              ca-certificates \
              curl \
              software-properties-common \
              nfs-common
        * curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
        * add-apt-repository \
              "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
              $(lsb_release -cs) \
              stable"
        * apt-get update && apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.03 | head -1 | awk '{print $3}')
    * Ubuntu:
        * apt install docker.io

    * Insecure Registry (10.109.49.71 - Service IP address, not a pods)
        * Run on each node:
            * echo '{ "bip":"172.18.0.1/16", "insecure-registries":["10.109.49.71:5000"] }' > /etc/docker/daemon.json
            * service docker restart

5. Install kubeadm, kubelet and kubectl (https://kubernetes.io/docs/setup/independent/install-kubeadm/)
    * apt-get update && apt-get install -y apt-transport-https curl
    * curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    * cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    * deb http://apt.kubernetes.io/ kubernetes-xenial main
    * EOF
    * apt-get update
    * apt-get install -y kubelet kubeadm kubectl


6. Setup cluster node (follow steps 1 - 5)

7. Init cluster
    * Configure cgroup driver used by kubelet on Master Node
        * docker info | grep -i cgroup
        * vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
        * add following: Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs" and add $KUBELET_CGROUP_ARGS to the call line
        * systemctl daemon-reload
        * systemctl restart kubelet
    * Init master
        * kubeadm init --pod-network-cidr=10.244.0.0/16
        * Save join command from result output
        * Save config for kubectl
            * mkdir -p $HOME/.kube
            * cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
            * chown $(id -u):$(id -g) $HOME/.kube/config
        * Control k8s from local machine
            * scp root@<master-ip>:/etc/kubernetes/admin.conf .
            * export KUBECONFIG=./admin.conf
    * Init network (https://github.com/coreos/flannel)
        * sysctl net.bridge.bridge-nf-call-iptables=1
        * kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
        * Verify pods
            * kubectl get pods --all-namespaces
            * Expected result:
            ```
                NAMESPACE     NAME                                 READY     STATUS    RESTARTS   AGE
                kube-system   etcd-k8s-master                      1/1       Running   0          12h
                kube-system   kube-apiserver-k8s-master            1/1       Running   0          12h
                kube-system   kube-controller-manager-k8s-master   1/1       Running   0          12h
                kube-system   kube-dns-6f4fd4bdf-p4nxg             3/3       Running   0          12h
                kube-system   kube-flannel-ds-nbft5                1/1       Running   0          12h
                kube-system   kube-flannel-ds-swwv2                1/1       Running   3          12h
                kube-system   kube-proxy-5pgbx                     1/1       Running   3          12h
                kube-system   kube-proxy-twz2v                     1/1       Running   0          12h
                kube-system   kube-scheduler-k8s-master            1/1       Running   0          12h
            ```
    * Join the node
        * From Master:
            * kubeadm token create --print-join-command
        * From Node:
            * Run generated command from previous step
                * kubeadm join 172.20.0.74:6443 --token 5tczzv.d2n16tbfni3bj2zc --discovery-token-ca-cert-hash sha256:62cbc574ccffcde08e8b227546ce87cb41fdbf9b627d1914c32fc9b729e2cc37
        * Verify the nodes on master
            * kubectl get nodes
            * Expected result:
        	```
        	    NAME         STATUS    ROLES     AGE       VERSION
        	    k8s-master   Ready     master    12h       v1.9.2
        	    k8s-node1    Ready     <none>    12h       v1.9.2
            ```

9. Dashboard
    * kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
    * kubectl create -f dashboard-admin.yaml
        ```
            apiVersion: rbac.authorization.k8s.io/v1beta1
            kind: ClusterRoleBinding
            metadata:
              name: kubernetes-dashboard
              labels:
                k8s-app: kubernetes-dashboard
            roleRef:
              apiGroup: rbac.authorization.k8s.io
              kind: ClusterRole
              name: cluster-admin
            subjects:
            - kind: ServiceAccount
              name: kubernetes-dashboard
              namespace: kube-system
        ```
    * [master]: kubectl proxy
    * [local]: ssh -f <user>@<master-ip> -L 8001:127.0.0.1:8001 -N
    * http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login
        * lsof -ti:8001 | xargs kill -9
    * Press skip on login page

10. Helm
    * Install tool
        * curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get >> helm.sh
        * chmod 700 helm.sh
        * ./helm.sh
    * Init on k8s master
        * kubectl create serviceaccount --namespace kube-system tiller
        * kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
        * helm init --service-account tiller
        * helm ls

11.

10. LoadBalancer - Ingress (https://kubernetes.io/docs/concepts/services-networking/ingress/)
    * curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml \
          | kubectl apply -f -
    * curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml \
          | kubectl apply -f -
    * curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml \
          | kubectl apply -f -
    * curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml \
          | kubectl apply -f -
    * curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml \
          | kubectl apply -f -
    * curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/rbac.yaml \
          | kubectl apply -f -
    * curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/with-rbac.yaml \
          | kubectl apply -f -
    * curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml \
          | kubectl apply -f -
    * This will install and deploy Nginx servers as a loadbalancer on each node with the same port
    * In ingress-nginx namespace in the services: "ingress-nginx" port is specified
    * To expose service create ingress and add host name in the hosts of your local machine to map it

11. Persistent storage - NFS (https://joshrendek.com/2018/04/kubernetes-on-bare-metal/#registry)
    * Setup NFS server on master node (https://www.server-world.info/en/note?os=Debian_8&p=nfs)
        * ssh root@<master-node>
        * mkdir -p /k8s-storage/volume1
        * aptitude -y install nfs-kernel-server
        * Debian:
            * vim /etc/idmapd.conf
        * Ubuntu:
            * vim /etc/exports
        * add following: /k8s-storage 172.20.0.0/24(rw,no_root_squash)
        * systemctl restart nfs-kernel-server
    * Install nfs utils on all nodes:
        * apt install nfs-common
    * Set up NFS storage class in k8s
        * get git repo from https://github.com/kubernetes-incubator/external-storage.git
        * modify external-storage/nfs-client/deploy/class.yaml:
        ```
            apiVersion: storage.k8s.io/v1
            kind: StorageClass
            metadata:
              name: managed-nfs-storage
            provisioner: k8s-master-server/nfs
        ```
        * modify external-storage/nfs-client/deploy/deployment.yaml:
        ```
            kind: Deployment
            apiVersion: extensions/v1beta1
            metadata:
              name: nfs-client-provisioner
            spec:
              replicas: 1
              strategy:
                type: Recreate
              template:
                metadata:
                  labels:
                    app: nfs-client-provisioner
                spec:
                  serviceAccountName: nfs-client-provisioner
                  containers:
                    - name: nfs-client-provisioner
                      image: quay.io/external_storage/nfs-client-provisioner:latest
                      volumeMounts:
                        - name: nfs-client-root
                          mountPath: /persistentvolumes
                      env:
                        - name: PROVISIONER_NAME
                          value: k8s-master-server/nfs
                        - name: NFS_SERVER
                          value: <master-ip>
                        - name: NFS_PATH
                          value: /k8s-storage/volume1
                  volumes:
                    - name: nfs-client-root
                      nfs:
                        server: <master-ip>
                        path: /k8s-storage/volume1
        ```
        * Setup NFS storage in k8s
            * go to external-storage/nfs-client folder of the downloaded git project
            * kubectl apply -f deploy/deployment.yaml
            * kubectl apply -f deploy/class.yaml
            * kubectl create -f deploy/auth/serviceaccount.yaml
            * kubectl create -f deploy/auth/clusterrole.yaml
            * kubectl create -f deploy/auth/clusterrolebinding.yaml
            * kubectl patch deployment nfs-client-provisioner -p '{"spec":{"template":{"spec":{"serviceAccount":"nfs-client-provisioner"}}}}'
            * kubectl patch storageclass managed-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
        * Expected result:
            * Deployment should be created and running (nfs-client-provisioner)
            * Pod should be created and running (nfs-client-provisioner)
        * Test the result
            * Create a file called nfs-test.yaml:
                ```
                    kind: PersistentVolumeClaim
                    apiVersion: v1
                    metadata:
                      name: test-claim
                      annotations:
                        volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
                    spec:
                      accessModes:
                        - ReadWriteMany
                      resources:
                        requests:
                          storage: 1Mi
                    ---
                    kind: Pod
                    apiVersion: v1
                    metadata:
                      name: test-pod
                    spec:
                      containers:
                      - name: test-pod
                        image: gcr.io/google_containers/busybox:1.24
                        command:
                          - "/bin/sh"
                        args:
                          - "-c"
                          - "touch /mnt/SUCCESS && exit 0 || exit 1"
                        volumeMounts:
                          - name: nfs-pvc
                            mountPath: "/mnt"
                      restartPolicy: "Never"
                      volumes:
                        - name: nfs-pvc
                          persistentVolumeClaim:
                            claimName: test-claim
                ```
            * kubectl apply -f nfs-test.yaml
            * Expected result:
                * Claim should be created (test-claim)
                * Volume should be created (pvc-<UID>)

Helm
    * SSL (https://github.com/kubernetes/helm/blob/master/docs/tiller_ssl.md)
    * https://github.com/kubernetes/helm/issues/3460:
        * kubectl delete svc tiller-deploy -n kube-system
        * kubectl -n kube-system delete deploy tiller-deploy
        * kubectl create serviceaccount --namespace kube-system tiller
        * kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
        * helm init --service-account tiller
        * helm ls


Docker Registry
    * docker run --rm -ti xmartlabs/htpasswd <username> <password> > htpasswd
    * kubectl create secret docker-registry regcred --docker-server=registry.k8s.test.local:30443 --docker-username=admin --docker-password=admin --docker-email=k8s@mail.com
    * kubectl create secret docker-registry regcred -n oca-web --docker-server=10.109.49.71:5000 --docker-username=admin --docker-password=admin --docker-email=k8s@mail.com
    * Insecure Registry (10.109.49.71 - Service IP address, not a pods)
        * Run on each node:
            * echo '{ "bip":"172.18.0.1/16", "insecure-registries":["10.109.49.71:5000"] }' > /etc/docker/daemon.json
            * service docker restart

ifconfig br-ac6dbdbb51c3 down // for 172.17.0.1 net

https://akomljen.com/set-up-a-jenkins-ci-cd-pipeline-with-kubernetes/

{ "auths": { "10.109.49.71:5000": { "auth": "YWRtaW46YWRtaW4=" } } }


http://oca.k8s.test.local:30080/



Go to “Manage Jenkins” -> “Script console” Type and run below command:
System.setProperty("hudson.model.DirectoryBrowserSupport.CSP", "")

NOTE: Jenkins agent must login to registry at least once: run build web app job before e2e after jenkins restart
(TODO: add credentials to containerTemplate in pipeline)