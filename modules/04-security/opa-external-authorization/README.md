# Security - OPA External Authorization

This section shows external authorization capabilities of Istio service-mesh on Amazon EKS using [OPA envoy
external authorizer](https://www.openpolicyagent.org/docs/latest/envoy-introduction/) as an external authorization policy
evaluation engine.

Istio proxy uses Envoy's [External Authorization filter](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/security/ext_authz_filter.html)
architecture to delegate authorization decisions to an external service. This allows application teams to
integrate with external policy stores and extend the authorization semantics beyond what is natively available
in Istio using [AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/).

The external authorization filter architecture supports deployment of the policy evaluation engine either as a
co-located process in the same VM or pod along side the Istio proxy sidecar or as a separate service external to the VM or pod. In this
section the OPA envoy external authorizer container is deployed as a sidecar along side the Istio proxies in selected
application pods to evaluate the policies locally. This deployment model reduces latency and improves availability of the
service at the cost of increasing the resource footprint of individual pods. The sidecar injection is configured
using [Gatekeeper](https://github.com/open-policy-agent/gatekeeper)'s [Mutation](https://open-policy-agent.github.io/gatekeeper/website/docs/mutation/) feature.

The following diagram depicts the initial OPA sidecar injection and the configuration of the envoy proxy
to delegate all authorization decision checks to the injected sidecar followed by the subsequent
data plane request/response flow.

![External authorization request response flow](/images/04-external-authorization.png)

The following sequence prepares the Istio service mesh for external authorization enforcement.
  1. Gatekeeper Assign CRD is created to inject OPA sidecar in workload pods
  2. Istio control plane is configured with the service endpoint details of the external authorization service
  3. Istio control plane is configured to apply `CUSTOM` [`AuthorizationPolicy`](https://istio.io/latest/docs/reference/config/security/authorization-policy/) to workload pods

The following sequence depicts the Istio data plane request/response flow.
  1. incoming client requests are intercepted by the `envoy` proxy
  2. if the request matches `CUSTOM` [`AuthorizationPolicy`](https://istio.io/latest/docs/reference/config/security/authorization-policy/), then `envoy` proxy creates an ext-authz request and invokes the configured external authorization endpoint
  3. the external authorizer sidecar evaluates authorization policy locally against the request context and decides whether to allow or deny the request
  4. the external authorizer sidecar sends a response with a boolean result value of either `true` (allow) or `false` (deny)
  5. if the response is boolean `true` (allow), then the envoy proxy proceeds with request invocation to the workload container; otherwise the envoy proxy denies the request and sends back HTTP `403 Forbidden` response to the client.

## Prerequisites

**Note:** Make sure that the required resources have been created following the [setup instructions](../README.md#setup).

**:warning: WARN:** Some of the commands shown in this section refer to relative file paths that assume the current directory is `istio-on-eks/modules/04-security/terraform`. If your current directory does not match this path, then either change to the above directory to execute the commands or if executing from any other directory, then adjust the file paths like `../scripts/helpers.sh` and `../lb_ingress_cert.pem` accordingly.

## Deploy

### Disallow Requests with Missing Tokens
Follow the steps in [Disallow requests with missing tokens](/modules/04-security/request-authn-authz/request-authentication/README.md#disallow-requests-with-missing-tokens) to reject requests with missing JWT tokens.

### Gatekeeper Mutations
This section leverages [Gatekeeper](https://github.com/open-policy-agent/gatekeeper)'s [Mutation](https://open-policy-agent.github.io/gatekeeper/website/docs/mutation/) feature to inject OPA envoy external authorizer sidecar to the application pods.

Gatekeeper is already installed in the cluster.

Verify that all the gatekeeper pods are running.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl get all -n gatekeeper-system
```

The output should list `gatekeeper-controller-manager` and `gatekeeper-audit` deployments like below. 
Wait till all the pods are running and ready.

```
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/gatekeeper-audit-5b55979884-fgrcb                1/1     Running   0          25s
pod/gatekeeper-controller-manager-69d88fcd4f-44rbv   1/1     Running   0          25s
pod/gatekeeper-controller-manager-69d88fcd4f-tvqpm   1/1     Running   0          25s
pod/gatekeeper-controller-manager-69d88fcd4f-zl9cg   1/1     Running   0          25s

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/gatekeeper-webhook-service   ClusterIP   172.20.115.23   <none>        443/TCP   26s

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gatekeeper-audit                1/1     1            1           27s
deployment.apps/gatekeeper-controller-manager   3/3     3            3           27s

NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/gatekeeper-audit-5b55979884                1         1         1       27s
replicaset.apps/gatekeeper-controller-manager-69d88fcd4f   3         3         3       27s
```

### Create mutations to inject OPA external authorizer sidecar

Add the `Assign` rules to inject the OPA server container as a sidecar
in the selected workload pods that you want to protect using external OPA based access control policies.

The file [`opa-ext-authz-sidecar-assign.yaml`](/modules/04-security/request-authn-authz/opa-external-authorization/opa-ext-authz-sidecar-assign.yaml) contains the `Assign` rules.

```yaml
apiVersion: mutations.gatekeeper.sh/v1
kind: Assign
metadata:
  name: opa-istio
spec:
  applyTo:
    - groups: [""]
      kinds: ["Pod"]
      versions: ["v1"]
  match:
    scope: Namespaced
    kinds:
      - apiGroups: ["*"]
        kinds: ["Pod"]
    namespaceSelector:
      matchLabels:
        opa-istio-injection: enabled
  location: "spec.containers[name:opa-istio]"
  parameters:
    pathTests:
      - subPath: "spec.containers[name:opa-istio]"
        condition: MustNotExist
    assign:
      value:
        image: openpolicyagent/opa:0.60.0-istio-static
        name: opa-istio
        args:
          - run
          - --server
          - --addr=localhost:8181
          - --diagnostic-addr=0.0.0.0:8282
          - --disable-telemetry
          - --set
          - "plugins.envoy_ext_authz_grpc.addr=:9191"
          - --set
          - "plugins.envoy_ext_authz_grpc.path=istio/authz/allow"
          - --set
          - "decision_logs.console=true"
          - --watch
          - /policy/policy.rego
        volumeMounts:
          - mountPath: /policy
            name: opa-policy
        readinessProbe:
          httpGet:
            path: /health?plugins
            port: 8282
        livenessProbe:
          httpGet:
            path: /health?plugins
            port: 8282
---
apiVersion: mutations.gatekeeper.sh/v1
kind: Assign
metadata:
  name: opa-policy
spec:
  applyTo:
    - groups: [""]
      kinds: ["Pod"]
      versions: ["v1"]
  match:
    scope: Namespaced
    kinds:
      - apiGroups: ["*"]
        kinds: ["Pod"]
    namespaceSelector:
      matchLabels:
        opa-istio-injection: enabled
  location: "spec.volumes[name:opa-policy]"
  parameters:
    pathTests:
      - subPath: "spec.volumes[name:opa-policy]"
        condition: MustNotExist
    assign:
      value:
        name: opa-policy
        configMap:
          name: opa-policy
---
```

#### `Assign` rule details

| Name | Selector | Target | Location | Purpose | References |
|------|----------|--------|----------|---------|------------|
| `opa-istio` | All pods in namespaces with label `opa-istio-injection`=`enabled` | Pods | Containers | Mutates selected pods to add a container named `opa-istio` only if it doesn't already exist. | References a volume named `opa-policy`. |
| `opa-policy` | All pods in namespaces with label `opa-istio-injection`=`enabled` | Pods | Volumes | Mutates selected pods to add a volume named `opa-policy` only if it doesn't already exist. | References a ConfigMap named `opa-policy`. The ConfigMap is created later in the workload namespace. |

#### Plugin arguments
Note the arguments for the `opa-istio` container above. A subset of the arguments are related to the envoy external authorizer gRPC plugin and are explained in more detail below.

| Name | Purpose |
|------|---------|
| `plugins.envoy_ext_authz_grpc.addr=:9191` | Starts the gRPC server listening on port `9191` |
| `plugins.envoy_ext_authz_grpc.path=istio/authz/allow` | The response path with the decision outcome |
| `decision_logs.console=true` | Redirect decision log to `stdout` |

#### Apply the mutations
Apply the `Assign` rules in `opa-ext-authz-sidecar-assign.yaml` manifest file.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl apply -f ../opa-external-authorization/opa-ext-authz-sidecar-assign.yaml
```

The output should look similar to the sample output below.

```
assign.mutations.gatekeeper.sh/opa-istio created
assign.mutations.gatekeeper.sh/opa-policy created
```

### Create DNS record for local authorizer sidecar

A [`ServiceEntry`](https://istio.io/latest/docs/reference/config/networking/service-entry/) DNS record allows dynamic
resolution of the external authorizer endpoint by the Istio proxies. This indirection provides cluster operators flexibility
to later relocate the external authorization service endpoint without updating the extension provider configuration in the
Istio mesh config as shown later.

Inspect the file [`opa-ext-authz-serviceentry.yaml`](/modules/04-security/request-authn-authz/opa-external-authorization/opa-ext-authz-serviceentry.yaml) for the `ServiceEntry` definition.

Note the `hosts` list value of `opa-ext-authz-grpc.local` and `ports` list value of `9191`. These values are registered
with Istio in the next step.

The DNS record for `opa-ext-authz-grpc.local` resolves to the loopback address `127.0.0.1` for the co-located OPA sidecar.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: opa-ext-authz-grpc-local
spec:
  hosts:
  - "opa-ext-authz-grpc.local"
  endpoints:
  - address: "127.0.0.1"
  ports:
  - name: grpc
    number: 9191
    protocol: GRPC
  resolution: DNS
```

Once the `ServiceEntry` is created the Istio proxies can dynamically resolve the configured external authorizer endpoint
using the host name `opa-ext-authz-grpc.local`.

Apply the manifest.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl apply -f ../opa-external-authorization/opa-ext-authz-serviceentry.yaml
```

The output should look similar to the sample output below.

```
serviceentry.networking.istio.io/opa-ext-authz-grpc-local created
```

### Register OPA envoy external authorizer extension with Istio

Update the `istio` ConfigMap in the root namespace (`istio-system`) to register the OPA external authorizer gRPC service as
an extension provider.

**:hourglass_flowing_sand: Command Line Execution**

```bash
EXT_ENVOY_EXT_AUTHZ_GRPC=$(cat <<EOM
extensionProviders:
- name: opa-ext-authz-grpc
  envoyExtAuthzGrpc:
    service: opa-ext-authz-grpc.local
    port: 9191
EOM
)

kubectl get cm istio -n istio-system -o json \
 | jq ".data.mesh += \"\n$EXT_ENVOY_EXT_AUTHZ_GRPC\"" \
 | kubectl apply -f -
```

The `kubectl apply` command may generate a warning similar to the one below, complaining of a missing annotation that 
`kubectl` uses to keep track of resources administered via `kubectl`. You can safely ignore this warning as 
this is the first time you are updating the `ConfigMap` using `kubectl`.

Make sure that the output shows that the ConfigMap is configured successfully.

```
Warning: resource configmaps/istio is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
configmap/istio configured
```

Verify that the `mesh` key in the ConfigMap has been updated.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl get cm istio -n istio-system -o jsonpath='{.data.mesh}'
```

The output should show a new `extensionProviders` list with the gRPC service coordinates of the envoy external
authorizer service.

```
accessLogFile: /dev/stdout
defaultConfig:
  discoveryAddress: istiod.istio-system.svc:15012
  tracing:
    zipkin:
      address: zipkin.istio-system:9411
defaultProviders:
  metrics:
  - prometheus
enablePrometheusMerge: true
rootNamespace: istio-system
trustDomain: cluster.local
extensionProviders:
- name: opa-ext-authz-grpc
  envoyExtAuthzGrpc:
    service: opa-ext-authz-grpc.local
    port: 9191
```

At this point, the mesh has been setup to start enforcing authorization policies using OPA.
The next step is to prepare the application workloads for policy enforcement.

### Setup application namespace for auto-injection

Label the `workshop` namespace so that Gatekeeper can automatically mutate the application pods.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl label namespace workshop opa-istio-injection=enabled
```

The output should look similar to the sample output below.

```
namespace/workshop labeled
```

### Add `AuthorizationPolicy` with `CUSTOM` action

The file [`productapp-authorizationpolicy.yaml`](/modules/04-security/request-authn-authz/opa-external-authorization/productapp-authorizationpolicy.yaml)
contains [`AuthorizationPolicy`](https://istio.io/latest/docs/reference/config/security/authorization-policy/)
definition for `workshop` namespace with `CUSTOM` action that forwards the access control decisions to the configured external authorizer.

Note that the authorization policy is applied to all the paths using `paths: ["*"]` matcher.

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: productapp
  namespace: workshop
spec:
  action: CUSTOM
  provider:
    name: opa-ext-authz-grpc
  rules:
  - to:
    - operation:
        paths: ["*"]
---
```

Apply the AuthorizationPolicy.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl apply -f ../opa-external-authorization/productapp-authorizationpolicy.yaml
```

The output should look similar to the sample output below.

```
authorizationpolicy.security.istio.io/productapp created
```

### Write the OPA policy

The OPA authorization policy is written in [Rego policy language](https://www.openpolicyagent.org/docs/latest/policy-language/).
The policy file [`policy.rego`](/modules/04-security/request-authn-authz/opa-external-authorization/policy.rego) is kept separate to allow
policy authors to test the policies independently by using `opa test` command.

For simplicity the policy is statically compiled and loaded in memory with the help of `opa-policy` ConfigMap
created later. A more scalable and robust production implementation will typically leverage bundles and
the set of [Management APIs](https://www.openpolicyagent.org/docs/latest/management-introduction/) OPA 
exposes to enable unified and logically centralized policy management.

The policy used in the demo uses [`io.jwt.decode`](https://www.openpolicyagent.org/docs/latest/policy-reference/#builtin-tokens-iojwtdecode) to decode
the bearer JWT tokens that contain embedded user roles in the `Authorization` HTTP header.

The bearer tokens are minted via Keycloak using the helper script [`helpers.sh`](/modules/04-security/scripts/helpers.sh) like below.

**:hourglass_flowing_sand: Command Line Execution**

```bash
TOKEN=$(../scripts/helpers.sh -g -u alice)
```

The minted tokens can then be inspected using the same helper script like below.

**:hourglass_flowing_sand: Command Line Execution**

```bash
../scripts/helpers.sh -i -t $TOKEN
```

#### Policy description

User `alice` is assigned to `guest` role and user `bob` is assigned to `admin` role. The roles
have associated permissions to access certain protected resources hosted by specific services.
There is also a set of unprotected resources that can be invoked by any user.

The policy enforces the following rules.

  * Allow anyone to access any of the operations in the `unprotected_operations` list
    ```
    ...
    allow if {
        some unprotected_operation in unprotected_operations
        unprotected_operation.host = http_destination[0]
        unprotected_operation.port = http_destination[1]
        unprotected_operation.method = http_request.method
        regex.match(unprotected_operation.path, http_request.path)
    }
    ...
    ```
  * Allow users with `guest` role to call `GET /` resources hosted by `frontend.workshop.svc.cluster.local` service at port `9000`
    ```
    ...
    "guest": [{
      "host": "frontend.workshop.svc.cluster.local",
      "port": "9000",
      "method": "GET",
      "path": "/",
    }]
    ...
    ```
  * Allow users with `admin` role to access `GET /` and `POST /products` resources hosted by
    `frontend.workshop.svc.cluster.local` service at port `9000`
    ```
    ...
    "admin": [
      {
        "host": "frontend.workshop.svc.cluster.local",
        "port": "9000",
        "method": "GET",
        "path": "/",
      },
      {
        "host": "frontend.workshop.svc.cluster.local",
        "port": "9000",
        "method": "POST",
        "path": "/products",
      },
    ]
    ...
    ```
  * Otherwise default deny
    ```
    ...
    default allow := false
    ...
    ```

#### Application personas
The following table lists the application personas.

| Role | Function |
|------|----------|
| `guest` | Views products list. |
| `admin` | Views and modifies products list. |
| `other` | Not authorized. |

#### Protected resources
The following table lists the protected resources and the roles authorized to access them. All access
requests to these protected resources by any other roles or unauthenticated identities are denied.

| Host | Port | Method | Path | Allowed roles |
|------|------|------|--------|---------------|
| `frontend.workshop.svc.cluster.local` | `9000` | `GET` | `/` | `guest`, `admin` |
| `frontend.workshop.svc.cluster.local` | `9000` | `POST` | `/products` | `admin` |

#### Unprotected resources
These resources are unprotected and can be accessed by any entity.

| Host | Port | Method | Path pattern |
|------|------|------|--------|
| `productcatalog.workshop.svc.cluster.local` | `5000` | `GET` | `^/products/$` |
| `productcatalog.workshop.svc.cluster.local` | `5000` | `GET` | `^/products/\\d+$` |
| `productcatalog.workshop.svc.cluster.local` | `5000` | `POST` | `^/products/\\d+$` |
| `catalogdetail.workshop.svc.cluster.local` | `3000` | `GET` | `^/catalogDetail$` |

#### User role mappings
The following table lists the user role assignments.

| Username | Role |
|----------|------|
| `alice` | `guest` |
| `bob` | `admin` |
| `charlie` | `other` |

Refer the [`policy.rego`](/modules/04-security/request-authn-authz/opa-external-authorization/policy.rego) file for the policy evaluation logic.

### Test the policy rules

The [`policy_test.rego`](/modules/04-security/request-authn-authz/opa-external-authorization/policy_test.rego) file contains test cases for the policy specified in `policy.rego` file.

#### Sample test scenario: `guest` role is denied access to `frontend` `POST /products`
The below snippet tests that user `alice` having `guest` role is not allowed to call `POST /products` hosted
on `frontend.workshop.svc.cluster.local:9000`.

```
...
test_frontend_post_products_guest_denied if {
	request := {
		"attributes": {
			"destination": {
				"principal": "spiffe://cluster.local/ns/workshop/sa/frontend-sa",
			},
			"request": {
				"http": {
					"host": "frontend.workshop.svc.cluster.local:9000",
					"method": "POST",
					"path": "/products",
					"headers": {"authorization": concat(" ", ["Bearer", get_bearer_token("alice")])},
				}
			}
		},
		"parsed_path": ["products"],
	}

	not allow with input as request
}
...
```

#### Sample test scenario: `admin` role is allowed access to `frontend` `POST /products`
The below snippet tests that user `bob` having `admin` role is allowed to call `POST /products`.

```
...
test_frontend_post_products_admin_allowed if {
	request := {
		"attributes": {
			"destination": {
				"principal": "spiffe://cluster.local/ns/workshop/sa/frontend-sa",
			},
			"request": {
				"http": {
					"host": "frontend.workshop.svc.cluster.local:9000",
					"method": "POST",
					"path": "/products",
					"headers": {"authorization": concat(" ", ["Bearer", get_bearer_token("bob")])},
				}
			}
		},
		"parsed_path": ["products"],
	}

	allow with input as request
}
...
```

#### Sample test scenario: any other identity is denied access to `frontend` `POST /products`
The below snippet tests that the call to `POST /products` by user `charlie` having no assigned roles is denied.

```
...
test_frontend_post_products_no_role_denied if {
	request := {
		"attributes": {
			"destination": {
				"principal": "spiffe://cluster.local/ns/workshop/sa/frontend-sa",
			},
			"request": {
				"http": {
					"host": "frontend.workshop.svc.cluster.local:9000",
					"method": "POST",
					"path": "/products",
					"headers": {"authorization": concat(" ", ["Bearer", get_bearer_token("charlie")])},
				}
			}
		},
		"parsed_path": ["products"],
	}

	not allow with input as request
}
...
```

The tests can be executed either by installing the `opa` binary following the instructions in
[Running OPA](https://www.openpolicyagent.org/docs/latest/#running-opa) or by running the same container
image injected as sidecar.

#### Run tests using installed `opa` binary

**:hourglass_flowing_sand: Command Line Execution**

```bash
opa test ../opa-external-authorization/policy_test.rego ../opa-external-authorization/policy.rego
```

If the policy is written correctly then all the tests should pass and you should see an output similar to the sample output
below.

```
PASS: xx/xx
```

#### Run tests using container image

**:hourglass_flowing_sand: Command Line Execution**

```bash
docker run \
  --name opa-istio \
  -v ./../opa-external-authorization/:/policy \
  --rm \
  openpolicyagent/opa:0.60.0-istio-static \
  test /policy/policy_test.rego /policy/policy.rego
```

Note that the current directory containing the policy and the test files are mounted as a volume at the path `/policy`.

If the policy is written correctly then all the tests should pass and you should see an output similar to the sample output
below.

```
PASS: xx/xx
```

### Generate `opa-policy` ConfigMap

After authoring and testing the policy rules, the next step is to generate a ConfigMap by importing
the policy file. To achieve this, [kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/builtins/)'s built-in
[ConfigMapGenerator](https://kubectl.docs.kubernetes.io/references/kustomize/builtins/#_configmapgenerator_) is used.

The policy file is imported through [`kustomization.yaml`](/modules/04-security/request-authn-authz/opa-external-authorization/kustomization.yaml) to create the
`opa-policy` ConfigMap in the `workshop` namespace. This ConfigMap is mounted as a volume in the injected `opa-istio` sidecar as explained earlier.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
configMapGenerator:
  - name: opa-policy
    namespace: workshop
    files:
    - policy.rego
generatorOptions:
  disableNameSuffixHash: true
```

Apply the `Kustomization` to create the ConfigMap.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl apply -k ../opa-external-authorization/
```

The output should look similar to the sample output below.

```
configmap/opa-policy created
```

Verify that the generated ConfigMap contains the entire content of the `policy.rego` policy file.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl describe configmap/opa-policy -n workshop
```

The output should look similar to the sample excerpt below.

```
Name:         opa-policy
Namespace:    workshop
Labels:       <none>
Annotations:  <none>

Data
====
policy.rego:
----
package istio.authz

import future.keywords

import input.attributes.destination.principal as principal
import input.attributes.request.http as http_request

default allow := false

allow if {
  some unprotected_operation in unprotected_operations
  unprotected_operation.method = http_request.method
  unprotected_operation.principal = principal
  regex.match(unprotected_operation.path, http_request.path)
}

...


BinaryData
====

Events:  <none>
```

### Restart the application deployments

Restart the application deployments to recreate the pods so that Gatekeeper can inject the OPA authorizer sidecar.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl rollout restart deployment/frontend -n workshop
kubectl rollout restart deployment/productcatalog -n workshop
kubectl rollout restart deployment/catalogdetail -n workshop
kubectl rollout restart deployment/catalogdetail2 -n workshop
```

The output should look similar to the sample output below.

```
deployment.apps/frontend restarted
deployment.apps/productcatalog restarted
deployment.apps/catalogdetail restarted
deployment.apps/catalogdetail2 restarted
```

Wait till all the older pods have been terminated and the new pods show 3/3 containers are ready and status are `Running`. 
It typically takes less than a minute.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl get pods -n workshop
```

The output should change from something similar to the sample output below

```
NAME                              READY   STATUS        RESTARTS   AGE
catalogdetail-6d8499cc46-qjdzb    3/3     Running       0          10s
catalogdetail-78fd977698-8h9b9    2/2     Terminating   0          13h
catalogdetail2-759645968b-5x97m   2/2     Terminating   0          13h
catalogdetail2-79c9bfc785-zdqnp   3/3     Running       0          8s
frontend-5ddfb8b6c4-g5x5m         3/3     Running       0          12s
frontend-dc8f8698c-mzb2w          2/2     Terminating   0          13h
productcatalog-796f5f4bbb-fdt6s   2/2     Terminating   0          13h
productcatalog-987858dbd-qk69t    3/3     Running       0          11s
```

to something similar to the sample output below.

```
NAME                              READY   STATUS    RESTARTS   AGE
catalogdetail-6d8499cc46-qjdzb    3/3     Running   0          61s
catalogdetail2-79c9bfc785-zdqnp   3/3     Running   0          59s
frontend-5ddfb8b6c4-g5x5m         3/3     Running   0          63s
productcatalog-987858dbd-qk69t    3/3     Running   0          62s
```

## Validate

Export the ingress URL.

**:hourglass_flowing_sand: Command Line Execution**

```bash
export ISTIO_INGRESS_URL=$(kubectl get svc istio-ingress -n istio-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')
```

### Test access

#### Unauthorized request to `GET /` with `other` role

Generate a `curl` request with a randomly generated `x-req-id` custom HTTP request header to allow us 
to uniquely locate the decision log entry when searching the OPA decision log.

**:hourglass_flowing_sand: Command Line Execution**

```bash
REQ_ID=$RANDOM
TOKEN=$(../scripts/helpers.sh -g -u charlie)
curl --cacert ../lb_ingress_cert.pem --cacert ../lb_ingress_cert.pem -H "x-req-id: $REQ_ID" -H "Authorization: Bearer $TOKEN" https://$ISTIO_INGRESS_URL -s -o /dev/null -w "HTTP Response: %{http_code}\n"
```

Output should be similar to:

```
HTTP Response: 403
```

Verify that the decision log shows a log entry with a `true` result matching the `x-req-id` custom 
HTTP request header.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl logs deployment/frontend -c opa-istio -n workshop \
  | grep "\"x-req-id\":\"$REQ_ID\"" \
  | grep -o "\"result\":false"
```

The output should show a matching entry like below.
The full matching decision log event JSON line can be inspected by removing the second `grep` command above.

```
"result":false
```

It is possible that the log entries may get rotated out as newer requests keep flowing in.
If no match is returned then rerun the `curl` request and search within a minute or so.

#### Authorized request to `GET /` with `guest` role

Generate a `curl` request with a randomly generated `x-req-id` custom HTTP request header to allow us 
to uniquely locate the decision log entry when searching the OPA decision log.

**:hourglass_flowing_sand: Command Line Execution**

```bash
REQ_ID=$RANDOM
TOKEN=$(../scripts/helpers.sh -g -u alice)
curl --cacert ../lb_ingress_cert.pem -H "x-req-id: $REQ_ID" -H "Authorization: Bearer $TOKEN" https://$ISTIO_INGRESS_URL -s -o /dev/null -w "HTTP Response: %{http_code}\n"
```

Output should be similar to:

```
HTTP Response: 200
```

Verify that the decision log shows a log entry with a `true` result matching the `x-req-id` custom 
HTTP request header.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl logs deployment/frontend -c opa-istio -n workshop \
  | grep "\"x-req-id\":\"$REQ_ID\"" \
  | grep -o "\"result\":true"
```

The output should show a matching entry like below.
The full matching decision log event JSON line can be inspected by removing the second `grep` command above.

```
"result":true
```

It is possible that the log entries may get rotated out as newer requests keep flowing in.
If no match is returned then rerun the `curl` request and search within a minute or so.

#### Authorized request to `GET /` with `admin` role

Generate a `curl` request with a randomly generated `x-req-id` custom HTTP request header to allow us 
to uniquely locate the decision log entry when searching the OPA decision log.

**:hourglass_flowing_sand: Command Line Execution**

```bash
REQ_ID=$RANDOM
TOKEN=$(../scripts/helpers.sh -g -u bob)
curl --cacert ../lb_ingress_cert.pem -H "x-req-id: $REQ_ID" -H "Authorization: Bearer $TOKEN" https://$ISTIO_INGRESS_URL -s -o /dev/null -w "HTTP Response: %{http_code}\n"
```

Output should be similar to:

```
HTTP Response: 200
```

Verify that the decision log shows a log entry with a `true` result matching the `x-req-id` custom 
HTTP request header.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl logs deployment/frontend -c opa-istio -n workshop \
  | grep "\"x-req-id\":\"$REQ_ID\"" \
  | grep -o "\"result\":true"
```

The output should show a matching entry like below.
The full matching decision log event JSON line can be inspected by removing the second `grep` command above.

```
"result":true
```

It is possible that the log entries may get rotated out as newer requests keep flowing in.
If no match is returned then rerun the `curl` request and search within a minute or so.

#### Unauthorized request to `POST /products` with `other` role

Generate a `curl` request with a randomly generated `x-req-id` custom HTTP request header to allow us 
to uniquely locate the decision log entry when searching the OPA decision log.

**:hourglass_flowing_sand: Command Line Execution**

```bash
REQ_ID=$RANDOM
TOKEN=$(../scripts/helpers.sh -g -u charlie)
curl --cacert ../lb_ingress_cert.pem -X POST -H "x-req-id: $REQ_ID" -H "Authorization: Bearer $TOKEN" https://$ISTIO_INGRESS_URL/products -s -o /dev/null -w "HTTP Response: %{http_code}\n"
```

Output should be similar to:

```
HTTP Response: 403
```

Verify that the decision log shows a log entry with a `false` result matching the `x-req-id` custom
HTTP request header.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl logs deployment/frontend -c opa-istio -n workshop \
  | grep "\"x-req-id\":\"$REQ_ID\"" \
  | grep -o "\"result\":false"
```

The output should show a matching entry like below.
The full matching decision log event JSON line can be inspected by removing the second `grep` command above.

```
"result":false
```

It is possible that the log entries may get rotated out as newer requests keep flowing in.
If no match is returned then rerun the `curl` request and search within a minute or so.

#### Unauthorized request to `POST /products` with `guest` role

Generate a `curl` request with a randomly generated `x-req-id` custom HTTP request header to allow us 
to uniquely locate the decision log entry when searching the OPA decision log.

**:hourglass_flowing_sand: Command Line Execution**

```bash
REQ_ID=$RANDOM
TOKEN=$(../scripts/helpers.sh -g -u alice)
curl --cacert ../lb_ingress_cert.pem -X POST -H "x-req-id: $REQ_ID" -H "Authorization: Bearer $TOKEN" https://$ISTIO_INGRESS_URL/products -s -o /dev/null -w "HTTP Response: %{http_code}\n"
```

Output should be similar to:

```
HTTP Response: 403
```

Verify that the decision log shows a log entry with a `false` result matching the `x-req-id` custom
HTTP request header.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl logs deployment/frontend -c opa-istio -n workshop \
  | grep "\"x-req-id\":\"$REQ_ID\"" \
  | grep -o "\"result\":false"
```

The output should show a matching entry like below.
The full matching decision log event JSON line can be inspected by removing the second `grep` command above.

```
"result":false
```

It is possible that the log entries may get rotated out as newer requests keep flowing in.
If no match is returned then rerun the `curl` request and search within a minute or so.

#### Authorized request to `POST /products` with `admin` role

Generate a `curl` request with a randomly generated `x-req-id` custom HTTP request header to allow us 
to uniquely locate the decision log entry when searching the OPA decision log.

**:hourglass_flowing_sand: Command Line Execution**

```bash
REQ_ID=$RANDOM
TOKEN=$(../scripts/helpers.sh -g -u bob)
curl --cacert ../lb_ingress_cert.pem -X POST -H "x-req-id: $REQ_ID" -H "Authorization: Bearer $TOKEN" https://$ISTIO_INGRESS_URL/products -d "id=1" -d "name=Apples" -s -o /dev/null -w "HTTP Response: %{http_code}\n"
```

Output should be similar to:

```
HTTP Response: 302
```

Verify that the decision log shows a log entry with a `true` result matching the `x-req-id` custom
HTTP request header.

**:hourglass_flowing_sand: Command Line Execution**

```bash
kubectl logs deployment/frontend -c opa-istio -n workshop \
  | grep "\"x-req-id\":\"$REQ_ID\"" \
  | grep -o "\"result\":true"
```

The output should show a matching entry like below.
The full matching decision log event JSON line can be inspected by removing the second `grep` command above.

```
"result":true
```

It is possible that the log entries may get rotated out as newer requests keep flowing in.
If no match is returned then rerun the `curl` request and search within a minute or so.

## Clean up

Clean up the resources set up in this section.

**:hourglass_flowing_sand: Command Line Execution**

```bash
# Delete the Assign rules
kubectl delete -f ../opa-external-authorization/opa-ext-authz-sidecar-assign.yaml
# Clean up the ServiceEntry
kubectl delete -f ../opa-external-authorization/opa-ext-authz-serviceentry.yaml

# Delete the opa-policy ConfigMap
kubectl delete -k ../opa-external-authorization/

# Remove AuthorizationPolicy
kubectl delete -f ../opa-external-authorization/productapp-authorizationpolicy.yaml

# Deregister the extension provider
kubectl get configmap/istio -n istio-system -o json \
  | sed 's/\\nextensionProviders:\\n- name: opa-ext-authz-grpc\\n  envoyExtAuthzGrpc:\\n    service: opa-ext-authz-grpc.local\\n    port: 9191//' \
  | kubectl apply -f -

# Remove the auto-injection label
kubectl label namespace workshop opa-istio-injection-

# Restart the deployments
kubectl rollout restart deployment/frontend -n workshop
kubectl rollout restart deployment/productcatalog -n workshop
kubectl rollout restart deployment/catalogdetail -n workshop
kubectl rollout restart deployment/catalogdetail2 -n workshop
```

Congratulations!!! You've now successfully validated OPA based external authorization setup in Istio on Amazon EKS. :tada:

You can either move on to the other sub-modules or if you're done with this module then refer to [Clean up](../README.md#clean-up) to clean up all the resources provisioned in this module.