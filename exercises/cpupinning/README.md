# CPU Manager and CPU Pinning

This exercise demonstrates how to dedicate entire CPU cores exclusively to specific pods in OpenShift. This is critical for latency-sensitive applications like High-Performance Computing (HPC), telecommunications/5G, or real-time trading platforms where CPU context switching would degrade performance.

By default, the CPU Manager policy is set to `none`, meaning CPU resources are shared fairly among all workloads. Here, we configure OpenShift to use the `static` policy on a subset of worker nodes.

## Prerequisites

Before starting, identify a worker node in your cluster that you want to dedicate to CPU pinning. Look at the available nodes:

```bash
oc get nodes
```

and select one or more worker nodes for this exercise. Let's say we select `<NODE_NAME>` for this exercise.

## Scenario: Enabling CPU Pinning

### 1. Create a custom MachineConfigPool

First, we create a new `MachineConfigPool` (MCP) for the nodes that will participate in CPU pinning. We do this rather than applying it to *all* workers because enabling CPU Manager requires altering the kubelet config and rebooting the affected nodes. 

Apply the MachineConfigPool:

```bash
oc apply -f worker-cpumanaged-mcp.yaml
```

### 2. Label the Node

Label your chosen worker node so it becomes a part of our new pool. Replace `<NODE_NAME>` with your worker node's actual name.

```bash
oc label node <NODE_NAME> cpumanager=true
```

You can verify the node has successfully joined the new pool by checking the MachineConfigPools:
```bash
oc get mcp
```
*(Wait until the `worker-cpumanaged` pool shows 1 machine count).*

### 3. Deploy the KubeletConfig

Next, we deploy the `KubeletConfig` that configures the `static` CPU Manager policy specifically targeting our new user-defined MachineConfigPool.

```bash
oc apply -f cpumanager-enabled-kubeletconfig.yaml
```

**Note:** Applying this configuration will instruct the Machine Config Operator to render a new configuration for the target nodes. The node will drain its pods and **reboot**. You must wait for the node to return to a `Ready` state and for the `worker-cpumanaged` MCP to finish `UPDATING` before proceeding.

### 4. Deploy the Application (Guaranteed QoS)

Finally, we deploy the example application. For a Pod to gain exclusive CPU pinning under the `static` policy, it must strictly satisfy the following criteria:

1. Use a **Guaranteed** Quality of Service (QoS) class. This means its CPU and Memory `requests` must exactly match its `limits`.
2. Request an **integer** number of CPU cores (e.g., `1`, `2` — never fractional like `500m`).

Deploy the application:

```bash
oc apply -f cpu-pin-demo-dep.yaml
```

You can verify that the pod has been successfully scheduled on the target node. Because it meets the conditions above and lands on an enabled node, its process will be pinned strictly to an exclusive CPU core.

### 5. Verify CPU Pinning

We can definitively prove that the pod has been granted exclusive access to a CPU core by checking its **CPU affinity mask** using the `taskset` command. 

Run the following command to check the affinity of the main process (PID 1) running inside our pod:

```bash
oc exec deployment/cpu-pin-demo -- taskset -pc 1
```

**Expected Output:**
If CPU pinning is working correctly, the output will show exactly **one** CPU core assigned to the process. For example:

> `pid 1's current affinity list: 3`

*(This means the pod has been granted exclusive, uninterrupted access to CPU core #3).*

If CPU pinning were **not** active (the default OpenShift behavior), the pod would instead be allowed to share all available cores on the node, and the output would look like this:

> `pid 1's current affinity list: 0-7`
