Project Structure
 -backend/ folder: 
    app.py (The Python application code)
    Dockerfile (Instructions to containerize the backend)

 -frontend/ folder:
    index.html (The user interface/web page)
    Dockerfile (Instructions to containerize the frontend)

 -manifests/ folder:
    task.yaml (A single file containing all Kubernetes configurations)

-----------------------------------------------------------

#login 
docker login

# backend
docker build -t nabilaebrahim/my-backend:v1 ./backend
docker push nabilaebrahim/my-backend:v1

# frontend
docker build -t nabilaebrahim/my-frontend:v1 ./frontend
docker push nabilaebrahim/my-frontend:v1

#
minikube start

#ingress controller
minikube addons enable ingress

#
1-kubectl create namespace frontend-ns
2-ubectl create namespace backe
3-kubectl create namespace db-storage-ns
4-openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=myapp.local"
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key -n frontend-ns

[

*Important Note: Why we create Namespaces manually first
You might notice that we create the Namespaces manually even though they are already defined in the manifest.yaml file.

*Here is why: In our manifest, the Ingress is configured to use a secret called tls-secret to enable HTTPS. If we apply the manifest file first, Kubernetes will look for this secret, won't find it, and will result in an Error.

*The correct order is:

1-Create the Namespaces manually.

2-Create the TLS Secret inside the namespace.

3-Then apply the manifest file.

By doing this, the secret is already waiting for the manifest, and everything runs smoothly without errors.

]




#
kubectl apply -f manifests/task.yaml

#
kubectl get pods -A

#Accessing the Application
1-Open a Tunnel (open a new terminal  and put this command  --> minikube tunnel )
2-in the current terminal window put this command -->

echo "$(minikube ip) nabila.local" | sudo tee -a /etc/hosts 

3- on browser       https://nabila.local  


##                                                                                                                                                                       Handling the Browser Security Warning
When you open the site, you will see a warning: "Warning: Potential Security Risk Ahead."

Why is this happening? This is perfectly normal. The message appears because we are using a Self-signed certificate (created manually using OpenSSL). Firefox does not recognize it as an official certificate, which is expected in a Development Environment.

How to bypass this (2 Simple Steps):
1-Click on "Advanced": Click the Advanced button shown on the warning page.
2-Accept the Risk: A new section will appear. Click the button that says: "Accept the Risk and Continue."


###
-Redis Network Policy (redis-policy)
-Goal: To protect the Redis instance running in the db-storage-ns namespace.
-The Rule: By default, all traffic to Redis is blocked. Only Pods coming from the backend-ns namespace are allowed to communicate with Redis.                          

ex1:
-Testing the Policy: Allowing the Backend
-To verify that our policy works, we will send a "test message" to Redis from the   Backend zone.

-Run this command in your terminal:

    (kubectl run test-redis-allow --rm -it --image=busybox -n backend-ns -- nc -zv            redis-headless.db-storage-ns.svc.cluster.local 6379)
  Expected Result: You will see a message saying: 6379 port [tcp/6379] open.


ex2:
-We will now test the scenario where the Frontend tries to talk to Redis.
To ensure the Network Policies work correctly, we need to restart Minikube with a specific network plugin (Calico). Follow these steps:

1-First, delete the existing cluster to start fresh: --> ( minikube delete)

2-Start Minikube with Calico --> ( minikube start --cni=calico --memory=2200mb --cpus=2  )

3-Enable Ingress --> (nikube addons enable ingress)

4-Create and Label Namespaces --> ( 

kubectl create namespace frontend-ns
kubectl create namespace backend-ns
kubectl create namespace db-storage-ns

kubectl label namespace frontend-ns name=frontend-ns
kubectl label namespace backend-ns name=backend-ns
kubectl label namespace db-storage-ns name=db-storage-ns   )

5-Create the TLS Secret -->  (

kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key -n frontend-ns )

6- Deploy the Manifests  --> (  kubectl apply -f manifests/task.yaml  )

7- The "Denied" Test
Now, try to reach Redis from the Frontend namespace: -->  (

kubectl run test-redis-deny --rm -it --image=busybox -n frontend-ns -- nc -zv redis-headless.db-storage-ns.svc.cluster.local 6379  )

Expected Result: The command will get stuck

#
*To ensure that only the Frontend can communicate with the Backend. We will restrict access so that any other traffic from outside the frontend-ns is blocked.

-We will add a new Network Policy to the backend-ns. This policy allows incoming traffic only from the Frontend on Port 5000.

Test 1: we will test if the Frontend can successfully reach the Backend.

( kubectl run test-allowed --rm -it --image=busybox -n frontend-ns -- nc -zv backend-service.backend-ns.svc.cluster.local 5000 )

Expected Result: 5000 port [tcp/5000] open.

Test 2: Now, let’s try to reach the Backend from a different namespace (like the default namespace).

( kubectl run test-blocked --rm -it --image=busybox -n default -- nc -zv backend-service.backend-ns.svc.cluster.local 5000  )

Expected Result: The command will stay stuck. 


##


-mplementing Horizontal Pod Autoscaler (HPA)
-The Goal: We implemented the HPA for the backend to handle high traffic automatically. We configured the Target CPU Utilization to 50%. This means if a Pod consumes more than half of its resources, the HPA will start creating new Pods (Scaling Up) to share the load.

Minimum Pods: 1

Maximum Pods: 5

*Step 1: Enable the Metrics Server
The HPA acts like a "scale," and it needs "sensors" to measure CPU usage. The Metrics Server provides these measurements. We enabled it using: ( minikube addons enable metrics-server )


*Step 2: Monitoring and Load Testing (Two Terminals)
To see the HPA in action, we will use two separate terminal windows:

Terminal 1: Monitoring the HPA Run this command to watch the HPA status and the number of replicas in real-time:  ( kubectl get hpa -n backend-ns -w  )

Terminal 2: Generating Load (Stress Test) Run this command to create a continuous loop of requests to the backend, which will increase the CPU usage:

( kubectl run load-generator --rm -it --image=busybox:1.28 -n frontend-ns --labels="app=frontend" -- /bin/sh -c "while true; do wget -q -O- http://backend-service.backend-ns.svc.cluster.local:5000; done" )



*Step 3: Observation (The Results)
Scaling Up: In Terminal 1, you will notice the REPLICAS count starts to increase (from 1 to 2, 3, etc.) as the load rises.

Scaling Down: Once you stop the stress test command in Terminal 2 (by pressing Ctrl+C), the CPU usage will drop, and you will see the number of replicas gradually decrease back to the minimum.


##
-Instead of deploying a container image that might have security vulnerabilities (weaknesses that hackers could use to break into your Pods), we perform a "comprehensive scan."
-Here is how to explain the Image Scanning steps using Trivy:
1- ( sudo yum install wget -y
wget https://github.com/aquasecurity/trivy/releases/download/v0.44.1/trivy_0.44.1_Linux-64bit.rpm
sudo yum localinstall trivy_0.44.1_Linux-64bit.rpm -y  )

2- sudo rpm -ivh trivy_0.44.1_Linux-64bit.rpm

3- trivy --version

4- trivy image nabilaebrahim/my-backend:v1 

##

-Digital Signing with Cosign
-The Goal: The purpose of using Cosign is to ensure Image Integrity. It guarantees that the image running in your cluster is exactly your original image and has not been changed or replaced by anyone else.

1-  ( wget https://github.com/sigstore/cosign/releases/download/v2.2.1/cosign-linux-amd64 )

2- ( chmod +x cosign-linux-amd64 )

3- ( sudo mv cosign-linux-amd64 /usr/local/bin/cosign )

4- ( cosign version )

5- (cosign generate-key-pair)

6- ( docker login )


7-( ( cosign sign --key cosign.key nabilaebrahim/my-backend:v1 )                           and then  enter password the key  )


8-( cosign verify --key cosign.pub nabilaebrahim/my-backend:v1 )




 
















