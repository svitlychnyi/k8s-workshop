### Setup Cluster:
1. Create new VM using Debian 9 image
    * Set name: k8s-master
    * Set memory 16GB
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

2.1. Set timezone
    * dpkg-reconfigure tzdata
    * Jenkins deployment:
        * -Duser.timezone=Europe/Berlin

3. Disable swap (https://unix.stackexchange.com/questions/224156/how-to-safely-turn-off-swap-permanently-and-reclaim-the-space-on-debian-jessie)  
    * swapoff -a
    * Debian:
        * Delete the swap partition
            * w
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

    * Insecure Registry (10.98.163.131 - Service IP address, not a pods)
        * Run on each node:
            * echo '{ "bip":"172.18.0.1/24", "insecure-registries":["10.98.163.131:5000"] }' > /etc/docker/daemon.json
            * service docker restart

5. Install kubeadm, kubelet and kubectl (https://kubernetes.io/docs/setup/independent/install-kubeadm/)
    * apt-get update && apt-get install -y apt-transport-https curl
    * curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    * cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    * deb http://apt.kubernetes.io/ kubernetes-xenial main
    * EOF
    * apt-get update
    * apt-get install -y kubelet kubeadm kubectl

5.1. To install specific version of the package it is enough to define it during the apt-get install command:
     
     NOTE: In the current case kubectl and kubelet packages are installed by dependencies when we install kubeadm, so all these three packages should be installed with a specific version:
         $ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - && \
           echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list && \
           sudo apt-get update -q && \
           sudo apt-get install -qy kubelet=<version> kubectl=<version> kubeadm=<version> 
       example:
         $ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - && \
           echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list && \
           sudo apt-get update -q && \
           sudo apt-get install -qy kubelet=1.11.1-00 kubectl=1.11.1-00 kubeadm=1.11.1-00 kubernetes-cni=0.6.0-00
       
     NOTE: to get actual list of supported version run following:
         $ curl -s https://packages.cloud.google.com/apt/dists/kubernetes-xenial/main/binary-amd64/Packages | grep Version | awk '{print $2}'

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
                * kubeadm join 172.20.0.74:6443 --token 5tcszs.d2n16tbfni3bj2zc --discovery-token-ca-cert-hash sha256:62cba554ccffcde08e8b227546ce87cb41fdbf9b627d1914c32fc9b729e2cc37
        * Verify the nodes on master
            * kubectl get nodes
            * Expected result:
        	```
        	    NAME         STATUS    ROLES     AGE       VERSION
        	    k8s-master   Ready     master    12h       v1.11.1
        	    k8s-node1    Ready     <none>    12h       v1.11.1
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
    * [master]:
        * apt-get install openssh-server
        * kubectl proxy
    * [local]:
        * ssh-copy-id <user>@<master-ip>
            * ssh <user>@<master-ip>
        * ssh -f <user>@<master-ip> -L 8001:127.0.0.1:8001 -N
            * lsof -ti:8001 | xargs kill -9
        * http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login
        * Press skip on login page
    * Metrics
        * helm install -name heapster --namespace kube-system stable/heapster
        * kubectl -n kube-system edit deploy heapster-heapster
            * in spec/template/spec/command
                * - --source=kubernetes.summary_api:https://kubernetes.default?kubeletHttps=true&kubeletPort=10250&insecure=true
        * kubectl edit clusterRole system:heapster
            * add "- nodes/stats" to apiGroups.resources

10. Persistent storage - NFS (https://joshrendek.com/2018/04/kubernetes-on-bare-metal/#registry)
        * Setup NFS server on master node (https://www.server-world.info/en/note?os=Debian_8&p=nfs)
            * ssh root@<master-node>
            * mkdir -p /k8s-storage/volume1
            * Debian:
                * aptitude -y install nfs-kernel-server
            * Ubuntu:
                * apt-get -y install nfs-kernel-server
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
11. Helm
    * Install tool
        * curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get >> helm.sh
        * chmod 700 helm.sh
        * ./helm.sh
    * Init on k8s master
        * kubectl create serviceaccount --namespace kube-system tiller
        * kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
        * helm init --service-account tiller
        * export HELM_HOST=<service-endpoint>:44134
        * helm version

12. Loadbalancer
    * vim traefik.yaml:
        ```
            serviceType: NodePort
            externalTrafficPolicy: Cluster
            replicas: 1
            cpuRequest: 10m
            memoryRequest: 20Mi
            cpuLimit: 100m
            memoryLimit: 30Mi
            debug:
              enabled: false
            ssl:
              enabled: true
            acme:
              enabled: true
              email: your_email@example.com
              staging: false
              logging: true
              challengeType: http-01
              persistence:
                enabled: true
                annotations: {
                  volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
                }
                accessMode: ReadWriteOnce
                size: 1Gi
            dashboard:
              enabled: true
              domain: dashboard.k8s.workshop.local # YOUR DOMAIN HERE
              service:
                annotations:
                  kubernetes.io/ingress.class: traefik
              auth:
                basic:
                  admin: $apr1$VEY.Vz8J$s/gLBw8q0WKKI3TqhUrXk1 # FILL THIS IN WITH A HTPASSWD VALUE
            gzip:
              enabled: true
            accessLogs:
              enabled: false
              ## Path to the access logs file. If not provided, Traefik defaults it to stdout.
              # filePath: ""
              format: common  # choices are: common, json
            rbac:
              enabled: true
            ## Enable the /metrics endpoint, for now only supports prometheus
            ## set to true to enable metric collection by prometheus
            service:
              nodePorts:
                http: 30080
                https: 30443
            deployment:
              hostPort:
                httpEnabled: true
                httpsEnabled: true
        ```
    * helm install stable/traefik --name traefik -f traefik.yaml --namespace kube-system
    * Test:
        * kubectl run hello-minikube --image=k8s.gcr.io/echoserver:1.10 --port=8080
        * kubectl expose deployment hello-minikube --type=NodePort
        * curl <node-ip>:<service-port>
        * create  Ingress:
            ```
                apiVersion: extensions/v1beta1
                kind: Ingress
                metadata:
                  name: hello-minikube
                  annotations:
                    kubernetes.io/ingress.class: traefik
                spec:
                  rules:
                  - host: hello-minikube.k8s.workshop.local
                    http:
                      paths:
                      - backend:
                          serviceName: hello-minikube
                          servicePort: 8080
            ```
        * add local domain to the hosts file:
            * <node-ip> hello-minikube.k8s.workshop.local
        * open in browser:
            * http://hello-minikube.k8s.workshop.local:30080

13. Docker Registry
    * docker run --rm -ti xmartlabs/htpasswd evia evia2018 > htpasswd
    * cat htpasswd
    * vim registry.yaml:
        ```
            replicaCount: 1
            ingress:
              enabled: true
              # Used to create an Ingress record.
              hosts:
                - registry.k8s.workshop.local
              annotations:
                kubernetes.io/ingress.class: traefik
            persistence:
              accessMode: 'ReadWriteOnce'
              enabled: true
              size: 10Gi
              storageClass: 'managed-nfs-storage'
            # set the type of filesystem to use: filesystem, s3
            storage: filesystem
            secrets:
              haSharedSecret: ""
              htpasswd: "evia:$2y$05$NLaxeh6iNDFLVuDknNJvk.FK3NNTQeuO1Us9Z0MtwY8OWohOqrb6a"
        ```
    * helm install -f registry.yaml --name registry stable/docker-registry
    * kubectl create secret docker-registry regcred --docker-server=registry.k8s.workshop.local:30443 --docker-username=evia --docker-password=evia2018 --docker-email=k8s@mail.com
    * add Insecure Registry to each node + master (<registry-service-ip>)
        * vim /etc/docker/daemon.json
            * "insecure-registries": ["<registry-service-ip>:5000"]
        * service docker restart
    * docker login <registry-service-ip>:5000
        * evia / evia2018

14. Jenkins
    * kubectl create namespace jenkins
    * vim jenkins-account.yaml
        ```
            apiVersion: v1
            kind: ServiceAccount
            metadata:
              name: default
              namespace: jenkins
        ```
    * kubectl create -f jenkins-account.yaml
    * vim jenkins-role.yaml
        ```
            apiVersion: rbac.authorization.k8s.io/v1beta1
            kind: ClusterRoleBinding
            metadata:
              name: default
            roleRef:
              apiGroup: rbac.authorization.k8s.io
              kind: ClusterRole
              name: cluster-admin
            subjects:
            - kind: ServiceAccount
              name: default
              namespace: jenkins
        ```
    * kubectl create -f jenkins-account.yaml
    * vim jenkins.yaml
        ```
            Master:
              Name: jenkins-master
              Image: "jenkins/jenkins"
              ImageTag: "lts"
              ImagePullPolicy: "Always"
              Component: "jenkins-master"
              UseSecurity: true
              AdminUser: evia
              AdminPassword: evia2018
              Cpu: "200m"
              Memory: "256Mi"
              ServicePort: 8080
              ServiceType: ClusterIP
              ServiceAnnotations: {}
              ContainerPort: 8080
              HealthProbes: true
              HealthProbesTimeout: 60
              HealthProbeLivenessFailureThreshold: 12
              SlaveListenerPort: 50000
              DisabledAgentProtocols:
                - JNLP-connect
                - JNLP2-connect
              CSRF:
                DefaultCrumbIssuer:
                  Enabled: true
                  ProxyCompatability: true
              CLI: false
              SlaveListenerServiceType: ClusterIP
              SlaveListenerServiceAnnotations: {}
              LoadBalancerSourceRanges:
              - 0.0.0.0/0
              InstallPlugins:
                - kubernetes
                - workflow-aggregator
                - workflow-job
                - credentials-binding
                - git
                - ghprb
                - blueocean
              InitScripts:
              CustomConfigMap: false
              NodeSelector: {}
              Tolerations: {}

              Ingress:
                ApiVersion: extensions/v1beta1
                Annotations:
                  kubernetes.io/ingress.class: traefik
                TLS:

            Agent:
              Enabled: true
              Image: jenkins/jnlp-slave
              ImageTag: 3.10-1
              Component: "jenkins-slave"
              Privileged: false
              Cpu: "200m"
              Memory: "256Mi"
              AlwaysPullImage: false
              volumes:
              NodeSelector: {}

            Persistence:
              Enabled: true
              Annotations: {
                volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
              }
              AccessMode: ReadWriteOnce
              Size: 8Gi
              volumes:
              mounts:

            NetworkPolicy:
              Enabled: false
              ApiVersion: extensions/v1beta1

            rbac:
              install: false
              serviceAccountName: default
              apiVersion: v1beta1
              roleRef: cluster-admin
        ```
    * helm install -f jenkins.yaml --name jenkins --namespace jenkins stable/jenkins
    * create ingress
        ```
            apiVersion: extensions/v1beta1
            kind: Ingress
            metadata:
              name: jenkins
              annotations:
                kubernetes.io/ingress.class: traefik
            spec:
              rules:
              - host: jenkins.k8s.workshop.local
                http:
                  paths:
                  - backend:
                      serviceName: jenkins
                      servicePort: 8080
        ```
    * System.setProperty("org.jenkinsci.plugins.durabletask.BourneShellScript.HEARTBEAT_CHECK_INTERVAL", "36000")

15. Auto-scale:
    * https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

16. Prometheus:
    * https://medium.com/@timfpark/simple-kubernetes-cluster-monitoring-with-prometheus-and-grafana-dd27edb1641

17. SonarQube:
    * https://coderise.io/installing-sonarqube-in-kubernetes/

18. Garbage collection:
    * Disk usage in current folder:
        du -sh -- *
    * Delete old persistent volumes:
        rm -r /k8s-storage/volume1/archived*
    * Clean up docker:
        docker system prune