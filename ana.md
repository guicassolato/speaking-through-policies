# Ana: App Developer

![Ana](images/ana-intro.png)

### Login to the cluster

```sh
ANA_TOKEN=$(pbpaste)

kubectl config set-credentials ana --token=$ANA_TOKEN
kubectl config set-context ana --cluster=kind-evil-genius-cupcakes --user=ana --namespace=bakery-apps
alias kubectl="kubectl --context=ana"
```

### Deploy the baker app

```sh
kubectl apply -n bakery-apps -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: baker
spec:
  selector:
    matchLabels:
      app: baker
  template:
    metadata:
      labels:
        app: baker
    spec:
      containers:
      - name: baker-app
        image: quay.io/kuadrant/authorino-examples:baker-app
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8000
  replicas: 1
---
apiVersion: v1
kind: Service
metadata:
  name: baker
spec:
  selector:
    app: baker
  ports:
    - port: 8000
      protocol: TCP
EOF
```

### Test the baker app within the cluster

```sh
kubectl run curl -n bakery-apps --attach --rm --restart=Never -q --image=curlimages/curl --image-pull-policy=IfNotPresent -- http://baker:8000/baker -s
```

<br/>
<br/>
<br/>

### Check status of the gateway

```sh
kubectl get gateway/bakery-apps -n ingress-gateways -o jsonpath='{.status.conditions[?(.type=="Programmed")].status}'
# True%
```

### Attach a route to the gateway

```sh
kubectl apply -n bakery-apps -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: baker-route
spec:
  parentRefs:
  - kind: Gateway
    name: bakery-apps
    namespace: ingress-gateways
  rules:
  - matches:
    - path:
        value: /baker
    backendRefs:
    - kind: Service
      name: baker
      port: 8000
EOF
```

### Check the status of the route affected by a DNSPolicy

```sh
kubectl get httproute/baker-route -n bakery-apps \
  -o jsonpath='{.status.parents[?(.controllerName=="kuadrant.io/policy-controller")]}' | jq

# {
#   "conditions": [
#     {
#       "lastTransitionTime": "2025-03-19T11:35:43Z",
#       "message": "Object affected by DNSPolicy [ingress-gateways/bakery-dns]",
#       "observedGeneration": 1,
#       "reason": "Accepted",
#       "status": "True",
#       "type": "kuadrant.io/DNSPolicyAffected"
#     }
#   ],
#   "controllerName": "kuadrant.io/policy-controller", …
# }
```

### Check the spec of the gateway

```sh
kubectl get gateway/bakery-apps -n ingress-gateways -o yaml | yq
```

### Test the baker app through the gateway

In the browser:

Open http://cupcakes.demos.kuadrant.io/baker in a browser

In the terminal:

```sh
curl http://cupcakes.demos.kuadrant.io/baker
# 200
```

<br/>
<br/>
<br/>

### Test the baker app after TLS configured

```sh
curl http://cupcakes.demos.kuadrant.io/baker
# curl: (7) Failed to connect to cupcakes.demos.kuadrant.io port 80 after 7116 ms: Couldn't connect to server
```

### Check the status of the route affected by a TLSPolicy

```sh
kubectl get httproute/baker-route -n bakery-apps \
  -o jsonpath='{.status.parents[?(.controllerName=="kuadrant.io/policy-controller")]}' | jq

# {
#   "conditions": [
#     {
#       "lastTransitionTime": "2025-03-19T10:44:19Z",
#       "message": "Object affected by TLSPolicy [ingress-gateways/bakery-tls]",
#       "observedGeneration": 1,
#       "reason": "Accepted",
#       "status": "True",
#       "type": "kuadrant.io/TLSPolicyAffected"
#     }, …
#   ],
#   "controllerName": "kuadrant.io/policy-controller", …
# }
```

### Test the baker app hitting the HTTPS endpoint

```sh
curl https://cupcakes.demos.kuadrant.io/baker
# curl: (60) SSL certificate problem: unable to get local issuer certificate
# More details here: https://curl.se/docs/sslcerts.html
#
# curl failed to verify the legitimacy of the server and therefore could not
# establish a secure connection to it. To learn more about this situation and
# how to fix it, please visit the web page mentioned above.
```

### Bypass self-signed certificate verification by the client

```sh
curl https://cupcakes.demos.kuadrant.io/baker --insecure
# 200
```

<br/>
<br/>
<br/>

### Test the baker app behind the deny-all default auth policy

```sh
curl https://cupcakes.demos.kuadrant.io/baker --insecure
# {
#   "error": "Forbidden",
#   "message": "Access denied by default by the gateway operator. If you are the administrator of the service, create a specific auth policy for the route."
# }
```

### Check the status of the route affected by an AuthPolicy

```sh
kubectl get httproute/baker-route -n bakery-apps \
  -o jsonpath='{.status.parents[?(.controllerName=="kuadrant.io/policy-controller")]}' | jq

# {
#   "conditions": [
#     {
#       "lastTransitionTime": "2025-03-19T10:46:26Z",
#       "message": "Object affected by AuthPolicy [ingress-gateways/deny-all]",
#       "observedGeneration": 1,
#       "reason": "Accepted",
#       "status": "True",
#       "type": "kuadrant.io/AuthPolicyAffected"
#     }, …
#   ],
#   "controllerName": "kuadrant.io/policy-controller", …
# }
```

### Define an AuthPolicy to replace the default deny-all one

```sh
kubectl apply -n bakery-apps -f -<<EOF
apiVersion: kuadrant.io/v1
kind: AuthPolicy
metadata:
  name: baker-auth
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: baker-route
  rules:
    authentication:
      "bakers": # north-south
        jwt:
          issuerUrl: https://gitlab.com
        credentials:
          cookie:
            name: jwt
        priority: 1
      "apps": # east-west
        kubernetesTokenReview:
          audiences:
          - https://kubernetes.default.svc.cluster.local
        overrides:
          "iss":
            value: https://kubernetes.default.svc.cluster.local
        priority: 0
    authorization:
      "bakers":
        when:
        - predicate: auth.identity.iss == "https://gitlab.com"
        patternMatching:
          patterns:
          - predicate: '"evil-genius-cupcakes" in auth.identity.groups_direct'
      "apps":
        when:
        - predicate: auth.identity.iss == "https://kubernetes.default.svc.cluster.local"
        patternMatching:
          patterns:
          - predicate: auth.identity.user.username.split(":")[2] == "bakery-apps" && request.method == "GET"
    response:
      unauthenticated:
        code: 302
        headers:
          location:
            value: https://gitlab.com/oauth/authorize?client_id=c0b3a4e52c5e60ccb40ccf7c9bd63828476cde4b71910beb463897069ce1ae29&redirect_uri=https://cupcakes.demos.kuadrant.io/auth/callback&response_type=code&scope=openid
          set-cookie:
            expression: |
              "target=" + request.path + "; domain=cupcakes.demos.kuadrant.io; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=3600"
EOF
```

Check the status of the AuthPolicy:

```sh
kubectl get authpolicy/baker-auth -o yaml | yq
```

### Create a route and AuthPolicy to handle the OIDC flow

```sh
kubectl apply -f -<<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: oauth-route
spec:
  parentRefs:
  - kind: Gateway
    name: bakery-apps
    namespace: ingress-gateways
  rules:
  - matches:
    - path:
        value: /auth/callback
---
apiVersion: kuadrant.io/v1
kind: AuthPolicy
metadata:
  name: oauth
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: oauth-route
  rules:
    metadata:
      "token":
        when:
        - predicate: request.query.split("&").map(entry, entry.split("=")).filter(pair, pair[0] == "code").map(pair, pair[1]).size() > 0
        http:
          url: https://gitlab.com/oauth/token
          method: POST
          body:
            expression: |
              "code=" + request.query.split("&").map(entry, entry.split("=")).filter(pair, pair[0] == "code").map(pair, pair[1])[0] + "&redirect_uri=https://cupcakes.demos.kuadrant.io/auth/callback&client_id=c0b3a4e52c5e60ccb40ccf7c9bd63828476cde4b71910beb463897069ce1ae29&grant_type=authorization_code"
    authorization:
      "location":
        opa:
          rego: |
            cookies := { name: value |
              raw_cookies := input.request.headers.cookie
              cookie_parts := split(raw_cookies, ";")
              part := cookie_parts[_]
              kv := split(trim(part, " "), "=")
              count(kv) == 2
              name := trim(kv[0], " ")
              value := trim(kv[1], " ")
            }
            location := concat("", ["https://cupcakes.demos.kuadrant.io", cookies.target]) { input.auth.metadata.token.id_token; cookies.target }
            location := "https://cupcakes.demos.kuadrant.io/baker" { input.auth.metadata.token.id_token; not cookies.target }
            location := "https://gitlab.com/oauth/authorize?client_id=c0b3a4e52c5e60ccb40ccf7c9bd63828476cde4b71910beb463897069ce1ae29&redirect_uri=https://cupcakes.demos.kuadrant.io/auth/callback&response_type=code&scope=openid" { not input.auth.metadata.token.id_token }
            allow = true
          allValues: true
        priority: 1
      "deny":
        opa:
          rego: allow = false
        priority: 2
    response:
      unauthorized:
        code: 302
        headers:
          set-cookie:
            expression: |
              "jwt=" + auth.metadata.token.id_token + "; domain=cupcakes.demos.kuadrant.io; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=3600"
          location:
            expression: auth.authorization.location.location
EOF
```

Check the status of the AuthPolicy:

```sh
kubectl get authpolicy/oauth -o yaml | yq
```

### Test the baker app impersonating an external user

1. Open https://cupcakes.demos.kuadrant.io/baker in a browser<br/>
   _(The browser will redirect to the auth server's login page.)_

2. Log in with a valid Gitlab user who is a member of the 'evil-genius-cupcakes' group.<br/>
   _(The auth server will redirect to the page of the baker app originally requested.)_

### Test the baker app impersonating another pod running within the cluster

#### Create a Service Account to identify another pod 'toppings'

```sh
kubectl apply -n bakery-apps -f -<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: toppings
EOF
```

#### Send an authenticated request to the baker app

```sh
export POD_SA_TOKEN=$(kubectl create token toppings)
curl -H "Authorization: Bearer $POD_SA_TOKEN" https://cupcakes.demos.kuadrant.io/baker/api --insecure
# 200
```

#### Try to access a forbidden endpoint of the baker app

```sh
export POD_SA_TOKEN=$(kubectl create token toppings)
curl -H "Authorization: Bearer $POD_SA_TOKEN" https://cupcakes.demos.kuadrant.io/baker/api --insecure -X POST -i
# HTTP/2 403
# x-ext-auth-reason: Unauthorized
```

### Define a Rate Limit Policy for the baker app for a maximum of 5rp10s

```sh
kubectl apply -n bakery-apps  -f -<<EOF
apiVersion: kuadrant.io/v1
kind: RateLimitPolicy
metadata:
  name: baker-rate-limit
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: baker-route
  limits:
    global:
      rates:
      - limit: 5
        window: 10s
EOF
```

### Check the status of the route affected by a RateLimitPolicy

```sh
kubectl get httproute/baker-route -n bakery-apps \
  -o jsonpath='{.status.parents[?(.controllerName=="kuadrant.io/policy-controller")]}' | jq

# {
#   "conditions": [
#     {
#       "lastTransitionTime": "2025-03-19T10:51:48Z",
#       "message": "Object affected by RateLimitPolicy [bakery-apps/baker-rate-limit]",
#       "observedGeneration": 1,
#       "reason": "Accepted",
#       "status": "True",
#       "type": "kuadrant.io/RateLimitPolicyAffected"
#     }, …
#   ],
#   "controllerName": "kuadrant.io/policy-controller", …
# }
```

### Check the status of the RateLimitPolicy

```sh
kubectl get ratelimitpolicy/baker-rate-limit -o yaml | yq
```

### Send a few requests to the baker app

```sh
while :; do curl -H "Authorization: Bearer $POD_SA_TOKEN" https://cupcakes.demos.kuadrant.io/baker/api --insecure -s --output /dev/null --write-out '%{http_code}\n' | grep -E --color "\b(429)\b|$"; sleep 1; done
# 200
# 200
# 200
# 200
# 200
# 429
# 429
# 429
# 429
# 429
# 200
# 200
```

<br/>
<br/>
<br/>

### Check the status of the route affected by another RateLimitPolicy

```sh
kubectl get httproute/baker-route -n bakery-apps \
  -o jsonpath='{.status.parents[?(.controllerName=="kuadrant.io/policy-controller")]}' | jq

# {
#   "conditions": [
#     {
#       "lastTransitionTime": "2025-03-19T10:51:48Z",
#       "message": "Object affected by RateLimitPolicy [bakery-apps/baker-rate-limit ingress-gateways/gateway-rate-limit]",
#       "observedGeneration": 1,
#       "reason": "Accepted",
#       "status": "True",
#       "type": "kuadrant.io/RateLimitPolicyAffected"
#     }, …
#   ],
#   "controllerName": "kuadrant.io/policy-controller", …
# }
```

### Check the status of the baker RateLimitPolicy overridden by the more restrictive policy

```sh
kubectl get ratelimitpolicy/baker-rate-limit -n bakery-apps -o jsonpath='{.status.conditions[?(.type=="Enforced")]}' | jq
# {
#   "lastTransitionTime": "2025-03-19T11:09:17Z",
#   "message": "RateLimitPolicy is overridden by [ingress-gateways/gateway-rate-limit]",
#   "reason": "Overridden",
#   "status": "False",
#   "type": "Enforced"
# }
```

### Send a few requests to the baker app

```sh
while :; do curl -H "Authorization: Bearer $POD_SA_TOKEN" https://cupcakes.demos.kuadrant.io/baker/api --insecure -s --output /dev/null --write-out '%{http_code}\n' | grep -E --color "\b(429)\b|$"; sleep 1; done
# 200
# 200
# 429
# 429
# 429
# 429
# 429
# 429
# 429
# 429
# 200
# 200
```
