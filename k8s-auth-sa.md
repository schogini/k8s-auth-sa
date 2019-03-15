# K8s - Authentication, Authorization and ServiceAccounts

We will be using the single node AWS cluster we created earlier for this lab.



## Populating variables to be used for the lab on the Master node.

```
[ON MASTER NODE]
USER_NAME=user3
GROUPS=/o=group3
CLUSTER_NAME=cluster3
SERVER_URL=https://<master_node_private_ip>:6443
```

## Creating certificate requests, certificates, cluster, context and kubeconfig files.

Method1: Manually using custom script.
```
[ON MASTER NODE]
sudo su
nano authentication.sh   [Copy and paste the contents from the authentication.sh file from this repo.]
sh authentication.sh
```

Method2: Using docker container.
```
[ON MASTER NODE]
sudo su
mkdir /sree
docker run --rm \
-v /etc/kubernetes/pki/ca.crt:/usr/src/certs/ca.crt \
-v /sree:/certs \
-v ~/.kube/config:/root/.kube/config \
-e USER_NAME=user3 \
-e GROUPS=/o=group3 \
-e CLUSTER_NAME=cluster3 \
-e SERVER_URL=<master_node_private_ip>:6443 \
rbagby/kubernetes-cert-creator:0.1
```

## Change the KUBECONFIG value to the custom configuration.

```
kubectl get pods
export KUBECONFIG=/sree/config-user3
kubectl get pods
```
Note: You will see that once the kubeconfig is changed, you will get a forbidden message in the 'default' namespace.

## Resetting the KUBECONFIG value to default.

```
export KUBECONFIG=
kubectl get pods
```
Note: Once you change the kubeconfig to the default value, you should be able to access the 'default' namespace.

## Creating new namespace and creating RoleBinding, ClusterRole & ClusterRoleBinding for that namespace.

```
# Creating new namespace called sree.
kubectl create namespace sree

# Creating configuration files. Copy and paste the contents from the files in this repo with the same name. Inside these files, you will see the use of namespace and the usernames.
nano roleBinding.yaml
nano clusterRole.yaml
nano clusterRoleBinding.yaml

# Creating RoleBinding for user3 within namespace sree.
kubectl create -f roleBinding.yaml

# Creating ClusterRole for reading pods.
kubectl create -f clusterRole.yaml

# Creating ClusterRoleBinding to bind the ClusterRole with user3.
kubectl create -f clusterRoleBinding.yaml
```
Now your custom user named user3 is bound to the namespace sree.

## Testing RBAC.

```
# Switch to custom user configuration.
export KUBECONFIG=/sree/config-user3

# Try getting the pods for 'default' namespace. This will give forbidden error.
kubectl get pods

# Now try getting the pods for 'sree' namespace. This will succeed because of the role bindings.
kubectl get pods --namespace sree

# Try creating a deployment in the 'default' namespace. This will also give forbidden error.
kubectl run nginx2 --image nginx --generator=run-pod/v1

# Now try creating a deployment in the 'sree' namespace. This will succeed because of the role bindings.
# kubectl run nginx2 --image nginx --namespace sree --generator=run-pod/v1
```

## Creating ServiceAccounts

Creating ServiceAccounts and getting the informations.

```
# Adding Users to Kubernetes (kubectl)
kubectl create sa sree

# Get related secret
secret=$(kubectl get sa sree -o json | jq -r .secrets[].name)
echo $secret
kubectl get secret $secret -o json | jq -r '.data["ca.crt"]' | base64 -D

# Get ca.crt from secret (using OSX base64 with -D flag for decode)
kubectl get secret $secret -o json | jq -r '.data["ca.crt"]' | base64 -D > ca.crt

# Get service account token from secret
user_token=$(kubectl get secret $secret -o json | jq -r '.data["token"]' | base64 -D)
echo $user_token

# Get current context
kubectl config current-context
c=`kubectl config current-context`

# Get cluster name of context
name=`kubectl config get-contexts $c | awk '{print $3}' | tail -n l`
echo $name

# Get endpoint of current context
endpoint=`kubectl config view -o jsonpath="{.clusters[?(@.name == \"$name\")].cluster.server}"`
echo $endpoint
```

## Setting the cluster based on ServiceAccount.

```
# Set cluster (run in directory where ca.crt is stored)
kubectl config set-cluster cluster-staging \
  --embed-certs=true \
  --server=$endpoint \
  --certificate-authority=./ca.crt

# Set user credentials
kubectl config set-credentials sree-staging --token=$user_token

# Set context based on cluster, user and namespace.
kubectl config set-context sree-staging \
  --cluster=cluster-staging \
  --user=sree-staging \
  --namespace=sree
