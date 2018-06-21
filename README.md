# k8s-workshop

* Day 1
    * Motivation
    * Overview
        * History
        * Architecture
        * Components
        * Tools
            * Minikube
            * kubectl
            * kubeadm
    * Objects: https://kubernetes.io/docs/concepts
        * Pod: https://kubernetes.io/docs/concepts/workloads/pods/pod-overview
        * ReplicaSet: https://kubernetes.io/docs/concepts/workloads/controllers/replicaset
        * Deployment: https://kubernetes.io/docs/concepts/workloads/controllers/deployment
        * Service: https://kubernetes.io/docs/concepts/services-networking/service
            * Create deployment with container image: joffotron/docker-net-tools
            * kubectl edit deployment/<deployment>
                * command: ["/bin/sh"]
                * args: ["-c", "while true; do echo hello; sleep 10;done"]
            * kubectl exec -it <pod> sh
            * curl $HELLO_MINIKUBE_SERVICE_HOST:$HELLO_MINIKUBE_SERVICE_PORT
        * Ingress: https://kubernetes.io/docs/concepts/services-networking/ingress
        * Horizontal Pod Autoscaler: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale
            * kubectl run resource-consumer --image=k8s.gcr.io/resource_consumer:beta --expose --service-overrides='{ "spec": { "type": "NodePort" } }' --port 8080
            * Add resource limit: https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource
                * kubectl edit deployment/resource-consumer
                * limit: 0.7, requested: 0.1
            * kubectl autoscale deployment resource-consumer --min=1 --max=10 --cpu-percent=80
            * curl --data "millicores=300&durationSec=600" http://10.0.2.15:<NodePort>/ConsumeCPU
        * Persistent Volumes: https://kubernetes.io/docs/concepts/storage/persistent-volumes
        * Secrets: https://kubernetes.io/docs/concepts/configuration/secret

* Day 2 (Might change after Day 1)
    * Installation
        * Master
        * Node
    * Setup
        * Dashboard
        * Monitoring
        * Helm
        * Persistent storage
        * Loadbalancer
        * Docker registry
        * Jenkins
    * Demo
        * ? Demo app
        * ? DDos
        * ? Pipeline




