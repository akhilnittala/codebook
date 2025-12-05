# This is the documentation for configuring external authentication via keycloak to argocd UI.
1. crete a aws cluster via cluster bot using command "launch 4.19 aws,techpreview"
Confirm the TechPreviewNoUpgrade feature set has been enabled which will then enable the External Authentication feature by executing the following command:

```sh
oc get featuregate cluster -o json | jq -r '.status.featureGates[0].enabled[] | select(.name == "ExternalOIDC") | length > 0'
```

If the response provided by the following command displays true, the cluster is eligible to leverage the External Authentication feature 

2. once the cluster is ready login to the console and install openshift-gitops operator from operator hub.
# Install keycloak and Postgres DB using below templates

3. create keycloak namespace

```sh
oc create ns keycloak
```

# create operator group for keycloak in keycloak namespace
4. create operator group

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

# create subscription for RHBK operator
5. create subscription for keycloak operator using below template

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

# create database for keycloak
6. create database for keycloak using below statefulset template

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

# expose db service as a route
7. expose db service as a roue

```sh
oc expose service postgres-db -n keycloak --name=postgres-db-route
```

# create db secret
8. create db secret 

```sh
oc create secret generic keycloak-db-secret -n keycloak\
  --from-literal=username=postgres \
  --from-literal=password=testpassword
```

# create keycloak instance
9. create keycloak instance in keycloak namespace

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

# check all pods are up and running in keycloak namespace
10. oc get po -n keycloak

```sh
akhilnittala@nakhil-mac ~ % oc get po -n keycloak -w
NAME                             READY   STATUS    RESTARTS   AGE
argocd-keycloak-0                1/1     Running   0          75s
postgresql-db-0                  1/1     Running   0          3m54s
rhbk-operator-5d9bcd868d-crt8k   1/1     Running   0          4m36s
```

# annotate the keycloak service
11. annotate the keycloak service

```sh
oc annotate svc/argocd-keycloak-service -n keycloak service.beta.openshift.io/serving-cert-secret-name=argocd-keycloak-tls
```

# wait for the keycloak instance to be ready
12. wait for keycloak instance to be ready

```sh
oc wait keycloaks/argocd-keycloak -n keycloak \
  --for=jsonpath='{.status.conditions[?(@.type=="Ready")].status}=True' \
  --timeout=3m
```

# get the keycloak admin secret
13. get the keycloak admin secret

```sh
oc get secret argocd-keycloak-initial-admin -n keycloak -o jsonpath='{.data.username}' | base64 -d
oc get secret argocd-keycloak-initial-admin -n keycloak -o jsonpath='{.data.password}' | base64 -d
```

# create the keycloak realm import custom resource to create a realm
14. create keycloak realm import cr

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

# wait for realm to be initialized
15. wait for realm to be initialized

```sh
oc wait keycloakrealmimport/argocd-realm-kc -n keycloak \
  --for=jsonpath='{.status.conditions[?(@.type=="Done")].status}=True' \
  --timeout=3m
```

# get the keycloak ingress route and connect to admin console
16. get the route to connect keycloak admin console

```sh
oc get routes -n keyclok


akhilnittala@nakhil-mac ~ % oc get routes -n keycloak
NAME                            HOST/PORT                                                                          PATH   SERVICES                  PORT    TERMINATION            WILDCARD
argocd-keycloak-ingress-cfc27   argocd-keycloak-ingress-keycloak.apps.ci-ln-vq7yx72-76ef8.aws-2.ci.openshift.org          argocd-keycloak-service   https   passthrough/Redirect   None
postgres-db-route               postgres-db-route-keycloak.apps.ci-ln-vq7yx72-76ef8.aws-2.ci.openshift.org                postgres-db               5432                           None
akhilnittala@nakhil-mac ~ %
```

# connect to keycloak admin console
17. open chrome browser and browse "argocd-keycloak-ingress-keycloak.apps.ci-ln-vq7yx72-76ef8.aws-2.ci.openshift.org" which is the route created for keycloak ingress service using credentials in step 13.
18. after logging in to admin console make the realm created "argocd-realm" as master realm.
19. create one client as below:

> before adding urls get the route url for argocd-server using "oc get routes -n openshift-gitops". copy it and use it in below parameters after https:// 



    left side bar > clients > create client > client id <argocd> > client authentication (on) > root url (https://openshift-gitops-server-openshift-gitops.apps.ci-ln-vq7yx72-76ef8.aws-2.ci.openshift.org) > home url (/applications) > valid redirect urls (https://openshift-gitops-server-openshift-gitops.apps.ci-ln-vq7yx72-76ef8.aws-2.ci.openshift.org/auth/callback) > valid post logout uri (https://openshift-gitops-server-openshift-gitops.apps.ci-ln-vq7yx72-76ef8.aws-2.ci.openshift.org/applications) > web origin (+)

20. save the client.
21. patch the secret in openshift-gitops namespace with client credentials value

# get the client credentials from keycloak admin console
22. keycloak admin console login > clients > created client(eg: argocd) > credentials tab > client secret (copy to clipboard)

```sh
oc patch secret argocd-secret -n openshift-gitops \
  --type=merge \
  -p '{"stringData":{"oidc.keycloak.clientSecret":"v3QTAQaanSnuJnWMJwyITedwbGovT0Iy"}}'

```

# verify the argocd-secret in openshift-gitops namespace
23. verify the key and value is updated in secret

```sh
oc get secret argocd-secret -n openshift-gitops -o jsonpath='{.data.oidc\.keycloak\.clientSecret}' | base64 -d
```

# create groups claimm, user, password in keycloak admin console
24. In order for ArgoCD to provide the groups the user is in we need to configure a groups claim that can be included in the authentication token.

To do this we'll start by creating a new Client Scope called groups.

keycloak admin console left tab > client scopes > create client scope > name (should be groups) > include in token scope (on) > save.

25. Once you've created the client scope you can now add a Token Mapper which will add the groups claim to the token when the client requests the groups scope.

In the Tab "Mappers", click on "Configure a new mapper" and choose Group Membership.

Make sure to set the Name as well as the Token Claim Name to groups. Also disable the "Full group path".

mappers > configure a new mapper > mapper type (group membership) > name (groups) > token claim name (groups) > full group path toggle (off) > save.

26. We can now configure the client to provide the groups scope.

Go back to the client we've created earlier and go to the Tab "Client Scopes".

Click on "Add client scope", choose the groups scope and add it either to the Default or to the Optional Client Scope.

If you put it in the Optional category you will need to make sure that ArgoCD requests the scope in its OIDC configuration. Since we will always want group information, I recommend using the Default category.



27. create users.
left navigation tab > users > create new user > name (nakhil) > email ( nakhil@redhat.com) > first name (akhil) > last name (nittala) > create.
28. create password.

from created users > credentials tab > set password > click password and confirm password > temporary toggle off > save password.
29. create groups
left tab > groups > create group > name (argocd) > create group
30. add group to users
left tab > users > groups > join group > argocd group > join > save.

31. add groups to argocd cm
policy.csv: |
    g, argocd, role:admin

```sh
kubectl edit cm argocd-rbac-cm -n openshift-gitops
```

32. add oidc config and tls skip in argocd cr

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

33. delete the argocd-server pod to make sure changes are reflecting,

```sh
oc delete po openshift-gitops-server-867599dc85-qd6lb -n openshift-gitops
```

34. wait for pod to come up and try to login to argocd route using login via keycloak

```sh
oc console > networking > routes > argocd-openshift-gitops-server - copy the route URL and try to login via keycloak option.
```

35. login via keycloak > username and password as created in step-27 and 28, we need to able to successfully login and see argocd ui.

# REFEREBCES:
CODEBOOK: https://github.com/anandf/openshift-gitops-cookbook/tree/main/keycloak-integration
ARGOCD UI KEYCLOAK INTEGRATION: https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/keycloak/