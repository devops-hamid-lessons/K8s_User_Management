# How to manage users in K8s

This short tutorial tries to explain how to manage users and permissions in k8s using certificates and RBAC. 

We will take these steps:
1. Generating a `kubeconfig` file for a user named `bob` based on a certificate signed by k8s.
2. Giving access to the created user using RBAC.
3. Testing `kubeconfig` file and the granted permissions.
* Note that we consider `bob` as the name of the user and `devops` as his team name.

## Step 1: `kubeconfig` file generation
1.1. Generate a private key and a certificate signing request file for `bob`
``` 
openssl req -new -newkey rsa:4096 -nodes -keyout bob.key -out bob.csr -subj "/CN=bob/O=devops"
```
* Command above will generate two files: `bob.key` and `bob.csr`

1.2. Ask k8s to sign `bob.csr` and generate `bob.crt` as his certificate.
```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: bob
spec:
  groups:
  - system:authenticated
  request: $(cat bob.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
EOF
```

1.3. You need to approve the certificate, and also fetch and store it as `bob.crt` file.
```
kubectl get csr
kubectl certificate approve bob
kubectl get csr bob -o jsonpath='{.status.certificate}' | base64 --decode > bob.crt
```
1.4. Extract kubernetes cluster cert as `k8s-ca.crt`
```
kubectl config view -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' --raw | base64 --decode - > k8s-ca.crt
```

1.5. Now that we have all required files, we can generate the required `kubeconfig` file for bob.
```
kubectl config set-cluster $(kubectl config view -o jsonpath='{.clusters[0].name}') --server=$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}') --certificate-authority=k8s-ca.crt --kubeconfig=bob-k8s-config --embed-certs
kubectl config set-credentials bob --client-certificate=bob.crt --client-key=bob.key --embed-certs --kubeconfig=bob-k8s-config
kubectl config set-context bob --cluster=$(kubectl config view -o jsonpath='{.clusters[0].name}') --namespace=dev --user=bob --kubeconfig=bob-k8s-config
```
* above commands will create a kubeconfig file named `bob-k8s-config` for "bob". Check its content using `less bob-k8s-config` command.
* Note to the third command above and the param `--namespace=dev`, which will consider `dev` as the default `namespace` for `bob`. It means once bob is connected to the cluster, It will be inside `dev` namespace. You can remove that line. 
* Now using ` kubectl --kubeconfig bob-k8s-config config get-contexts`, You can see the `bob` context is ready.

1.6. Create `dev` namespace for `bob`
```  
kubectl create ns dev
```

## Step 2: Permission assignment using RBAC
Option 1. For namespace-restricted actions, apply `namespace_wide_RBAC.yaml` file. It will give access to get and list pods only inside the `dev` namespace. 
```
kubectl apply -f namespace_wide_RBAC.yaml
```
Option 2. For cluster-wide actions, apply `cluster_wide_RBAC.yaml` file. It will give access to get and list pods in all namespaces.
```
kubectl apply -f cluster_wide_RBAC.yaml
```

## Step 3. Test 
* After Step 2. al configs are finished and admin should deliver `bob-k8s-config` file to `bob` to do test as below.

3.1. To do test, after obtaining `bob-k8s-config` file, `bob` needs to first set context as `bob`:
```
kubectl --kubeconfig bob-k8s-config config use-context bob
```
3.2. Now `bob` can do test to check permissions based on your choice in step 2 (change the namespace to check permissions):
```
kubectl get pods --kubeconfig=bob-k8s-config -n dev
```
* To get rid of `--kubeconfig=bob-k8s-config` from above commands, rename `bob-k8s-config` file as `config` and relocate it to `~/.kube` folder.

* Another much more scalable approach to manage users is to use `kube-openid-connect`. To learn about it, refer to this link: `https://devopstales.github.io/kubernetes/kube-openid-connect-1.0/` 