* kubctl
    * install: https://kubernetes.io/docs/tasks/tools/install-kubectl/
    * test on minikube:
        * kubectl run hello-minikube --image=k8s.gcr.io/echoserver:1.10 --port=8080
        * kubectl expose deployment hello-minikube --type=NodePort
        * curl $(minikube service hello-minikube --url)
            * Open port: http://ask.xmodulo.com/access-nat-guest-from-host-virtualbox.html

* minikube
    * configure Docker daemon:
        * ip address
        * echo '{ "bip":"172.18.0.1/16", "insecure-registries":[] }' > /etc/docker/daemon.json
        * service docker restart
        * restart PC
        * ip address
    * install https://github.com/kubernetes/minikube/releases:
        * curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.28.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
    * setup:
        * sudo minikube start --vm-driver=none
        * sudo minikube dashboard
        * sudo minikube addons enable heapster
        * HPA: https://github.com/kubernetes-incubator/metrics-server

* kubeadm
    * install:
        * sudo apt-get install -y kubeadm

