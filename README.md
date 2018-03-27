# Kubernetes Dashboard Proxy [![CircleCI](https://circleci.com/gh/int128/kubernetes-dashboard-proxy.svg?style=shield)](https://circleci.com/gh/int128/kubernetes-dashboard-proxy)

This is a Helm chart with [keycloak-proxy](https://github.com/gambol99/keycloak-proxy) to protect the Kubernetes Dashboard with OpenID Connect (OIDC).


## TL;DR

1. Setup your OIDC Identity Provider
1. Configure the Kubernetes API Server allows OIDC
1. Install Kubernetes Dashboard
1. Install Kubernetes Dashboard Proxy

You can install this using `helm` as follows:

```sh
helm repo add int128.github.io https://int128.github.io/helm-charts
helm repo update
helm install int128.github.io/kubernetes-dashboard-proxy -f kubernetes-dashboard-proxy.yaml
```


## Getting Started

`kubectl` and `helm` are required.


### 1. Setup OIDC Identity Provider

Setup an OIDC client with the following:

- Issuer URL: `https://keycloak.example.com/auth/realms/hello`
- Redirect URL: `https://kubernetes-dashboard.example.com/*`
- Client ID: `kubernetes`
- Client Secret: `Mx3xL96Ixn7j4ddWOCH1l8VkB6fiXDBW` (this is an example, usually generated random string)
- Groups claim: `groups` (optional for group based access controll)

If you are using Keycloak, see also [this article](https://medium.com/@int128/protect-kubernetes-dashboard-with-openid-connect-104b9e75e39c).


### 2. Setup Kubernetes API Server

Configure the kube-apiserver accepts an OIDC ID token.

If you are using kops, `kops edit cluster` and append the following settings:

```yaml
spec:
  kubeAPIServer:
    oidcClientID: kubernetes
    oidcGroupsClaim: groups
    oidcIssuerURL: https://keycloak.example.com/auth/realms/hello
```


### 3. Install Kubernetes Dashboard

You can install a dashboard from [the Helm chart](https://github.com/kubernetes/charts/tree/master/stable/kubernetes-dashboard).

```sh
helm install stable/kubernetes-dashboard --namespace kube-system --name kubernetes-dashboard
```


### 4. Install Kubernetes Dashboard Proxy

You can install this from this Helm chart.

```sh
helm repo add int128.github.io https://int128.github.io/helm-charts
helm repo update
helm install int128.github.io/kubernetes-dashboard-proxy -f kubernetes-dashboard-proxy.yaml
```

You can specify the following values.

Key | Value
----|------
`proxy.oidc.discoveryURL` | Discovery URL.
`proxy.oidc.clientID` | Client ID.
`proxy.oidc.clientSecret` | Client secret.
`proxy.oidc.redirectURL` | Redirect URL. This may be same to the external URL in most cases.
`proxy.cookieEncryptionKey` | Encryption key to store a session to a browser cookie. This should be 16 or 32 bytes string. Defaults to 32 bytes random string.
`proxy.upstreamURL` | Kubernetes Dashboard service URL. Defaults to `https://kubernetes-dashboard.kube-system.svc.cluster.local`.

You can install this from [helmfile.yaml](helmfile.yaml) using [helmfile](https://github.com/roboll/helmfile) as well.

```sh
export YOUR_DOMAIN=example.com
export YOUR_KEYCLOAK_REALM=hello
export YOUR_CLIENT_SECRET=Mx3xL96Ixn7j4ddWOCH1l8VkB6fiXDBW
helmfile sync
```

Now the dashboard is available via the Ingress.

If you are using [nginx-ingress](https://github.com/kubernetes/ingress-nginx), make sure `proxy_buffer_size` option is larger than 4kB.
You can configure that by the ConfigMap.

```yaml
    proxy-buffer-size: "64k"
```


### 5. Assign a role

Open the dashboard by `https://kubernetes-dashboard.example.com`.

Since no role is given to the current user or group, an Unauthorized warning may be shown on the dashboard.

Here assign the `cluster-admin` role to the current group.

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: keycloak-admin-group
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
# If you want to specify the current user
- kind: User
  name: https://keycloak.example.com/auth/realms/hello#874c4a74-faf3-45a0-bcfe-9ddf4fb802ea
# If you want to specify the current group
- kind: Group
  name: /admin
```

Now all objects are shown in the dashboard.

Note that the `cluster-admin` role is a super administrator and can do everything.
Consider a dedicated role in your actual operation.


## Special thanks

This depends on [gambol99/keycloak-proxy](https://github.com/gambol99/keycloak-proxy).
Thank you for the great work.


## Contributions

This is an open source software licensed under Apache License 2.0.
Feel free to open issues or pull requests.
