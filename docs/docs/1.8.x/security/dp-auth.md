# Data plane proxy authentication

To obtain a configuration from the control-plane, a data plane proxy must authenticate itself. There are several authentication methods availble.

## Service Account Token

On Kubernetes, a data plane proxy proves its identity with the [Service Account Token](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#service-account-automation) that is mounted in every pod.

Keep in mind that if you don't explicitly specify `serviceAccountTokenName` in Deployment, the Pod is run with the `default` Service Account Token in the Namespace.
This means that authentication scope is bound to a Namespace, so any Pod in the Namespace can authenticate as any other Pod in a Namespace.

To have a strict security bound to a Deployment, every Deployment should use unique Service Account Token.
On top of that, users should not be able to modify `serviceAccountTokenName` in `Deployment`. This can be achieved for example with [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/docs/).

Service Account Token is not bound to a mesh, see [data plane proxy membership](./dp-membership.md) how to restrict which Pods can join a mesh.

## Data plane proxy token

On Universal, a data plane proxy must be explicitly configured with a unique security token (data plane proxy token) that will be used to prove its identity.

The data plane proxy token is a [JWT token](https://jwt.io) that contains:
* Mesh to which data plane belongs (required)
* The name of a data plane proxy (optional)
* List of tags that are permitted to use (optional)
* Expiration date of the token (required, 10 years if not specified)

The Data plane proxy token is signed by a signing key that is autogenerated when Mesh is created.
Tokens are never stored in the control plane, the only thing that is stored are signing keys that are used to verify if a token is valid.
The signing key is RSA256 encrypted.

You can check for the signing key:
```sh
kumactl get secrets
```
which returns something like:
```
MESH      NAME                                    AGE
default   dataplane-token-signing-key-default-1   49s
```

### Usage

You can generate token either by REST API
```bash
curl -XPOST \
  -H "Content-Type: application/json" \
  --data '{"name": "dp-echo-1", "mesh": "default", "tags": {"kuma.io/service": ["backend", "backend-admin"]}, "validFor": "720h"}' \
  http://localhost:5681/tokens/dataplane
```

or by using `kumactl`
```bash
kumactl generate dataplane-token \
  --name dp-echo-1 \
  --mesh default \
  --tag kuma.io/service=backend,backend-admin \
  --valid-for 720h > /tmp/kuma-dp-echo1-token
``` 

The token should be stored in a file and then used when starting `kuma-dp`
```bash
kuma-dp run \
  --cp-address=https://127.0.0.1:5678 \
  --dataplane-file=dp-backend.yaml \
  --dataplane-token-file=/tmp/kuma-dp-echo-1-token
```

You can also pass Dataplane Token inline as `KUMA_DATAPLANE_RUNTIME_TOKEN` Environment Variable.

### Data Plane Proxy Token boundary

As we can see in the example above, we can generate a token by passing a `name`, `mesh`, and a list of tags.
The control plane will then verify the data plane proxy resources that are connecting to it against the token. This means we can generate a token by specifying:

* Only `mesh`. By doing so we can reuse the token for all dataplanes in a given mesh.
* `mesh` + `tag` (ex. `kuma.io/service`). This way we can use one token across all instances/replicas of the given service. 
  Please keep in mind that we have to specify to include all the services that a data plane proxy is in charge of. 
  For example, if we have a Dataplane with two inbounds, one valued with `kuma.io/service: backend` and one with `kuma.io/service: backend-admin`, we need to specify both values (`--tag kuma.io/service=backend,backend-admin`).
* `mesh` + `name` + `tag` (ex. `kuma.io/service`). This way we can use one token for one instance/replica of the given service.

### Token Revocation

Kuma does not keep the list of issued tokens. Whenever the single token is compromised, we can add it to revocation list so it's no longer valid.

Every token has its own ID which is available in payload under `jti` key. You can extract ID from token using jwt.io or [`jwt-cli`](https://www.npmjs.com/package/jwt-cli) tool. Here is example of `jti`
```
0e120ec9-6b42-495d-9758-07b59fe86fb9
```

Specify list of revoked IDs separated by `,` and store it as `Secret` named `dataplane-token-revocations-{mesh}`

:::: tabs :options="{ useUrlFragment: false }"
::: tab "Universal"
```sh
echo "
type: Secret
mesh: default
name: dataplane-token-revocations-default
data: {{ revocations }}" | kumactl apply --var revocations=$(echo '0e120ec9-6b42-495d-9758-07b59fe86fb9' | base64) -f -
```
:::
::: tab "Kubernetes"
```sh
REVOCATIONS=$(echo '0e120ec9-6b42-495d-9758-07b59fe86fb9' | base64) && echo "apiVersion: v1
kind: Secret
metadata:
  name: dataplane-token-revocations-default
  namespace: kuma-system 
data:
  value: $REVOCATIONS
type: system.kuma.io/secret" | kubectl apply -f -
```
:::
::::

### Signing key rotation

If the signing key is compromised, we must rotate it and all the tokens that was signed by it.

1. Generate new signing key
   The signing key is stored as a `Secret` with a name that looks like `dataplane-token-signing-key-{mesh}-{serialNumber}`.

   Make sure to generate the new signing key with a serial number greater than the serial number of the current signing key.

   :::: tabs :options="{ useUrlFragment: false }"
   ::: tab "Universal"
   Check what is the current highest serial number.
   ```sh
   kumactl get secrets
   MESH      NAME                                    AGE
   default   dataplane-token-signing-key-default-1   49s
   ```

   In this case, the highest serial number is `1`. Generate a new signing key with a serial number of `2`
   ```sh
   echo "
   type: Secret
   mesh: default
   name: dataplane-token-signing-key-default-2
   data: {{ key }}" | kumactl apply --var key=$(kumactl generate signing-key) -f -
   ```
   :::
   ::: tab "Kubernetes"
   Check what is the current highest serial number.

   ```sh
   kubectl get secrets -n kuma-system --field-selector='type=system.kuma.io/secret'
   NAME                                 TYPE                    DATA   AGE
   dataplane-token-signing-key-mesh-1   system.kuma.io/secret   1      25m
   ```

   In this case, the highest serial number is `1`. Generate a new signing key with a serial number of `2`
   ```sh
   TOKEN="$(kumactl generate signing-key)" && echo "
   apiVersion: v1
   data:
     value: $TOKEN
   kind: Secret
   metadata:
     name: dataplane-token-signing-key-mesh-2
     namespace: kuma-system
   type: system.kuma.io/secret
   " | kubectl apply -f - 
   ```

   :::
   ::::

2. Regenerate tokens
   Create new data plane proxy tokens. These tokens are automatically created with the signing key that’s assigned the highest serial number, so they’re created with the new signing key.
   At this point, tokens signed by either new or old signing key are valid.

3. Remove the old signing key
   :::: tabs :options="{ useUrlFragment: false }"
   ::: tab "Universal"
   ```sh
   kumactl delete secret dataplane-token-signing-key-default-1 --mesh=default
   ```
   :::
   ::: tab "Kubernetes"
   ```sh
   kubectl delete secret dataplane-token-signing-key-default-1 -n kuma-system
   ```
   :::
   ::::
   All new connections to the control plane now require tokens signed with the new signing key.

### Token rotation

If you need to generate a new token for a `Dataplane` or you are using service account token projection on Kubernetes, it's possible to configure dynamic token reloading. To enable this behaviour, set the `kuma-cp` configuration property `dpServer.auth.useTokenPath` to `true`. When you enable the property, `kuma-dp` detects changes to the token file, reloads the token and uses the new value when establishing a new connection to `kuma-cp`.

### Multizone

When running in multizone, mode we can generate data plane proxy token both on global and zone control plane.
If the deployment pipeline is configured to generate data plane proxy token before running the proxy, it can rely on the Zone CP. This way Global CP is not a single point of failure.
Signing key rotation or token revocation should be performed on the global control plane.

## None

We can turn off the authentication by setting `KUMA_DP_SERVER_AUTH_TYPE` to `none`.

::: warning
If we disable the authentication between the control plane and the data plane proxies, any data plane proxy will be able to impersonate any service, therefore this is not recommended in production.
:::
