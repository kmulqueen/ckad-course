# Security

[[TOC]]

## Security Primitives

1st line of defense is controlling access to the `kube-apiserver` itself.

2 decisions:

- Who can access the cluster?
  - _Authentication_
- What can they do?
  - _Authorization_

### Authentication

Different users that may be accessing the cluster:

- Admins
- Developers
- Bots

#### Accounts

- User Account
- Service Account

K8s doesn't manage User accounts natively. It relies on an external source ie. file w/ user details, certificates, 3rd party identity service (LDAP) to manage users. You can't create users in a cluster.

All user access is managed through the `kube-apiserver`. It

1. _authenticates_ each request before
2. processing the request

Authentication Mechanisms

- Static Password File: File w/ list of usernames & passwords
- Static Token File: File w/ list of usernames & tokens
  - To authenticate using these basic creds while accessing the apiserver, use a `curl` command like `curl -v -k https://master-node-ip:6443/api/v1/pods -u "user1:password123"`. These mechanisms (password/token files) are NOT recommended.
- Certificates
- Identity Services (LDAP)

K8s can create/manage Service accounts using the k8s API.

## KubeConfig

The Config file found usually at `$HOME/.kube/config` holds information regarding which users are able to access which cluster. These relationships are built using _contexts_ in which a user from the `users` property of the config file and a cluster from the `clusters` property are paired together in the `contexts` property of the config file. Here is the minikube example. Use the command `kubectl config view` to see the file contents.

```
kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /Users/<user-name>/.minikube/ca.crt
    extensions:
    - extension:
        last-update: Sun, 28 Jul 2024 21:23:36 PDT
        provider: minikube.sigs.k8s.io
        version: v1.33.1
      name: cluster_info
    server: https://127.0.0.1:49380
  name: minikube
contexts:
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Sun, 28 Jul 2024 21:23:36 PDT
        provider: minikube.sigs.k8s.io
        version: v1.33.1
      name: context_info
    namespace: default
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: /Users/<user-name>/.minikube/profiles/minikube/client.crt
    client-key: /Users/<user-name>/.minikube/profiles/minikube/client.key

```

To specify a different kubeconfig file, you can use the comand `kubectl config --kubeconfig=/path/to/my-kube-config use-context <context-name>`. If you don't want to have to specify the kubeconfig file option on each command, set the my-kube-config file as the default kubeconfig by overwriting the content of ~/.kube/config with the content of the my-kube-config file.

### Conexts

To set a default context to use, add the property `current-context: <context-name>` to the config file. To specify a specific context to use when using `kubectl`, use the command `kubectl config use-context <context-name>`.

### Config commands

Use `kubectl config -h` to list all available commands.

## API Groups

Focus in this section was about the `/version` and `/api` paths. All resources (see below in Named Groups) in k8s are grouped into different API groups. The k8s API is grouped into multiple groups based on their purpose, including:

- `/logs`
  - Used for integrating with 3rd party logging applications.
- `/api`
  - Responsible for the cluster functionality.
  - _core_ group
- `/apis`
  - Responsible for the cluster functionality.
  - _named_ group
- `/version`
  - For viewing the version of the cluster.
- `/healthz`
  - For monitoring the health of the cluster.
- `/metrics`
  - For monitoring the health of the cluster.

### Core Groups

`/api` Where all core functionality exists. Here are a few examples:

- `/v1`
  - namespaces
  - pods
  - replication controllers
  - events
  - endpoints
  - nodes
  - bindings
  - persistent volumes
  - persistent volume claims
  - configmaps
  - secrets
  - services

### Named Groups

`/apis` More organized than the core group. Going forward all the new features will be available through these named groups. Here are a few examples (parent levels are _groups_, and the children are _resources_):

- `/apps` (group)
  - `/v1`
    - `/deployments` (resource)
    - `/statefulsets`
    - `/replicasets`
- `/extensions`
- `/networking.k8s.io`
  - `/v1`
    - `/networkpolicies`
- `/storage.k8s.io`
- `/authentication.k8s.io`
- `/certificates.k8s.io`
  - `/v1`
    - `/certificatesigningrequests`

Each resource has a set of actions known as _verbs_ associated with them. For example: list, get, create, delete, update, watch.

## Authorization

Once a user has access to a resource/cluster, what can they do? That's what _authorization_ defines.

### Authorization Modes

Can be set using the `--authorization-mode` option on the kubeapi-server. Defaults to `AlwaysAllow` (see below). Can also provide a comma separated list to use multiple modes ex: `--authorization-mode=Node,RBAC,Webhook`

Identify authorization modes in the `kube-apiserver` by running:
`kubectl describe pod kube-apiserver-controlplane -n kube-system` and then find it under the `Containers.kube-apiserver.Command` properties.

- Node Authorization
- ABAC (Attribute Based Access Control)

  - User/User Group permissions. ex: Dev-Users can view/create/delete pods
  - Difficult to manage because everytime you change this policy file you have to restart the kube-api server.
  - Example policy for Dev User group authorization

    ```
    {
      "kind": "Policy",
      "spec": {
        "user": "dev-user",
        "namespace": "*",
        "resource": "pods",
        "apiGroup": "*"
      }
    }
    ```

- RBAC (Role Based Access Control)
  - Instead of associating a user/goup w/ a set of permissions, we define a role with the permissions and then associate all the appropriate users to that role. ex: Developer-role.
  - Now we only have to modify the role to manage all associated users rather than each user/group like in ABAC.
- Webhook
  - Outsource authorization mechanisms. ex: Open Policy Agent.
- Always Allow
  - Allows all requests w/o any authorization checks
- Always Deny
  - Denies all requests

### RBAC

Roles & RoleBindings fall under the scope of namespaces. If you want to specify the namespace, add it to the `metadata` property

#### Create a Role

role-definition.yaml

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list", "get", "create", "update", "delete"]
  - apiGroups: [""]
    resources: ["ConfigMap"]
    verbs: ["create"]
```

#### Linking a user to a role

Create another object `RoleBinding`.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
  - kind: User
    name: dev-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

### Check Access

To check if you can run a certain verb on a specific resource, you can run `kubectl auth can-i <verb resource>`. For example: `kubectl auth can-i get deployments`

Admins can check access for other users with the `--as` option. Example:
`kubectl auth can-i get deployments --as dev-user`

Check by namespace as well:
`kubectl auth can-i get deployments --as dev-user --namespace test`

### Resource Names

Limit which resources in particular are affected by the role. For example, limiting to only "blue" and "orange" pods.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list", "get", "create", "update", "delete"]
    resourceNames: ["blue", "orange"]
  - apiGroups: [""]
    resources: ["ConfigMap"]
    verbs: ["create"]
```

## Cluster Roles

Authorize users to cluster scoped resources. Cluster scoped instead of namespace scoped, meaning that these are not tied to any namespace. ** This is not a hard rule ** You can create cluster roles for _namespace scoped_ resources as well which will give the user access to the resources across _all_ namespaces.

Other cluster scoped resources include:

- Nodes
- Persistent Volumes
- Cluster Roles
- Cluster Role Bindings
- Namespaces
- Certificate Signing Requests

\*\* To see a full list run `kubectl api-resources --namespaced=true|false`

Example cluster role for an admin to view/create/delete nodes:

cluster-role-definition.yaml

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "get", "create", "delete"]
```

cluster-role-binding-definition.yaml

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

## Admission Controllers

Help implement better security measures to enforce how a cluster is used. Can validate the configuration, change the request itself, or perform additional operations before the request is made.

Example admission controllers that come w/ k8s:

- AlwaysPullImages
  - Ensures that everytime a pod is created, the images are always pulled
- DefaultStorageClass
  - Observes the creation of PVCs and automatically adds a default storage class to them if not specified.
- EventRateLimit
  - Can help set a limit on requests that the apiserver can handle at a time
- NamespaceExists (_deprecated in favor of NamespaceLifecycle_)
  - Rejects requests to namespaces that don't exist
- NamespaceAutoProvision (_deprecated in favor of NamespaceLifecycle_)
  - Creates a namespace from the one provided in the request if it doesn't exist
- NamespaceLifecycle
  - Makes sure that requests to a non-existent namespace are rejected and that the default namespaces (default, kube-system, kube-public) cannot be deleted

### View Enabled Admission Controllers

To see default enabled admission controllers run
`kube-apiserver -h | grep enable-admission-plugins`

### Enable Admission Controllers

To add a new admission controller, update the `kube-apiserver.service` and add to the comma separated list under the `--enable-admission-plugins` option.

### Disable Admission Controllers

Similarly, you can disable certain admission controllers by adding the `--disable-admission-plugins` flag.

## Validating and Mutating Admission Controllers

Different types of admission controllers including:

- Validating
  - Example: NamespaceLifecycle
  - Approve/Reject requests based on some matching/failing criteria
- Mutating
  - Example: DefaultStorageClass
  - Modifies/mutates the request before the resource is created. In the case of DefaultStorageClass, if no storage class is defined in your PVC definition file, then this admission controller adds a default one for you _before_ creating the resource. If you inspect the resource after it is created, you will see a default stroage class there even if you left it out of your definition file.

Some admission controllers can do both validation and mutation. Generally, mutation admission controllers are processed first and then validation admission controllers. This is so that any change made in the mutation can be considered during the validation.

## External Admission Controllers

These are admission controllers with our own logic and validation. To support these, k8s gives us 2 special admission controllers:

- MutatingAdmissionWebhook
- ValidatingAdmissionWebhook

We can configure these webhooks to point to a server either inside the k8s cluster or outside of it. Our server will have it's own admission webhook service running with our own code and logic. Requests coming to our server will have an `AdmissionReview` JSON object that tells who is trying to make what request, and other information. The server responds with an `AdmissionReview` JSON object on wether or not the request is allowed. These are the steps to set this process up:

1. Deploy our own webhook server.

- Contains the logic to permit/reject a request.
- Must be able to receive and respond with the appropriate JSON responses that the webhook expects.

2. Host our webhook server.

- Can containerize it and run it in our k8s cluster somewhere as a deployment.
  - If deployed as a deployment in our k8s cluster, it needs a service to be accessible. ie. webhook-service

3. Configure our cluster to reach out to service and validate/mutate the request.

- We need to create a `ValidatingWebhookConfiguration` or `MutatingWebhookConfiguration` object (depending on wether you need to validate/mutate the request)
- In the webhook configuration file, the `clientConfig` property of the webhook in the `webhooks` section is where we configure the location of our admission webhook server.
  - If we deployed this server on our own and it is not a part of a k8s deployment on our cluster, then the value would be a URL of the server (see below examples).
  - If it is a deployment on our k8s cluster then we can use the service configuration and provide the namesapce and name of the service.
  - The communication between the api server and the webhook server has to be over TLS, so a certificate bundle `caBundle` has to be configured.
    - Server has to be configured with a pair of certificates, then a caBundle needs to be created and passed into the `clientConfig` as a `caBundle`

`rules` control when to call our API server for validation/mutation. In the following examples, our API server would be called anytime a pod is to be created, and then do some validation and either permit/reject the request.

Example webhook configuration objects:

Webhook Server IS NOT part of k8s deployment:

```
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "pod-policy.example.com"
webhooks:
  - name: "pod-policy.example.com"
    clientConfig:
      url: "https://external-server.example.com"
    rules:
      - apiGroups: [""]
        apiVersions: [""]
        operations: ["CREATE"]
        resources: ["pods"]
        scope: "Namespaced"
```

Webhook Server IS part of k8s deployment:

```
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "pod-policy.example.com"
webhooks:
  - name: "pod-policy.example.com"
    clientConfig:
      service:
        namespace: "webhook-namespace"
        name: "webhook-service"
      caBundle: "someCrAzYC3r71F1Cat3StriNGBRO"
    rules:
      - apiGroups: [""]
        apiVersions: [""]
        operations: ["CREATE"]
        resources: ["pods"]
        scope: "Namespaced"
```

## API Versions

- `v1` - means that group is a Generally Available Stable version (GA/Stable).
- `v1alpha1` - Becomes available to the k8s release for the very first time. Not enabled by default. No gaurantee that the API will be available in future releases. Recommended audience is expert users who are interested in giving early feedback.
- `v1beta1` - Has end to end tests and is enabled by default. Committed to completing the feature and may be moving to GA/Stable.

### Multiple Versions

Some API groups have multiple different API versions. When this is the case, only one of the API versions is the preferred/storage version. This comes into play when running `kubectl` commands. The preferred version will be used when running `kubectl` commands, ie. `kubectl get deployment` will run the _preferred_ version of the /apps API. To get the preferred version that is being used in your k8s environment, run `kubectl explain <resource>`, something like `kubectl explain deployment`.

The _storage_ version is the version of the object that is stored in etcd, respective of the version found in the yaml file you used to create the object. To view the storage version, you have to query the object in the etcd database itself, there is no API endpoint to get this.

## API Versions Deprecation

Rules:

1. API elements may only be removed by incrementing the version of the API group.

   - ie moving from `/v1alpha1 -> /v1alpha2`.

2. API objects must be able to round-trip between API versions in a given release without information loss, with the exception of whole REST resources that don't exist in some versions.

   - If there was a field in the `spec` that was present in `/v1alpha1`, and then a new field was introduced _on top of the existing one_ in `/v1alpha2`, and then the release went back to `/v1alpha1`, that new field that was introduced would become a part of `/v1alpha1`.

3. An API version in a given track may not be deprecated until a new API version at least as stable is released.
4. a. Other than the most recent API versions in each track, older API versions must be supported after their announced deprecation for a duration of no less than:

   - GA: 12 months or 3 releases (whichever is longer)
   - Beta: 9 months or 3 releases (whichever is longer)
   - Alpha: 0 releases.

   b. The _preferred_ API version and the _storage version_ for a given group may not advance until after a release has been made that supports both the new verison and the previous version.

   - GA versions can deprecate beta & alpha versions, but not the other way around. ie. `/v2alpha1` cannot deprecate `/v1`

### Kubectl Convert Command

When k8s clusters are upgraded, we have new APIs being added and some being deprecated. When an old API is removed, it is important that you update your manifest files to represent the new version. You can use the `kubectl convert` command to help with this. It may not be available by default on your system, so you may need to install it because it's a separate plugin.

Usage:

- `kubectl convert -f <old-file> --output-version <new-api>`

## Custom Resource Definitions (CRD)

A _Resource_, a `deployment` for example, is stored in `etcd` where we can interact with it and check the status. ie. `get deployments`, `delete deployments`, etc. A _Controller_ is a process that runs in the background and monitors `etcd`. It is responsible for listening for changes to the resource and updating the status of that resource as necessary, ie. creating new deployments, listing deployments, deleting deployments, etc.

You can build custom resources and custom controllers to create and manage custom resource types that you want. To do this you need to construct a _Custom Resource Definition (CRD)_.
Example CRD `flightticket-custom-definition.yaml`:

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: flighttickets.flights.com
spec:
  scope: Namespaced
  group: flights.com # -> This is the API group that you would provide in the apiVersion. Example flights.com/v1
  names: # -> Name of the resource
    kind: FlightTicket
    singular: flightticket
    plural: flighttickets
    shortNames:
      - ft
  versions:
    - name: v1
      served: true
      storage: true # -> Wether or not this is the storage version
      schema: # -> Defines all the parameters that can be specified under the spec section
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                from:
                  type: string
                to:
                  type: string
                number:
                  type: integer
                  minimum: 1
                  maximum: 10
```

To create this custom resource run `kubectl create -f flightticket-custom-definition.yaml`

Here is another shorter example from a KodeKloud exercise:

```
kind: Global
apiVersion: traffic.controller/v1
metadata:
  name: datacenter
spec:
  dataField: 2
  access: true
```

Without a custom controller however, you can only store these custom resources in etcd. To actually be able to do things with the resources, you need to create a custom controller that can handle these resources.

## Custom Controllers

To create a custom controller, take a look at the k8s github repo [sample-controller](https://github.com/kubernetes/sample-controller) and modify the `controller.go` file with our custom code. After making your logic changes, you would build the file with `go build -o sample-controller .`, and then run the file to set it up into the kube config with `./sample-controller -kubeconfig=$HOME/.kube/config`.

## Operator Framework

** High Level ** - Using the operator framework (operators), CRDs and Custom Controllers can be packaged together into a single entity. This creates the custom resources and the controller as a deployment
