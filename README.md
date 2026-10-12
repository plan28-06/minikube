# Lab Exercise 15 — Kubernetes with Minikube

**Environment:** Windows PowerShell  
**Purpose:** Deploy and expose an NGINX application, inspect Service routing, scale and recover Pods, perform a rolling update and rollback, use YAML manifests and ConfigMaps, and configure health probes.

> **Before starting:** This guide skips the downloading/installation phase. It assumes Docker Desktop, Minikube, and `kubectl` are already installed. Start Docker Desktop and use PowerShell for the commands below. Run the commands in order. Where a command contains `POD_NAME` or a placeholder, replace it with the actual name shown by `kubectl get pods`.

---

## Start Minikube and verify the cluster

Start the local Kubernetes cluster using Docker as the driver:

```powershell
minikube start --driver=docker
```

Check that the Kubernetes node is ready:

```powershell
kubectl get nodes
```

**Expected:** A node named `minikube` should eventually show `Ready`. Do not continue until the cluster is ready.

---

# Exercise 1 — Deploy and Expose an Application

## What the question asks

1. Start Minikube and verify that the cluster/node is ready.
2. Create a Deployment named `webapp` using the `nginx:latest` image with three replicas.
3. Verify the Deployment and running Pods.
4. Expose the Deployment using a NodePort Service named `webapp-service` on port 80.
5. Access the application through Minikube.
6. Display the Service details.

## Step 1: Create the Deployment

```powershell
kubectl create deployment webapp --image=nginx:latest --replicas=3
```

This creates a Deployment called `webapp`. Kubernetes manages three NGINX Pods for it.

## Step 2: Verify the Deployment and Pods

```powershell
kubectl get deployments
kubectl get pods
```

**Check:** The `webapp` Deployment should show `3/3` ready replicas after startup. Its three Pods should eventually show `Running` and `1/1` ready.

## Step 3: Expose the Deployment with a NodePort Service

```powershell
kubectl expose deployment webapp --name=webapp-service --type=NodePort --port=80
```

This creates a Service named `webapp-service` that routes port 80 traffic to the Pods selected by the Deployment's labels.

## Step 4: Display the Service

```powershell
kubectl get services
kubectl describe service webapp-service
```

The Service type should be `NodePort`.

## Step 5: Open the application

```powershell
minikube service webapp-service
```

This opens the Service URL in a browser when supported by the Minikube driver. The NGINX welcome page should appear. Keep the terminal open if Minikube runs a tunnel or proxy for the URL.

**Exercise 1 is complete when:** the Deployment has three ready replicas, the Service exists, and the NGINX page can be accessed.

---

# Exercise 2 — Labels, Selectors and Service Routing

## What the question asks

1. Display the labels of all `webapp` Pods.
2. Display the selector configured for `webapp-service`.
3. Display the endpoints associated with the Service.
4. Create a Deployment named `testapp` using NGINX with two replicas.
5. Verify that `testapp` Pods are not receiving traffic from `webapp-service`.
6. Explain why the Service routes traffic only to `webapp` Pods.

## Step 1: Display Pod labels

```powershell
kubectl get pods --show-labels
```

Find the Pods belonging to `webapp`. Their labels should include `app=webapp`.

## Step 2: Inspect the Service selector and endpoints

```powershell
kubectl describe service webapp-service
kubectl get endpoints webapp-service
```

**Check:**
- The Service selector should be `app=webapp`.
- The endpoints should list the IP addresses and port of the selected, ready `webapp` Pods.

## Step 3: Create the second Deployment

```powershell
kubectl create deployment testapp --image=nginx:latest --replicas=2
```

Verify both Deployments' Pods and labels:

```powershell
kubectl get pods --show-labels
```

Inspect the Service endpoints again:

```powershell
kubectl get endpoints webapp-service
```

**Expected observation:** `testapp` creates two additional Pods, but they do not appear in the endpoints for `webapp-service`.

## Explanation

A Kubernetes Service uses its **selector** to find matching Pods. `webapp-service` selects Pods labelled `app=webapp`. The `testapp` Pods are labelled `app=testapp`, so they do not match the selector and are not added to this Service's endpoints.

**Exercise 2 is complete when:** the selector and endpoints are displayed, and the two `testapp` Pods are separate from the Service's backend endpoints.

---

# Exercise 3 — Scaling and Self-Healing

## What the question asks

1. Ensure `webapp` is running with three replicas.
2. Scale the Deployment to five replicas.
3. Verify that five Pods are running.
4. Delete two `webapp` Pods manually.
5. Observe the Pods immediately after deletion and after recovery.
6. Verify that Kubernetes returns the Deployment to five replicas.
7. Scale the Deployment down to two replicas and observe the result.

## Step 1: Ensure the initial replica count is three

```powershell
kubectl scale deployment webapp --replicas=3
kubectl get deployments
kubectl get pods
```

Wait until the three `webapp` Pods are ready.

## Step 2: Scale up to five replicas

```powershell
kubectl scale deployment webapp --replicas=5
kubectl get deployments
kubectl get pods
```

Wait until five `webapp` Pods are ready.

## Step 3: Identify and delete two `webapp` Pods

List the Pods:

```powershell
kubectl get pods
```

Copy the name of one Pod belonging to `webapp`, then delete it:

```powershell
kubectl delete pod POD_NAME
```

Replace `POD_NAME` with the actual name. Repeat with a **different** `webapp` Pod.

Do not delete `testapp` or `webapp-yaml` Pods.

## Step 4: Observe recovery

```powershell
kubectl get pods -w
```

Watch for the deleted Pods to terminate and replacement Pods to be created. Press **Ctrl+C** to stop watching.

Verify the Deployment:

```powershell
kubectl get deployments
kubectl get pods
```

**Expected observation:** The Deployment controller creates replacement Pods to return the `webapp` Deployment to its desired count of five replicas.

## Step 5: Scale down to two replicas

```powershell
kubectl scale deployment webapp --replicas=2
kubectl get deployments
kubectl get pods
```

Wait until only two Pods belonging to `webapp` remain.

## Explanation

A Deployment maintains the desired number of replicas. If a managed Pod is deleted, Kubernetes creates a replacement. Scaling changes the desired replica count, and Kubernetes adjusts the number of Pods to match it.

**Exercise 3 is complete when:** you have observed five replicas, deleted two Pods and seen recovery, and then scaled the Deployment down to two.

---

# Exercise 4 — Rolling Update and Rollback

## What the question asks

1. Verify the current NGINX image used by `webapp`.
2. Update the Deployment to `nginx:1.27`.
3. Monitor the rollout until it completes.
4. Display the rollout history.
5. Roll back to the previous revision.
6. Verify that the rollback completed successfully.

## Step 1: Check the current image

```powershell
kubectl get deployment webapp -o wide
```

The Deployment was initially created with `nginx:latest`.

## Step 2: Update the image

```powershell
kubectl set image deployment/webapp nginx=nginx:1.27
```

This changes the image used by the `nginx` container in the `webapp` Deployment.

## Step 3: Monitor the rollout

```powershell
kubectl rollout status deployment/webapp
kubectl get pods
kubectl get deployment webapp -o wide
```

Wait until the rollout reports that it has completed successfully.

## Step 4: Display rollout history

```powershell
kubectl rollout history deployment/webapp
```

This displays the Deployment's available revision history.

## Step 5: Roll back to the previous revision

```powershell
kubectl rollout undo deployment/webapp
```

## Step 6: Verify the rollback

```powershell
kubectl rollout status deployment/webapp
kubectl get deployment webapp -o wide
kubectl describe deployment webapp
```

**Expected observation:** The Deployment returns to its previous revision's Pod template/image configuration. The replica count remains at the value set in Exercise 3 (two, unless you changed it).

**Exercise 4 is complete when:** the image update succeeds, rollout history is shown, and the rollback completes.

---

# Exercise 5 — YAML-Based Deployment

## What the question asks

1. Create a file named `deployment.yaml`.
2. Define a Deployment named `webapp-yaml`.
3. Configure three replicas.
4. Use `nginx:1.27` as the image.
5. Add the label `app: webapp-yaml` to the Pod template.
6. Apply the YAML file using `kubectl`.
7. Verify the Deployment and Pods.
8. Change the replica count from three to five and apply the file again.
9. Verify the new desired state.

## Step 1: Create the YAML file

Choose a working directory for your lab files. Open Notepad from PowerShell:

```powershell
notepad deployment.yaml
```

Paste the following YAML:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-yaml
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp-yaml
  template:
    metadata:
      labels:
        app: webapp-yaml
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Save the file as `deployment.yaml`. Make sure it is not accidentally saved as `deployment.yaml.txt`.

## Step 2: Apply the manifest

In PowerShell, change to the directory containing the file if necessary, then run:

```powershell
kubectl apply -f deployment.yaml
```

## Step 3: Verify the Deployment and Pods

```powershell
kubectl get deployments
kubectl get pods
kubectl get deployment webapp-yaml -o wide
```

**Check:** `webapp-yaml` should have three desired replicas, and its Pods should become ready.

## Step 4: Change the replica count to five

Open the file:

```powershell
notepad deployment.yaml
```

Change:

```yaml
replicas: 3
```

to:

```yaml
replicas: 5
```

Save the file, then apply it again:

```powershell
kubectl apply -f deployment.yaml
```

Verify the new state:

```powershell
kubectl get deployments
kubectl get pods
kubectl get deployment webapp-yaml
```

**Expected observation:** `webapp-yaml` now has five desired replicas.

## Explanation of the manifest

- `apiVersion: apps/v1` specifies the Kubernetes API version for a Deployment.
- `kind: Deployment` declares the resource type.
- `metadata.name` sets the Deployment name.
- `spec.replicas` sets the desired number of Pods.
- `selector.matchLabels` tells the Deployment which Pods it manages.
- `template.metadata.labels` assigns the label that must match the selector.
- `containers` defines the container name, image, and container port.

**Exercise 5 is complete when:** the manifest creates three replicas and then updates the Deployment to five replicas.

---

# Exercise 6 — ConfigMap and Application Configuration

## What the question asks

1. Create a ConfigMap named `app-config` containing `environment=development`.
2. Display and describe the ConfigMap.
3. Modify a Deployment so the value is injected as an environment variable named `APP_ENV`.
4. Apply the updated Deployment.
5. Verify the environment variable from inside a running Pod.

## Step 1: Create the ConfigMap

```powershell
kubectl create configmap app-config --from-literal=environment=development
```

## Step 2: Display and describe it

```powershell
kubectl get configmap
kubectl describe configmap app-config
```

**Check:** The ConfigMap data should include `environment` with value `development`.

## Step 3: Update `deployment.yaml`

For this exercise, modify the `webapp-yaml` Deployment created in Exercise 5.

Open the file:

```powershell
notepad deployment.yaml
```

Keep the existing manifest, and add the `env` section beneath the NGINX container's `image` field. The full container section should look like this:

```yaml
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: environment
```

Keep the indentation exactly as shown. Save the file.

## Step 4: Apply the updated Deployment

```powershell
kubectl apply -f deployment.yaml
```

The Pod template has changed, so Kubernetes will update the Deployment's Pods.

Wait until the Pods are ready:

```powershell
kubectl rollout status deployment/webapp-yaml
kubectl get pods
```

## Step 5: Verify `APP_ENV` inside a Pod

Find a Pod belonging to `webapp-yaml`:

```powershell
kubectl get pods
```

Copy its actual name and run:

```powershell
kubectl exec POD_NAME -- printenv APP_ENV
```

Replace `POD_NAME` with the real Pod name.

**Expected output:**

```text
development
```

## Explanation

A ConfigMap stores configuration separately from the container image. The `configMapKeyRef` reads the `environment` key from `app-config` and injects its value into the container as the environment variable `APP_ENV`.

**Exercise 6 is complete when:** the ConfigMap exists and `printenv APP_ENV` returns `development`.

---

# Exercise 7 — Readiness and Liveness Probes

## What the question asks

1. Create a Deployment with two NGINX replicas.
2. Add a readiness probe that checks HTTP availability on port 80.
3. Add a liveness probe that checks HTTP availability on port 80.
4. Apply the Deployment.
5. Display the Pod status and detailed probe information.
6. Explain the difference between readiness and liveness.
7. Observe the Service endpoints and explain why a Pod that is not ready should not receive new traffic.

## Step 1: Create `probes.yaml`

Open a new file:

```powershell
notepad probes.yaml
```

Paste the following manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-probes
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-probes
  template:
    metadata:
      labels:
        app: nginx-probes
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
```

Save the file as `probes.yaml`.

## Step 2: Apply the manifest

```powershell
kubectl apply -f probes.yaml
```

Verify the Deployment and Pods:

```powershell
kubectl get deployments
kubectl get pods
```

Wait until both `nginx-probes` Pods are ready.

## Step 3: Inspect a Pod's probe configuration

List the Pods:

```powershell
kubectl get pods
```

Copy the name of a Pod belonging to `nginx-probes`, then run:

```powershell
kubectl describe pod POD_NAME
```

Look at the `Readiness` and `Liveness` probe details and the Events section.

## Step 4: Create a Service to inspect endpoints

The exercise asks you to inspect the Service endpoints. Create a Service for this Deployment:

```powershell
kubectl expose deployment nginx-probes --name=nginx-probes-service --type=ClusterIP --port=80
```

Display the Service and endpoints:

```powershell
kubectl get services
kubectl get endpoints nginx-probes-service
```

The ready Pods should be listed as endpoints. With current Kubernetes versions, you can also inspect EndpointSlices:

```powershell
kubectl get endpointslices
```

## Explanation: Readiness vs. liveness

- **Readiness probe:** Checks whether a Pod is ready to serve requests. If readiness fails, Kubernetes excludes that Pod from the Service's ready endpoints until it becomes ready again.
- **Liveness probe:** Checks whether the container is healthy. Repeated liveness failures can cause Kubernetes to restart the container.

A Pod that is not ready should not receive new Service traffic because it may not be able to serve requests correctly.

**Exercise 7 is complete when:** the two-replica Deployment is running, both probes are configured, Pod details are displayed, and Service endpoints are inspected.

---

# Final verification

Run these commands to review the resources created during the lab:

```powershell
kubectl get nodes
kubectl get deployments
kubectl get pods
kubectl get services
kubectl get configmaps
```

For a more detailed view of the Deployments:

```powershell
kubectl describe deployment webapp
kubectl describe deployment webapp-yaml
kubectl describe deployment nginx-probes
```

## Notes

- Replace placeholders such as `POD_NAME` with actual names from `kubectl get pods`.
- Wait for Pods to become ready before checking results.
- `webapp` is scaled and updated in Exercises 3 and 4; `webapp-yaml` is used for the YAML and ConfigMap exercises; `nginx-probes` is used for health probes.
- The exercise does not require a Service for `webapp-yaml`. A Service is created for `nginx-probes` in Exercise 7 specifically so its endpoints can be inspected.
- If a command reports that a resource already exists because you reran a step, inspect the current state with `kubectl get` or `kubectl describe` before deciding whether to delete or recreate anything.
- To stop Minikube after finishing the lab, run:

```powershell
minikube stop
```
