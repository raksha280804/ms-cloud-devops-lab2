# Lab 2 — hello-app on Kubernetes

Your copy of this repository is your submission. The lab sheet is on the course site; this file is where you record what you did.

What's here:

| File | What it is |
|---|---|
| `src/`, `index.html` | a small React page that shows which Pod answered |
| `nginx.conf` | serves the page, and answers `/whoami` with the Pod name |
| `Dockerfile` | builds the page, then serves it with nginx on port 80 |

You don't change any of these. You add `hello-app.yaml` and fill in the sections below.

## Before you push, check

- [x] `hello-app.yaml` is in the root of this repository
- [x] `screenshot.png` is in the root of this repository
- [x] every section below is filled in

---

## 1. My image

Docker Hub image I deployed (for example `ravi123/hello-app:v1`):

```
shree1290/hello-app:v1
```

## 2. Pods across nodes

Output of `kubectl get pods -o wide` after you applied `hello-app.yaml`:

```
NAME                        READY   STATUS    RESTARTS   AGE     IP           NODE              NOMINATED NODE   READINESS GATES
hello-app-5bf857766-cff86   1/1     Running   0          6m33s   10.244.1.8   desktop-worker2   <none>           <none>
hello-app-5bf857766-j6wnj   1/1     Running   0          5m56s   10.244.2.6   desktop-worker    <none>           <none>
hello-app-5bf857766-vjfrw   1/1     Running   0          6m33s   10.244.2.5   desktop-worker    <none>           <none>
hello-app-5bf857766-xrcpg   1/1     Running   0          6m33s   10.244.1.9   desktop-worker2   <none>           <none>
```

## 3. The Service and its endpoints

Output of `kubectl get service hello-app` and `kubectl get endpointslices -l kubernetes.io/service-name=hello-app`:

```
NAME        TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
hello-app   LoadBalancer   10.96.6.124   172.18.0.7    8080:31597/TCP   3m46s
NAME              ADDRESSTYPE   PORTS   ENDPOINTS                                      AGE
hello-app-zgt77   IPv4          80      10.244.2.6,10.244.1.8,10.244.2.5 + 1 more...   3m46s
```

## 4. Browser

Add `screenshot.png`: the page at `http://localhost:8080` with at least two Pods in the "Answers so far" table.

## 5. The four traps

For each trap, write what you saw and why it happened, in two or three lines of your own words.

**Trap 1 — the local image.**
 I built hello-app:v1 locally and created a Deployment with it before pushing to Docker Hub. I expected ImagePullBackOff, but the pods started normally and kubectl describe showed "Successfully pulled image hello-app:v1". On Docker Desktop, the kind nodes seem to share my laptop's image store, so the local image was found. On a normal cluster each node has its own image storage and would try to pull hello-app:v1 from Docker Hub, where it doesn't exist, so the image has to be pushed to a registry. I pushed it as shree1290/hello-app:v1 and used that name from then on.

**Trap 2 — the selector typo.**
 With the Service selector set to helo-app, the EndpointSlice showed PORTS <unset> and ENDPOINTS <unset>. The Service and the pods still existed, but no pod had the label app: helo-app, so the Service had no pods to send traffic to. Fixing the selector back to hello-app and applying again brought the endpoints back.

**Trap 3 — the wrong targetPort.**
 With targetPort set to 8080, the EndpointSlice listed my pod IPs but with PORTS 8080. The selector was right, so the Service found the pods, but it sent traffic to port 8080, where nothing listens because nginx is on port 80. Changing targetPort back to 80 and applying fixed it.


**Trap 4 — the EXTERNAL-IP address.**
 The EXTERNAL-IP first showed <pending> and then 172.18.0.7. That address is inside Docker Desktop's private network, so it can't be opened from my browser. Docker Desktop publishes the LoadBalancer port on localhost, so http://localhost:8080 works instead. On a cloud provider the EXTERNAL-IP would be a real public address.

## 6. Your YAML vs the generated YAML

Name two fields in the `--dry-run=client -o yaml` output that were not in your file. Why does your file still work without them?

Two fields in the generated YAML that were not in my file are `strategy: {}` and `resources: {}` (the generated output also had `status: {}`). My file still works without them because they are empty or default values: Kubernetes fills in a default rolling-update strategy and applies no resource limits when they are left out, and `status` is filled in by the cluster itself.
