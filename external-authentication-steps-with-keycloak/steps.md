# This is the documentation for configuring external authentication via keycloak to argocd UI.
1. crete a aws cluster via cluster bot using command "launch 4.19 aws,techpreview"
Confirm the TechPreviewNoUpgrade feature set has been enabled which will then enable the External Authentication feature by executing the following command:

```sh
oc get featuregate cluster -o json | jq -r '.status.featureGates[0].enabled[] | select(.name == "ExternalOIDC") | length > 0'
```

If the response provided by the following command displays true, the cluster is eligible to leverage the External Authentication feature 

2. once the cluster is ready login to the console and install openshift-gitops operator from operator hub.