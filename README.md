# mqtt

## Goal
- Deploy an EMQX MQTT Broker cluster with 2 replicas on kubernetes, exposing it via a LoadBalancer service, so both broker can act as one logical MQTT broker.

## Step 1: Create Namespace 
 ```bash
    sudo kubectl create namespace emqx
 ```
## Step 2: Create Deployment
- Create a file named deployment.yaml
 ```bash
   apiVersion: apps/v1
kind: Deployment
metadata:
  name: emqx
  namespace: emqx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: emqx
  template:
    metadata:
      labels:
        app: emqx
    spec:
      containers:
      - name: emqx
        image: emqx/emqx:5.2.1
        ports:
        - containerPort: 1883  # MQTT
        - containerPort: 8083  # WebSocket
        - containerPort: 18083 # Dashboard
        env:
        - name: EMQX_NODE__NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: EMQX_CLUSTER__DISCOVERY
          value: dns
        - name: EMQX_CLUSTER__DNS__NAME
          value: emqx-headless.emqx.svc.cluster.local
        - name: EMQX_CLUSTER__DNS__APP
          value: emqx
        - name: EMQX_LISTENER__TCP__EXTERNAL
          value: "1883"
        - name: EMQX_LISTENER__WS__EXTERNAL
          value: "8083"
 
 ```  
---

## Step 3: Create Headless service(for internal clustering)
 - Create a file named `service.yaml`:
 ```bash
    apiVersion: v1
kind: Service
metadata:
  name: emqx-headless
  namespace: emqx
spec:
  clusterIP: None
  selector:
    app: emqx
  ports:
  - name: mqtt
    port: 1883
  - name: websocket
    port: 8083
  - name: dashboard
    port: 18083

 ```
---

## Step 4: Expose with a LoadBalancer (External Access) 
- Add a LoadBalancer service to expose EMQX externally:

 ```bash
    apiVersion: v1
kind: Service
metadata:
  name: emqx-lb
  namespace: emqx
spec:
  type: LoadBalancer
  selector:
    app: emqx
  ports:
  - name: mqtt
    port: 1883
    targetPort: 1883
  - name: websocket
    port: 8083
    targetPort: 8083
  - name: dashboard
    port: 18083
    targetPort: 18083

 ```
### Apply all files:
 ```bash
     Kubectl apply -f deployment.yaml
     kubectl apply -f service.yaml
 ```
---
## Step 5: Verify Deployment 
 ```bash
    kubectl get pods -n emqx
    kubectl get svc -n emqx 
 ```
- You should see:
   - 2Pods in Running state
   - LoadBalancer EXTERNAL-IP after a few seconds/minutes

## Step 6: Test MQTT Broker 
 - Once Loadbalancer IP is available, test with any MQTT client with the help of loadBalancer ip so u can connect broker with that IP and subscriber and pulisher communicate.
   ---

   # EMQX Dashboard
   - The EMQX Dashboard is a powerful web-based interface for managing,monitoring,and configuring your MQTT broker cluster. It provides real-time visibility into your EMQX deployment and allows administrators to perform most broker management task visually without using CLI commands.
  ## Accessing the Dashboard
  - After deploying EMQX (for example on kuberentes with a LoadBalancer), the dashboard is available on `port 18083`.
  - You can open it in a web browser:
   ```bash
    http://<EMQX-EXTERNAL-IP>:18083
  ```

 ### Default credentials:
 - Username: `admin`
 - Password: `public`
- (You can change credentials in the dashboard or via Helm values).
    


   
