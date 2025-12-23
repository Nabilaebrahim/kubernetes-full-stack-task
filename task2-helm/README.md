#

mkdir task2-helm
cd task2-helm
helm create my-app
rm -rf my-app/templates/*

#
-Now, we will start defining our custom values inside the my-app/values.yaml file.

#

my-app/templates/
   -templates/01-setup.yaml (ConfigMap  & Namespaces )
   -templates/02-database.yaml ( Storage configurations, the StatefulSets, and their associated services )
    
   -templates/03-apps.yaml (The Backend, the Frontend, and the HPA )
   -templates/04-network-ingress.yaml (NetworkPolicy  and Ingress )

#
Code Verification
(helm lint my-app/ )


#

( minikube start --cni=calico --memory=2200mb --cpus=2  )
(  minikube addons enable metrics-server )

#
Actual Installation
(helm install nabila-project ./my-app  )


#

( kubectl get pods,svc -A )
( kubectl get hpa -n backend-ns )

#

(kubectl create secret tls tls-secret \
  --cert=/home/nabila/Documents/task2/tls.crt \
  --key=/home/nabila/Documents/task2/tls.key \
  -n frontend-ns )



( kubectl get secrets -n frontend-ns )


#


- In one terminal, I will run [minikube tunnel], and in the other, I will run [curl -k https://nabila.local].








