# Custom SCC

This exercise demonstrates the need for a custom SCC, and how to fulfil it.

The deployment tcpdump-dep represents an application that requires more
Linux capabilities than the standard SCCs provide (assuming we do not want
to escalate all the way to privileged, which we shouldn't) - in this case,
NET_RAW. To ensure it works, we create a new SCC, network-raw. We start 
from restricted-v2, and add only the permissions that we require.

First, deploy the SCC as a cluster admin using:

```
oc apply -f network-raw-scc.yaml
```

Next, associate the SCC with a service account in the namespace where the
deployment will be applied, as a cluster admin.

```
oc adm policy add-scc-to-user network-raw -z network-raw-sa -n NAMESPACE
```

Note that the service account has not been created yet.

Finally, apply the deployment. You can do this as a regular user.

```
oc apply -f tcpdump-dep.yaml -n NAMESPACE
```

If you want to, you can apply the deployment without creating and applying
the custom SCC. The pod will crash. If you do it in the order specified 
here, the pod will run, and you can check its logs to see tcpdump running.
