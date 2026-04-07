# Operator

This exercise takes a simple web application, defined as a Kubernetes manifest, and turns it into an operator over a few steps. First, we add additional manifests to the web application to make it OpenShift-friendly. Then, we create a Helm chart from the manifests. Finally, we use the Helm chart to create an operator that can be used to deploy and manage the web application.

## Directory structure

The exercise is organised using the following subdirectories:

- original-manifests: The original Kubernetes manifests for the web application.
- openshift-manifests: The OpenShift-friendly manifests for the web application.
- helm-chart: The Helm chart for the web application.
- operator: The operator created from the helm chart, and further refined for packaging and deployment.

## Tool prereuisites

This exercise requires, in addition to oc, the following tools:

* helm
* The Operator SDK CLI
* make
* docker or podman
* kubectl
* olm

The Operator SDK CLI is currently supported only on MacOS or Linux. On Windows, you can use WSL2 to run the CLI.

## Steps

### 1. Preparing the Permissions

Before scaffolding our operator, it's important to remember that our underlying MySQL database expects to run as root to initialize itself. In OpenShift, this requires granting the `anyuid` Security Context Constraint (SCC) to the ServiceAccount our application uses. 

```bash
oc apply -f openshift-manifests/db-sa.yaml
oc adm policy add-scc-to-user anyuid -z db-sa
```

### 2. Initializing the Operator

We use the Operator SDK tooling to automatically scaffold a Helm-based operator. This process takes our existing Helm chart and wraps it in a controller that monitors the cluster for Custom Resources (CRs).

First, navigate into the empty `operator` directory:

```bash
cd operator
```

Next, run the `operator-sdk init` command to map the Helm chart to a new Custom Resource Definition (CRD):

```bash
operator-sdk init --plugins=helm \
  --helm-chart=../helm-chart/dbapp/ \
  --domain=rwsl.in \
  --group=apps \
  --version=v1beta1 \
  --kind=DbApp
```

**What this command does:**
*   `--plugins=helm`: Instructs the SDK to scaffold an operator that runs Helm templates under the hood (as opposed to Go or Ansible code).
*   `--helm-chart`: Points to the local path of your Helm chart. *Note: The path `../helm-chart/dbapp/` is used here because we changed directories into the `operator` folder.* 
*   `--domain`, `--group`, and `--version`: These mathematically combine to define the fully-qualified API Schema (e.g., `apps.rwsl.in/v1beta1`) for your new custom object.
*   `--kind`: Defines the actual name of the Custom Resource (e.g., `DbApp`) that users will instantiate to deploy the application.

### 3. Refining the Scaffolded Operator

While the SDK automatically analyzes your Helm chart to guess what the operator needs, it requires a little polishing. We manually updated two files to correct generation gaps:

**1. RBAC Permissions (`config/rbac/role.yaml`)**
To deploy your application, your operator's "manager" pod needs permission to create all the sub-objects defined in your Helm charts. The SDK generator successfully detected pods, deployments, and PVCs, but it missed a few. We manually added explicit rules for:
*   `serviceaccounts` (Core API Group)
*   `ingresses` (`networking.k8s.io` API Group)

Without these additions, our Operator would throw a `Forbidden` error when it tried to deploy the web app's ingress or the database's service account.

**2. Custom Resource Definition (`config/crd/bases/apps.rwsl.in_dbapps.yaml`)**
The standard SDK output provides a barebones schema that accepts almost anything. We fleshed out the OpenAPI v3 schema specifically for our `DbApp` kind so we can validate it before the cluster accepts it:
*   **Validation:** We strictly typed properties (like `.spec.host`), marked them as required, and even added regex patterns (like `^[0-9]+Gi$` for `.spec.db.storageSize`).
*   **Print Columns:** We added `additionalPrinterColumns` to the CRD. Now, when a user types `oc get dbapps`, the Kubernetes CLI will display clean, helpful columns like `HOST`, `DEPLOYED`, and `ERROR` instead of just a generic name and age!

### 4. Preparing for OperatorHub (OLM)

To distribute our Operator on OperatorHub, we need to package it for the Operator Lifecycle Manager (OLM). This begins by configuring build defaults and auto-generating a ClusterServiceVersion (CSV) document.

**1. Configuring the `Makefile` defaults**
By default, the SDK leaves placeholder names for container registries and versions. To make repeatedly building containers seamless and less error-prone, we edited the top of `operator/Makefile` to establish standard tags:
* `VERSION ?= 0.1.0`
* `IMAGE_TAG_BASE ?= quay.io/rajware/dbapp-operator`
* `IMG ?= $(IMAGE_TAG_BASE):v$(VERSION)`

With these solid defaults, we no longer need to constantly pass `IMG=...` or `VERSION=...` flags on the command line.

**2. Generating the Bundle**
With the Makefile updated, we ran the bundle generation command:
```bash
make bundle
```

This scraped our RBAC rules, updated CRDs, and Helm charts to automatically generate the core OLM metadata file: `config/manifests/bases/operator.clusterserviceversion.yaml`. This CSV provides the blueprint describing how to deploy the operator and serves as the visual representation of your operator within the OpenShift Web Console.

**3. Customizing the CSV Display**
Before finalizing the bundle, we manually edited `config/manifests/bases/operator.clusterserviceversion.yaml` to ensure the Operator looks professional in the OpenShift OperatorHub UI. The key additions were:
*   `icon`: A base64-encoded SVG logo.
*   `description`: A markdown-formatted explanation of the Operator's capabilities.
*   `alm-examples`: A valid JSON snippet representing a default `DbApp` Custom Resource, so users get a pre-filled template instead of a blank screen when they click "Create Instance". 
*   `customresourcedefinitions`: Human-readable display names for our internal API resources.

Finally, we ran `make bundle` one more time. This takes our customized base file and recompiles the final, immutable bundle manifests stored automatically in the `/bundle` directory!

### 5. Building the OLM Containers 

**1. Build the Core Controller**
Run the following to build and push the operator's controller container image to the registry:

```bash
make docker-build docker-push
```

**2. Build the Bundle Container**
Bundle images are empty `scratch` containers storing just YAML manifests:

```bash
make bundle-build bundle-push
```

**3. Build the Catalog Index Container**
Finally, Catalog images are the basis for containers that run the `opm` server binary to expose your bundle (or multiple bundles) via gRPC. 

```bash
make catalog-build catalog-push
```

With these 3 images pushed, the Operator is ready for distribution and installation on your OpenShift cluster!

### 6. Registering the Catalog in OpenShift

To make the Operator actually show up in the **OperatorHub** UI in the OpenShift Web Console, we need to register our custom catalog index as a `CatalogSource`.

**1. Create the `CatalogSource` YAML:**
In the `operator/deploy` directory, create a file named `catalog-source.yaml` (create the directory if not present):

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: dbapp-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/rajware/dbapp-operator-catalog:v0.1.0
  displayName: Rajware Training Catalog
  publisher: Rajware
```

**2. Apply the configuration:**
As a cluster admin, run:

```bash
oc apply -f deploy/catalog-source.yaml
```

**3. Verify in the Web Console:**
Once applied, students can navigate to **Operators -> OperatorHub** and search for "DbApp". Your operator will appear with its custom icon, provider name, and markdown description! From there, they can click "Install" to subscribe and begin creating `DbApp` instances.

### 7. Installing the Operator via `oc` (CLI)

While the Web Console is convenient, it's often better to automate the installation using the command line. Since our operator supports the `AllNamespaces` install mode, we can use the standard OpenShift **`openshift-operators`** namespace. This simplifies the process because the namespace and a global `OperatorGroup` are already present by default.

**1. Create the installation manifest:**
In the `operator/deploy` directory, we have created the following file:

*   **`subscription.yaml`**: The request to OLM to install our specific package from the catalog into the global operator pool.

**2. Execute the installation:**
Run the following command as a cluster admin:

```bash
# Create the Subscription in the standard operators namespace
oc apply -f deploy/subscription.yaml
```

**3. Verify the installation:**
You can monitor the status of the installation by checking the `ClusterServiceVersion` (CSV) in the `openshift-operators` namespace:

```bash
oc get csv -n openshift-operators
```

Once the `PHASE` column says `Succeeded`, the operator is live! You can now begin creating `DbApp` custom resources in any namespace.
