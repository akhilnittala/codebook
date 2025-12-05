# External Authentication with Keycloak for Argo CD (OpenShift)

This document describes the steps to configure external authentication via Keycloak for the Argo CD UI on an OpenShift cluster. Follow steps in order. Values, resource names and command strings are preserved exactly.

## 1. Launch the cluster
Create an AWS cluster via cluster bot:
- Command: launch 4.19 aws,techpreview

Confirm the TechPreviewNoUpgrade feature set has been enabled (required for External Authentication):

```sh
oc get featuregate cluster -o json | jq -r '.status.featureGates[0].enabled[] | select(.name == "ExternalOIDC") | length > 0'
```

If the response is `true`, the cluster supports the External Authentication feature.

## 2. Console / OperatorHub
Once the cluster is ready:
- Login to the OpenShift console.
- Install the `openshift-gitops` operator from Operator Hub.

## 3. Install Keycloak and PostgreSQL (templates provided below)

### 3.1 Create keycloak namespace
```sh
oc create ns keycloak
```

### 3.2 Create OperatorGroup for Keycloak
```sh
oc create -f - <<EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: keycloak-og
  namespace: keycloak
spec:
  targetNamespaces:
  - keycloak
  upgradeStrategy: Default
EOF
```

### 3.3 Create Subscription for Keycloak operator (RHBK)
```sh
oc create -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/rhbk-operator.keycloak: ""
  name: rhbk-operator
  namespace: keycloak
spec:
  channel: stable-v26.4
  installPlanApproval: Automatic
  name: rhbk-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### 3.4 Create PostgreSQL StatefulSet and Service
```sh
oc create -f - <<EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql-db
  namespace: keycloak
spec:
  serviceName: postgresql-db-service
  selector:
    matchLabels:
      app: postgresql-db
  replicas: 1
  template:
    metadata:
      labels:
        app: postgresql-db
    spec:
      containers:
        - name: postgresql-db
          image: postgres:latest
          volumeMounts:
            - mountPath: /data
              name: cache-volume
          env:
            - name: POSTGRES_PASSWORD
              value: testpassword
            - name: PGDATA
              value: /data/pgdata
            - name: POSTGRES_DB
              value: keycloak
      volumes:
        - name: cache-volume
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-db
  namespace: keycloak
spec:
  selector:
    app: postgresql-db
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
EOF
```

### 3.5 Expose DB service as a route
```sh
oc expose service postgres-db -n keycloak --name=postgres-db-route
```

### 3.6 Create DB secret
```sh
oc create secret generic keycloak-db-secret -n keycloak\
  --from-literal=username=postgres \
  --from-literal=password=testpassword
```

### 3.7 Create Keycloak instance
```sh
oc create -f - <<EOF
apiVersion: k8s.keycloak.org/v2alpha1
kind: Keycloak
metadata:
  name: argocd-keycloak
  namespace: keycloak
spec:
  instances: 1
  db:
    vendor: postgres
    host: postgres-db
    usernameSecret:
      name: keycloak-db-secret
      key: username
    passwordSecret:
      name: keycloak-db-secret
      key: password
  http:
    tlsSecret: argocd-keycloak-tls
    insecureSkipVerify: true
  ingress:
    className: openshift-default
EOF
```

## 4. Verify Keycloak components
Check pods in the `keycloak` namespace:
```sh
# Example watch output (for reference)
akhilnittala@nakhil-mac ~ % oc get po -n keycloak -w
NAME                             READY   STATUS    RESTARTS   AGE
argocd-keycloak-0                1/1     Running   0          75s
postgresql-db-0                  1/1     Running   0          3m54s
rhbk-operator-5d9bcd868d-crt8k   1/1     Running   0          4m36s
```

## 5. Annotate Keycloak service
```sh
oc annotate svc/argocd-keycloak-service -n keycloak service.beta.openshift.io/serving-cert-secret-name=argocd-keycloak-tls
```

## 6. Wait for Keycloak readiness
```sh
oc wait keycloaks/argocd-keycloak -n keycloak \
  --for=jsonpath='{.status.conditions[?(@.type=="Ready")].status}=True' \
  --timeout=3m
```

## 7. Retrieve Keycloak admin credentials
```sh
oc get secret argocd-keycloak-initial-admin -n keycloak -o jsonpath='{.data.username}' | base64 -d
oc get secret argocd-keycloak-initial-admin -n keycloak -o jsonpath='{.data.password}' | base64 -d
```

## 8. Create Keycloak realm (KeycloakRealmImport)
```sh
oc create -f - <<EOF
apiVersion: k8s.keycloak.org/v2alpha1
kind: KeycloakRealmImport
metadata:
  name: argocd-realm-kc
  namespace: keycloak
spec:
  keycloakCRName: argocd-keycloak
  realm:
    id: argocd-realm
    realm: argocd-realm
    displayName: Argo CD Realm
    enabled: true
EOF
```

Wait for the realm to initialize:
```sh
oc wait keycloakrealmimport/argocd-realm-kc -n keycloak \
  --for=jsonpath='{.status.conditions[?(@.type=="Done")].status}=True' \
  --timeout=3m
```

## 9. Get Keycloak route and connect to admin console
Get routes (note: example shows a small typo in a prior command; correct namespace used is `keycloak`):
```sh
oc get routes -n keycloak
```

Example output (for reference):
```
akhilnittala@nakhil-mac ~ % oc get routes -n keycloak
NAME                            HOST/PORT                                                                          PATH   SERVICES                  PORT    TERMINATION            WILDCARD
argocd-keycloak-ingress-cfc27   argocd-keycloak-ingress-keycloak.apps.ci-ln-vq7yx72-76ef8.aws-2.ci.openshift.org          argocd-keycloak-service   https   passthrough/Redirect   None
postgres-db-route               postgres-db-route-keycloak.apps.ci-ln-vq7yx72-76ef8.aws-2.ci.openshift.org                postgres-db               5432                           None
akhilnittala@nakhil-mac ~ %
```

Open the Keycloak admin console in a browser using the route shown above. Log in with credentials from step 7.

## 10. Configure Keycloak realm and client for Argo CD
1. In Keycloak admin console:
   - Make the "argocd-realm" the active realm.
2. Create a new client:
   - Left sidebar > Clients > Create client
   - Client ID: argocd
   - Client authentication: On
   - Root URL: (use the ArgoCD server route)  
     Example: https://openshift-gitops-server-openshift-gitops.apps.ci-ln-vq7yx72-76ef8.aws-2.ci.openshift.org
   - Home URL: /applications
   - Valid Redirect URLs:  
     https://openshift-gitops-server-openshift-gitops.apps.ci-ln-vq7yx72-76ef8.aws-2.ci.openshift.org/auth/callback
   - Valid Post Logout URI:  
     https://openshift-gitops-server-openshift-gitops.apps.ci-ln-vq7yx72-76ef8.aws-2.ci.openshift.org/applications
   - Web Origin: add the root origin (+)
3. Save the client.

## 11. Patch Argo CD secret with client credentials
Get the client secret from Keycloak admin console:
- Keycloak admin console > Clients > (argocd) > Credentials tab > Client secret (copy)

Patch the secret in the `openshift-gitops` namespace:
```sh
oc patch secret argocd-secret -n openshift-gitops \
  --type=merge \
  -p '{"stringData":{"oidc.keycloak.clientSecret":"v3QTAQaanSnuJnWMJwyITedwbGovT0Iy"}}'
```

Verify the secret:
```sh
oc get secret argocd-secret -n openshift-gitops -o jsonpath='{.data.oidc\.keycloak\.clientSecret}' | base64 -d
```

## 12. Add groups claim and mappers in Keycloak
To ensure Argo CD receives group information in the ID token:

1. Create a Client Scope named `groups`:
   - Keycloak admin console > Client Scopes > Create client scope
   - Name: groups
   - Include in token scope: On
   - Save

2. Add a Token Mapper to the client scope:
   - Mappers > Configure a new mapper > Choose "Group Membership"
   - Name: groups
   - Token Claim Name: groups
   - Disable "Full group path"
   - Save

3. Add the `groups` client scope to the client:
   - Client > Client Scopes > Add client scope > choose `groups`
   - Add it to Default (recommended) so ArgoCD always receives group info

## 13. Create Keycloak users, groups and assign membership
1. Create users:
   - Keycloak admin console > Users > Create new user
   - Example: name (nakhil), email (nakhil@redhat.com), first name (akhil), last name (nittala)

2. Set password:
   - From created user > Credentials tab > Set password
   - Confirm password, toggle Temporary: Off, Save

3. Create group:
   - Keycloak admin console > Groups > Create group
   - Example group: argocd

4. Add user to group:
   - Users > select user > Groups > Join group > argocd > Join > Save

## 14. Add RBAC policy in Argo CD
Edit the Argo CD RBAC ConfigMap and add group -> role mapping:

```sh
kubectl edit cm argocd-rbac-cm -n openshift-gitops
```

Add the policy (example entry):
```
policy.csv: |
  g, argocd, role:admin
```

## 15. Configure Argo CD CR with OIDC and TLS skip
Add OIDC configuration and skip TLS verification in Argo CD CR:

```sh
oc patch argocd openshift-gitops -n openshift-gitops --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/oidcConfig",
    "value": "name: Keycloak\nissuer: https://argocd-keycloak-ingress-keycloak.apps.ci-ln-vq7yx72-76ef8.aws-2.ci.openshift.org/realms/argocd-realm\nclientID: argocd\nclientSecret: $oidc.keycloak.clientSecret\nrequestedScopes: [\"openid\", \"profile\", \"email\", \"groups\"]"
  },
  {
    "op": "add",
    "path": "/spec/sso",
    "value": null
  },
  {
    "op": "add",
    "path": "/spec/extraConfig",
    "value": {
      "oidc.tls.insecure.skip.verify": "true"
    }
  }
]'
```

## 16. Restart Argo CD server
Delete the Argo CD server pod so changes take effect (replace the pod name with the current pod if different):
```sh
oc delete po openshift-gitops-server-867599dc85-qd6lb -n openshift-gitops
```

Wait for the pod to come up, then open the Argo CD route:
- Open console > Networking > Routes > argocd-openshift-gitops-server — copy the route URL
- Use "Login via Keycloak" option

## 17. Login and verify
Log in via Keycloak using the username/password created in steps 11 and 12 (user creation and password steps). You should be able to log in and see the Argo CD UI with the assigned roles.

## References
- CODEBOOK: https://github.com/anandf/openshift-gitops-cookbook/tree/main/keycloak-integration
- ARGOCD UI KEYCLOAK INTEGRATION: https://argo-cd.readthedocs.io/en/stable/operator-manual/user-