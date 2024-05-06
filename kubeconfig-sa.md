## Create a kubeconfig file for a Service Account

Create a kubeconfig file to access a cluster using a Service Account, typically for automation tasks in CICD pipelines but usable for other purposes.

Create Namespace
```
kubectl create ns automation
```

Create Service Account
```
kubectl create serviceaccount cicd -n automation
```

Create Secret (needed since k8s 1.24)
```
cat <<'EOF' > cicd-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: cicd
  name: cicd-secret
  namespace: automation
type: kubernetes.io/service-account-token
EOF
kubectl apply -f cicd-secret.yaml
```

Retrieve the token from the secret
```
export TOKEN=$(kubectl get secret cicd-secret -n automation -o jsonpath='{.data.token}' | base64 -d)
```

Create RBAC for the Servie Account (adjust to the type of access you need)
```
cat <<'EOF' > cicd-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cicd-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: cicd
  namespace: automation
EOF
kubectl apply -f cicd-rbac.yaml
```

Set context that has the right cluster info from your existing kubeconfig
```
export CONTEXT=good-context
```

Get cluster info from current kubeconfig using context name
```
export CLUSTER=$(kubectl config view -o jsonpath='{.contexts[?(@.name == "'$CONTEXT'")].context.cluster}')
```

Create new Credentials (we name our user with the Service Account name and the Cluster name)
```
kubectl config set-credentials cicd@$CLUSTER --token=$TOKEN
```

Create new Context with credentials and cluster info
```
kubectl config set-context cicd@$CLUSTER --cluster $CLUSTER --user=cicd@$CLUSTER
```

Switch to new context with Service Account credentials and export to kubeconfig file
```
kubectl config use-context cicd@$CLUSTER
kubectl config view --flatten --minify > ./cicd@$CLUSTER-kubeconfig.yaml
```
