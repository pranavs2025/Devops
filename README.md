ISTIO AMBIENT MODE

1. Introduction 
Istio is moving towards an ambient mode — with a sidecar-less model. Finally, we will be able to ditch the sidecar which was consuming a lot of our in terms of cpu & memory.

In the normal mode of Istio, we recall that every single application pod was getting injected with the envoy proxy. So,here envoy proxy is nothing but, it is actually a bus communication. Like, instead of service A - service B connection, it will have something like service-A to envoy - envoy to service-B.
However, in this new mode, application pods will be untouched :) and they will only be having their own application containers.

2. SideCar vs Ambient MODE
SideCar pic

This is the traditional sidecar model where each service pod contains the application container coupled with the envoy sidecar. All traffic to and from from the application is intercepted by the sidecar.

Ambient Node Pic

In this new ambient mode, the application pods are standalone pods with no sidecars. However, in this case, each kubernetes node in the cluster will be running a daemonset pod — the mighty ztunnel (zero-trust tunnel). All traffic to and from pods in the node is intercepted by the ztunnel. Ztunnel is a L4 per-node proxy.

Ambient Mode Pic wiht ztunnel + waypoint

Ztunnel is sufficient for networking between workloads which only have a need for L4 proxy. For L7 requirements, like http header based routing, L7 authorization, we deploy a workload called waypoint proxy, which are envoy pods, on a per application basis. They may run in the same or different nodes.

In this case, traffic originating from source ztunnel will hit waypoint proxy and waypoint will then forward it to the destination ztunnel.

Hence, if an app has no requirement for L7 processing, we can ditch the waypoint and only work with ztunnel. In the earlier mode, we were mandated to use envoy sidecars even if we only had L4 requirements.


Z-Tunnel (Zero-Trust tunnel)
It is a lightweight L4 proxy that runs on every node.

One ztunnel per node (DaemonSet)

What ztunnel actually does

Think of it as a network gatekeeper.

It handles:

1. Traffic interception
All pod traffic is redirected to ztunnel using iptables
2. mTLS (security)
Encrypts traffic automatically between services
3. Identity
Knows:
who is calling
who is receiving
4. L4 routing (TCP level)
Sends traffic to correct destination IP





What ztunnel CANNOT do
No HTTP header routing
No path-based routing (/api, /login)
No advanced policies

Because it's only Layer 4 (TCP)
mportant: HBONE

ztunnel uses a protocol called HBONE:

encrypted tunnel
built on HTTP/2
secure by default





What is Waypoint Proxy

Waypoint = Envoy proxy used only when needed

It is a full L7 proxy.

Where it runs
Namespace / App level:
  Waypoint Pod (Envoy)

Not per pod
Not per node
Per application or namespace

What waypoint does

Handles advanced traffic logic

1. HTTP routing

Example:

/api → service A
/login → service B
2. Header-based routing
user=premium → special service
3. Authorization policies
who can access what
4. Observability (detailed)
request-level metrics


Traffic flow with waypoint
App → ztunnel → Waypoint → ztunnel → App





Our AZ + DB Example (REAL UNDERSTANDING)
Your requirement:
App (AZ-1) → DB-1
App (AZ-2) → DB-2
 How ztunnel helps
App (AZ-1)
  ↓
ztunnel (Node AZ-1)
  ↓
ztunnel (Node AZ-1 or AZ-2)
  ↓
DB

 ztunnel ensures:

secure traffic
efficient forwarding

BUT:

 It does NOT decide:

which DB (AZ-1 or AZ-2)
 Who decides DB?

 Istio routing config (DestinationRule)

 So in your case:
You ONLY need ztunnel

Because:

DB traffic is TCP (no HTTP routing needed)
No headers involved.




OUR CASE

Got it — let’s make this **clean, structured, and easy to remember**. No clutter.

---

#  Locality Routing (Your Use Case — Clean Explanation)

##  Your Goal

* App in AZ-1 → DB-1 (same AZ)
* App in AZ-2 → DB-2 (same AZ)
* Avoid cross-AZ traffic (cost + latency)

---

# Core Idea (1 line)

**Istio prefers sending traffic to endpoints in the same AZ using locality routing.**

---

#  How it Works (4 simple steps)

### 1. Kubernetes knows AZ

Each node has a label:

```text
topology.kubernetes.io/zone=az1 / az2
```

---

### 2. Pods inherit AZ

* App pod → runs on node → AZ known
* DB pod → runs on node → AZ known

---

### 3. Service has multiple endpoints

```text
db-service:
  → DB-1 (AZ-1)
  → DB-2 (AZ-2)
```

---

### 4. Istio makes decision

With locality enabled:

```text
If request from AZ-1 → choose DB-1  
If request from AZ-2 → choose DB-2  
If local DB fails → fallback to other AZ
```

---

#  Full Flow (Simple)

### Case: App in AZ-1

```text
App (AZ-1)
  ↓
ztunnel
  ↓
Istio decides → DB-1
  ↓
ztunnel → DB-1
```

Same AZ

---

#Important Truth
 **ztunnel does NOT choose DB**
**Istio chooses DB based on AZ**

---

# Minimal Configuration (Clean YAML)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: db-locality
spec:
  host: db-service
  trafficPolicy:
    loadBalancer:
      localityLbSetting:
        enabled: true
```


localityLbSetting:
  enabled: true
  failover:
  - from: az1
    to: az2
  - from: az2
    to: az1

Istio = decides WHERE to send  
ztunnel = sends traffic  
