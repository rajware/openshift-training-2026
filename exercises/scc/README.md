# Service Context Constraints

This set of exercises is designed to demonstrate how Security Context Constraints (SCC) govern pod execution in OpenShift. You will walk through three scenarios that illustrate the transition from a "standard" container failure to a secured, successful deployment.

## Scenario 1: The Standard Image Failure

We begin by attempting to deploy the Docker Hub official `nginx:alpine` image. This fails because the image is designed to run as root (UID 0) and bind to port 80, both of which are strictly forbidden by OpenShift’s default restricted-v2 policy.

Deploy using

```
oc apply -f nginx-dep-fail.yaml
```

## Scenario 2: Using pre-hardened images like Red Hat UBI
We then deploy the Red Hat Universal Base Image (UBI) version of Nginx. This succeeds immediately because the image is pre-hardened: it listens on port 8080 and grants the root group (GID 0) the permissions necessary to run under a random, non-root User ID. 

Deploy using

```
oc apply -f nginx-success-1.yaml
```

## Scenario 3: The Exceptions Approach (anyuid)

Finally, we revisit the `nginx:alpine` image. To make it work, we create a dedicated ServiceAccount and grant it the `anyuid` SCC. This demonstrates how to selectively relax security "guardrails" for specific workloads that require root privileges to function.

First, associate the service account with the `anyuid` SCC as a cluster admin. Then, Deploy using

```
oc apply -f nginx-success-2.yaml
```

Reme
