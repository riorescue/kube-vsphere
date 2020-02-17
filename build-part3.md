# Kubernetes with vSphere Integration (Part 3)

In our last tutorial [Part 2](build-part2.md) we created an Ubuntu based Kubernetes cluster with Calico CNI. In this tutorial, we will be deploying the Kubernetes dashboard.

### Verify vSphere Cloud Provider
Now that we have a fully functioning (we hope) Kubernetes cluster we need to verify persistent storage with vSphere is working correctly. We will be testing by deploying a `StorageClass` and deploying mongodb to test that storage using Helm.

I will be using an Ubuntu 18.04 LTS machine to perform work in this tutorial. Installation of binaries will be specific to Ubuntu 18.04.

### Install Kubectl on Management System (Ubuntu 18.04)
Install kubectl:

```shell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list >/dev/null
```

```shell
$ yum update && yum install kubectl -y
```

After installing kubectl you will need a copy of `admin.conf` from the Control-Plane node at `/etc/kubernetes/admin.conf`. Copy `admin.conf` to the management machine (Ubuntu).

```shell
mkdir -p $HOME/.kube
sudo cp -i ~/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Test if connectivity to the cluster has been granted:

```shell
$ kubectl config view
```

### Install Kubernetes Dashboard
**On the control-plane node "k8s-master" ONLY**

We will deploy the latest Dashboard version compatable with the running version of Kubernetes (at time of writing 1.17.3).

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```

You should be presented with output similar to:

```shell
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```

Create a **Service Account** to access the Kubernetes Dashboard by creating a new file called `dash-user.yaml`:

```yaml
$ sudo tee ${PWD}/dash-user.yaml >/dev/null <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashuser
  namespace: kube-system
EOF
```

Create a **Cluster Role** for the new user `dashuser` so that it has `cluster-admin` privileges.

```yaml
$ sudo tee ${PWD}/dashuser-cluster-rolebinding.yaml >/dev/null <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashuser
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: dashuser
  namespace: kube-system
EOF
```

Import both `dash-user.yaml` and `dashuser-cluster-rolebinding.yaml` into Kubernetes:

```shell
kubectl create -f dash-user.yaml
kubectl create -f dashuser-cluster-rolebinding.yaml
```

Output:
```shell
serviceaccount/dashuser created
clusterrolebinding.rbac.authorization.k8s.io/dashuser created
```

We now have to get the access token for the `dashuser` account which is stored in a Kubernetes secret:

```shell
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep dashuser | awk '{print $1}')
```

Output:

```shell
Name:         dashuser-token-nlp4h
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashuser
              kubernetes.io/service-account.uid: 3d0487ee-222c-4cc1-a25d-d65a64678d53

Type:  kubernetes.io/service-account-token

Data
====
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImIySHRITWpGMjVfbThaWjNwQTk2TFg0NndreUxUd3h0cU1BMG5icEZHRDAifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNodXNlci10b2tlbi1ubHA0aCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJkYXNodXNlciIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjNkMDQ4N2VlLTIyMmMtNGNjMS1hMjVkLWQ2NWE2NDY3OGQ1MyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTpkYXNodXNlciJ9.EM7DNIEDATDnH7vgJXnHoWJALnQlBTm_FQvN10HWykEZ_nKXSN35-bUvwzVvslRLBh8fnT2VM-z7ofihup0hKtaJRmJ9vplyvSgAbMAq0bUSk5DC0KPk-ocS_gEiA8JckFi2kuXgCyZJASzfM01F-rCbcFJyWHBb7jBUuSSyGlTB4C5NABgpU2JDG4_2Ml6boUqM2Hl6pwTWZ6PT13kuZs8GmcZWRjUmGVOvQNEgTN_LtthjvOcnbAJZXZLNsZJxd5AuwBlnu5Dw4lg7I_1OO1WXN3TO-FURZ5BsyjPSHKy4CehSWF21gELxzSjgf-RS39WB3h-r9wiRj59j8oWTPA
ca.crt:     1025 bytes
```

By default, the Kubernetes Dashboard is ONLY accessble from within the cluster (not externally). You must leverage kubectl proxy to access the system. On the management node start the proxy:

```shell
$ kubectl proxy
```

Open the Dashboard

```shell
$ open http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

Change the access mode to `token` and copy/paste the token from the secret above. Below is an example of the token obtained from the secret.

```eyJhbGciOiJSUzI1NiIsImtpZCI6ImIySHRITWpGMjVfbThaWjNwQTk2TFg0NndreUxUd3h0cU1BMG5icEZHRDAifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNodXNlci10b2tlbi1ubHA0aCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJkYXNodXNlciIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjNkMDQ4N2VlLTIyMmMtNGNjMS1hMjVkLWQ2NWE2NDY3OGQ1MyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTpkYXNodXNlciJ9.EM7DNIEDATDnH7vgJXnHoWJALnQlBTm_FQvN10HWykEZ_nKXSN35-bUvwzVvslRLBh8fnT2VM-z7ofihup0hKtaJRmJ9vplyvSgAbMAq0bUSk5DC0KPk-ocS_gEiA8JckFi2kuXgCyZJASzfM01F-rCbcFJyWHBb7jBUuSSyGlTB4C5NABgpU2JDG4_2Ml6boUqM2Hl6pwTWZ6PT13kuZs8GmcZWRjUmGVOvQNEgTN_LtthjvOcnbAJZXZLNsZJxd5AuwBlnu5Dw4lg7I_1OO1WXN3TO-FURZ5BsyjPSHKy4CehSWF21gELxzSjgf-RS39WB3h-r9wiRj59j8oWTPA```

Once the Kubernetes Dashboard is loaded change the *Namespace* dropdown to *All namespaces*

![kube-dash](/assets/kube-dash.png)

