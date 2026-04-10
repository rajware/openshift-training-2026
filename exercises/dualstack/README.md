# Dual Stack Networking in OpenShift

This exercise demonstrates how OpenShift handles IPv4/IPv6 Dual Stack networking. You will learn how to:
1. Verify cluster-wide Dual Stack support.
2. Observe how OpenShift handles IPv6-only services within a dual-stack environment.
3. Enforce strict IP family isolation using NetworkPolicies.

## Prerequisites

To perform cluster-wide verification, you may need **cluster-admin** permissions to view the network configuration. However, the application deployment and testing steps can be performed by any user in their own project.

## Steps

### 1. Cluster Verification

Before starting, verify that your cluster is configured for Dual Stack networking.

```bash
oc get networks.config.openshift.io cluster -o yaml
```

In the output, identify the `clusterNetwork` and `serviceNetwork` fields. A dual-stack cluster will have both an IPv4 and an IPv6 CIDR block listed:

```yaml
status:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  - cidr: fd01::/48
    hostPrefix: 64
  serviceNetwork:
  - 172.30.0.0/16
  - fd02::/112
```

Alternatively, you can check the IPs assigned to any running pod in the cluster:
```bash
oc get pod <pod-name> -o jsonpath='{.status.podIPs}'
```

---

### 2. Deploying the Application

We will deploy a simple **echo-server** and a **client** pod to test connectivity.

```bash
# Deploy the echo-server and its service
oc apply -f echo-server-dep.yaml
oc apply -f echo-server-service.yaml

# Deploy the client pod
oc apply -f client-dep.yaml
```

Wait for both pods to reach the `Running` state:
```bash
oc get pods -l sample=dualstack
```

---

### 3. Verifying Service Connectivity (IPv6 Focus)

Take a look at `echo-server-service.yaml`. You will notice it is configured with `ipFamilyPolicy: SingleStack` and `ipFamilies: [IPv6]`. This means that even in a dual-stack cluster, this specific service is only reachable via IPv6.

Exec into the client pod and attempt to reach the server via the service name:

```bash
# Get the client pod name
CLIENT_POD=$(oc get pod -l layer=client -o name)

# Curl the service
oc exec $CLIENT_POD -- curl -s http://echo-server/echo
```

**Observation:** Look at the output details. You should see that the connection is using IPv6 addresses. 

You can explicitly attempt to force IPv4 to see it fail:
```bash
oc exec $CLIENT_POD -- curl -4 -s --connect-timeout 5 http://echo-server/echo
```
*(This should fail or timeout because the service has no IPv4 address mapping.)*

---

### 4. Direct Pod-to-Pod IPv4 Connectivity

Even though the **Service** is IPv6-only, the **Pods** themselves are Dual Stack by default. They still have IPv4 addresses and can communicate directly using them—as long as no NetworkPolicy is blocking them.

First, identify the direct IPv4 address of the `echo-server` pod:
```bash
ECHO_IPV4=$(oc get pod -l layer=echo-server -o jsonpath='{.status.podIPs[?(@.ip=~"^[0-9].*")].ip}')
echo "Echo Server IPv4: $ECHO_IPV4"
```

Now, try to curl that IPv4 address directly from the client:
```bash
oc exec $CLIENT_POD -- curl -s http://$ECHO_IPV4:8080/echo
```
**Observation:** This works! By default, OpenShift allows multi-family communication between pods.

---

### 5. Enforcing IPv6 with Network Policy

In some security scenarios, you may want to ensure a service *only* communicates over a specific protocol. Let's restrict the `echo-server` so that it **only** accepts IPv6 traffic, effectively blocking any IPv4-based communication.

Apply the provided NetworkPolicy:
```bash
oc apply -f allow-ipv6-only-netpol.yaml
```

This policy selects the `echo-server` and only allows ingress from the `::/0` (all IPv6) CIDR block. Because NetworkPolicy is an "isolated" model, any traffic not explicitly allowed (like IPv4) is dropped.

---

### 6. Verification of Isolation

Now, try the direct IPv4 pod-to-pod connection again:
```bash
oc exec $CLIENT_POD -- curl -s --connect-timeout 5 http://$ECHO_IPV4:8080/echo
```
**Result:** This should now **timeout**. The NetworkPolicy has successfully blocked the IPv4 traffic.

Finally, verify that the IPv6 service connectivity still works:
```bash
oc exec $CLIENT_POD -- curl -s http://echo-server/echo
```
**Result:** This still works perfectly because the traffic is flowing over IPv6, which is permitted by our policy.

---

## Summary

In this exercise, you learned:
*   How to verify Dual Stack configuration at the cluster and pod level.
*   That Services can be constrained to a single IP family (`SingleStack`), independent of the pod's capabilities.
*   That NetworkPolicies are family-aware; by omitting an IPv4 `ipBlock` and allowing an IPv6 one, you can strictly enforce IPv6-only communication.
