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

---

# 🖥️ Dashboard Features

- The EMQX Dashboard is divided into several key sections, each giving insight into a different part of your MQTT broker and cluster.

1. **Overview** (Home)

   - Displays a summary of system health including uptime, node count, CPU/memory usage, and connection statistics.

  - Shows real-time metrics such as:

     - Number of connected clients

     - Incoming/outgoing message rates

     - Subscription counts

     - Cluster node status

2. **Clients**

   - Lists all currently connected MQTT clients.

   - Shows client information:

        - Client ID, username, IP address

        - Connection time and status

        -  Subscriptions and message statistics

    - Admins can disconnect, ban, or inspect individual clients.

3. **Subscriptions**

    -  Displays all MQTT topics that clients have subscribed to.

    - Shows which clients are subscribed to which topics and their QoS (Quality of Service) levels.

    - Useful for debugging message routing and verifying topic structures.

4. **Topics** & **Messages**

    - Provides insights into MQTT traffic across the broker.

    - Shows message throughput, dropped messages, and topic-level statistics.

    - You can verify if messages are being published and delivered correctly.

5. **Cluster Management**

    - Displays all nodes in the EMQX cluster.

    - Shows node roles, status, uptime, and synchronization state.

    - Allows manual node management (join/leave) if needed.

    - Confirms that clustering is functioning correctly when running multiple EMQX pods in Kubernetes.

6. **Rule Engine**

    - A no-code interface for defining rules to process MQTT messages.

    - You can trigger actions based on message content or topic, such as:

         - Forwarding messages to databases (MySQL, PostgreSQL, MongoDB)

         - Sending data to Kafka, Redis, or webhooks

    - Rules are created visually using SQL-like expressions.

7. **Authentication & Authorization**

    -  Configure how clients are authenticated (username/password, JWT, LDAP, etc.)

    - Manage Access Control Lists (ACLs) for controlling who can publish or subscribe to certain topics.

    - View and test authentication results in real time.

8. **Monitoring**

    - Built-in metrics charts for:

       - Connections

       - Messages per second

       - System resource utilization

    - Integrates with Prometheus and Grafana for advanced monitoring and alerting.

9. **Admin Settings**

    - Manage dashboard users and roles.

    - Change passwords, create new users, or restrict permissions.

    - Configure TLS certificates for secure (HTTPS) dashboard access.
 
    


   
