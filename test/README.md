## E2E Tests

### How to Run

#### Test e2e in dev mode

```
make minishift-start
eval $(minishift docker-env)
make test-e2e-local
```

## Steps to verify operator registry on OCP4
### Pre-requisites
* have a 48h-ephemeral cluster on AWS for OCP4
* `oc` client installed
> NOTE: you can also do all those steps with the UI

### 1 and 2. create a catalog and subscription
* Create CatalogSource and associate subscription
Here we reuse [operator-registry image](../Dockerfile.registry) built by CI:
```
oc create -f http://operatorhubio-operator-hub.devtools-dev.ext.devshift.net/installopenshift4/devconsole.v0.1.0.yaml
catalogsource.operators.coreos.com/rhd-operatorhub-catalog created
subscription.operators.coreos.com/my-devconsole created
```
This catalog contains a link to the operator-registry image build by CI. 
Here is the content of the remote file:
```yaml
apiVersion: operators.coreos.com/v1alpha1 
kind: CatalogSource
metadata: 
  name: rhd-operatorhub-catalog 
  namespace: openshift-operator-lifecycle-manager 
spec: 
  sourceType: grpc
  image: quay.io/redhat-developer/operator-registry:latest
  displayName: Community Operators
  publisher: RHD Operator Hub 
--- 
apiVersion: operators.coreos.com/v1alpha1 
kind: Subscription 
metadata: 
  name: my-devconsole
  namespace: openshift-operators
spec: 
  channel: alpha
  name: devconsole
  source: rhd-operatorhub-catalog
  sourceNamespace: openshift-operator-lifecycle-manager
```
* login to OCP4 and verify all is well installed:
```
> oc project openshift-operator-lifecycle-manager
> oc get catsrc
NAME                      NAME                  TYPE       PUBLISHER          
olm-operators             OLM Operators         internal   Red Hat
rhd-operatorhub-catalog   Community Operators   grpc       RHD Operator Hub 
> oc get sub
NAME               READY     UP-TO-DATE   AVAILABLE 
catalog-operator   1/1       1            1 
```
You should see your new catalog.

### 3. Create a new Component
1) fom the command line:
```
oc new-project demo
oc apply -f examples/devconsole_v1alpha1_gitsource_cr.yaml
oc apply -f examples/devconsole_v1alpha1_component_cr.yaml
```
You should be able to see the route of your nodejs app.

2) alternatively form UI:
* create a new project
* go to left menu "Installed Operators", 
* see OpenShift Developer Console, hit GitSouce and enter yaml similar to 1)
> NOTE: change the namespace with the new project you created
```yaml
apiVersion: devconsole.openshift.io/v1alpha1
kind: GitSource
metadata:
  name: example-gitsource
  namespace: demo
spec:
  url: "https://github.com/sclorg/nodejs-ex" #"https://github.com/nodeshift-starters/nodejs-rest-http-crud"
  ref: master
  contextDir: /cmd/manager
  httpProxy: http://proxy.example.com
  httpsProxy: https://proxy.example.com
  noProxy: somedomain.com, otherdomain.com
  secretRef:
    name: mysecret
  flavor: github
```
* click create
* see OpenShift Developer Console, hit Component and enter yaml similar to 1)
```yaml
apiVersion: devconsole.openshift.io/v1alpha1
kind: Component
metadata:
  name: myapp
  namespace: demo  
spec:
  buildType: "nodejs"
  gitSourceRef: "example-gitsource"
  port: 8080
  exposed: true
```
## Steps to verify operator registry on minishift

### 1. Install OLM (not required for OpenShift 4)

If you are using OpenShift 3, install OLM with this command:

```
oc create -f https://raw.githubusercontent.com/operator-framework/operator-lifecycle-manager/master/deploy/upstream/quickstart/olm.yaml 
```

> NOTE: Alternately you can use `oc` command instead of `kubectl.`

### 2. Build and push the operator image to a public registry such as quay.io

Note: Instead of a public registry, the registry provided by OpenShift might work.

Checkout the `master` branch of [devconsole-operator](https://github.com/redhat-developer/devconsole-operator)

Then run these commands:

```
$ operator-sdk build quay.io/<username>/devconsole-operator
$ docker login -u <username> -p <password>  quay.io
$ docker push quay.io/<username>/devconsole-operator
```
> NOTE: make your repo public
When running the above command, substitute the `username` and `password` entries appropriately.

### 3. Update the CSV with the operator image location

Open this file
`manifests/devconsole/0.1.0/devconsole-operator.v0.1.0.clusterserviceversion.yaml` and change the image to point to the location pushed in the previous step.

Inside the file look for `image: REPLACE_IMAGE` and specify the image location.

### 4. Build the operator registry image

Now you are going to build the operator image using `Dockerfile.registry`

```
docker build -f Dockerfile.registry . -t quay.io/<username>/operator-registry:0.1.0 \
	--build-arg image=quay.io/<username>/devconsole-operator --build-arg version=0.1.0
docker push quay.io/<username>/operator-registry:0.1.0
```

When running the above command, substitute the `username` with your quay.io username.

### 5. Create CatalogSource and Subscription

Use this template to create a YAML file, say `cat-sub.yaml`:

```
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: my-catalog
  namespace: olm
spec:
  sourceType: grpc
  image: quay.io/<username>/operator-registry:0.1.0
  displayName: Community Operators
  publisher: Red Hat
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: my-devconsole
  namespace: operators
spec:
  channel: alpha
  name: devconsole
  source: my-catalog
  sourceNamespace: olm
```

Before applying the above file, point to the newly created operator registry image (substitute the `username` with your quay.io username).

Example:

```
oc apply -f cat-sub.yaml
```

### 6. Verify gitsources CRD presence

Check for the existence of a Custom Resource Definitions with the name as `gitsources.devconsole.openshift.io`

Run this command to list CRDs:

```
oc get crds
```
