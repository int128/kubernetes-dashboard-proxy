# Kubernetes Dashboard Proxy [![CircleCI](https://circleci.com/gh/int128/kubernetes-dashboard-proxy.svg?style=shield)](https://circleci.com/gh/int128/kubernetes-dashboard-proxy)

A Helm chart with [keycloak-proxy](https://github.com/gambol99/keycloak-proxy) to protect the Kubernetes Dashboard with OpenID Connect (OIDC).

![diagram.png](diagram.png)


## TL;DR

You can install the charts as follows:

```sh
# Kubernetes Dashboard
helm install stable/kubernetes-dashboard --namespace kube-system --name kubernetes-dashboard

# Kubernetes Dashboard Proxy
helm repo add int128.github.io https://int128.github.io/helm-charts
helm repo update
helm install int128.github.io/kubernetes-dashboard-proxy --namespace kube-system --name kubernetes-dashboard-proxy -f kubernetes-dashboard-proxy.yaml
```

See also [this article](https://medium.com/@int128/protect-kubernetes-dashboard-with-openid-connect-104b9e75e39c).


## Getting Started

### 1. Setup your OIDC Identity Provider

Setup an OIDC client on your Identity Provider.

#### Using Keycloak

Create an OIDC client as follows:

- Redirect URL: `https://kubernetes-dashboard.example.com/oauth/callback`
- Issuer URL: `https://keycloak.example.com/auth/realms/hello`
- Client ID: `kubernetes`
- Groups claim: `groups` (optional for group based access controll)

#### Using Google Account

Open [Google APIs Console](https://console.developers.google.com/apis/credentials) and create an OAuth client as follows:

- Application Type: Web application
- Redirect URL: `https://kubernetes-dashboard.example.com/oauth/callback`


### 2. Setup the Kubernetes API Server

Setup the Kubernetes API Server accepts an OIDC ID token.

If you are using kops, `kops edit cluster` and append the following settings:

```yaml
spec:
  kubeAPIServer:
    # using Keycloak
    oidcIssuerURL: https://keycloak.example.com/auth/realms/hello
    oidcClientID: kubernetes
    oidcGroupsClaim: groups

    # using Google Account
    oidcIssuerURL: https://accounts.google.com
    oidcClientID: xxx-xxx.apps.googleusercontent.com
```


### 3. Install the Kubernetes Dashboard and Proxy

You can install [the Kubernetes Dashboard](https://github.com/kubernetes/charts/tree/master/stable/kubernetes-dashboard) as follows:

```sh
helm install stable/kubernetes-dashboard --namespace kube-system --name kubernetes-dashboard
```

You can install the Kubernetes Dashboard Proxy as follows:

```sh
helm repo add int128.github.io https://int128.github.io/helm-charts
helm repo update
helm install int128.github.io/kubernetes-dashboard-proxy --namespace kube-system --name kubernetes-dashboard-proxy -f kubernetes-dashboard-proxy.yaml
```

You can set the following values for the Kubernetes Dashboard Proxy.
See also [kubernetes-dashboard-proxy.yaml](kubernetes-dashboard-proxy.yaml).

Parameter | Description | Default
----------|-------------|--------
`proxy.oidc.discoveryURL` | Discovery URL. | (mandatory)
`proxy.oidc.clientID` | Client ID. | (mandatory)
`proxy.oidc.clientSecret` | Client secret. | (mandatory)
`proxy.oidc.redirectURL` | Redirect URL. This may be same to the external URL in most cases. | (mandatory)
`proxy.cookieEncryptionKey` | Encryption key to store a session to a browser cookie. This should be 16 or 32 bytes string. | 32 bytes random string
`proxy.upstreamURL` | Kubernetes Dashboard service URL. | `https://kubernetes-dashboard.kube-system.svc.cluster.local`.
`ingress.enabled` | Enable ingress controller resource. | `false`
`ingress.hosts` | Hostnames | `[]`

#### nginx-ingress

If you are using [nginx-ingress](https://github.com/kubernetes/ingress-nginx), make sure `proxy_buffer_size` option is larger than 4kB.
You can set it in the ConfigMap of nginx-ingress.

```yaml
    proxy-buffer-size: "64k"
```


### 4. Assign a role

Open `https://kubernetes-dashboard.example.com`.

At this time, an `Unauthorized` error may appear on the dashboard because you have no role.

Here assign the `cluster-admin` role to the current group.

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: keycloak-admin-group
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  # NOTE: This is a super administrator and can do everything.
  # Consider a dedicated role in your actual operation.
  name: cluster-admin
subjects:
# (using Keycloak) If you want to specify the current user
- kind: User
  name: https://keycloak.example.com/auth/realms/hello#874c4a74-faf3-45a0-bcfe-9ddf4fb802ea
# (using Keycloak) If you want to specify the current group
- kind: Group
  name: /admin
# (using Google Account)
- kind: User
  name: https://accounts.google.com#1234567890
```

Congratulations!
Now all objects should appear in the dashboard.


## Advanced

### Manage the charts using Helmfile

You can manage the following charts using [Helmfile](https://github.com/roboll/helmfile).

- Kubernetes Dashboard Proxy
- [Kubernetes Dashboard](https://github.com/kubernetes/charts/tree/master/stable/kubernetes-dashboard)
- [Heapster](https://github.com/kubernetes/charts/tree/master/stable/heapster)

Run `helmfile sync` with the environment values exported.

```sh
export KUBE_DASHBOARD_DOMAIN=kubernetes-dashboard.example.com

# using Keycloak
export KUBE_OIDC_DISCOVERY_URL=https://keycloak.example.com/auth/realms/hello
export KUBE_OIDC_CLIENT_ID=kubernetes
export KUBE_OIDC_CLIENT_SECRET=Mx3xL96Ixn7j4ddWOCH1l8VkB6fiXDBW

# using Google Account
export KUBE_OIDC_DISCOVERY_URL=https://accounts.google.com
export KUBE_OIDC_CLIENT_ID=xxx-xxx.apps.googleusercontent.com
export KUBE_OIDC_CLIENT_SECRET=Mx3xL96Ixn7j4ddWOCH1l8VkB6fiXDBW
```

See [helmfile.yaml](helmfile.yaml) for details.


## Special thanks

This depends on [gambol99/keycloak-proxy](https://github.com/gambol99/keycloak-proxy).
Thank you for the great work.


## Contributions

This is an open source software licensed under Apache License 2.0.
Feel free to open issues or pull requests.
