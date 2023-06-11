# User Creation in Kubernetes

Kubernetes provides a robust role-based access control (RBAC) system that allows fine-grained control over user permissions within a cluster. In this blog post, we will walk through the steps to create a user in Kubernetes and grant them specific access using RBAC.

## Step 1: Create a Folder

First, create a folder to organize the necessary files. Run the following command to create a folder named `rbac`:

```bash
mkdir rbac
```

## Step 2: Generate Private Key and Certificate Signing Request

To generate a private key and certificate signing request (CSR) for the user, follow these steps:

```bash
# Generate a private key
openssl genrsa -out jane.key 2048

# Generate a CSR
openssl req -new -key jane.key -subj "/CN=jane" -out jane.csr
```

## Step 3: Encode the CSR

Encode the CSR to base64 format using the following command:

```bash
cat jane.csr | base64 | tr -d "\n"
```

Copy the output, as we will need it later.

## Step 4: Create a Kubernetes Resource

Create a YAML file named `csr.yaml` and populate it with the following content:

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  usages:
  - client auth
  signerName: kubernetes.io/kube-apiserver-client
  request: <output of previous step>
```

Replace `<output of previous step>` with the base64-encoded CSR obtained in Step 3.

Apply the YAML file to create the CertificateSigningRequest object in Kubernetes:

```bash
kubectl apply -f csr.yaml
```

## Step 5: Approve the Certificate Signing Request

To approve the Certificate Signing Request, run the following command:

```bash
kubectl certificate approve jane
```

## Step 6: Create the User's Public Key

There are two ways to obtain the user's signed certificate. Choose one of the following methods:

### Method 1:

```bash
kubectl get csr jane -o jsonpath='{.status.certificate}' | base64 --decode > jane.crt
```

### Method 2:

```bash
kubectl get csr jane -oyaml | grep "certificate"
```

Manually copy the value of the certificate and save it to a file named `jane.crt`.

Now you have the `jane.key` (private key) and `jane.crt` (public key) files.

## Step 7: Create a Kubeconfig File

Create a kubeconfig file to configure access for the user. Create a file named `jane.kubeconfig` and populate it with the following content:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <get it from /etc/kubernetes/pki/ca.crt>
    server: https://<API server IP>:<port>
  name: <cluster name>
contexts:
- context:
    cluster: <cluster name>
    user: jane
  name: <context name>
current-context: <context name>
kind: Config
preferences: {}
users:
- name: jane
  user:
    client-certificate-data: <jane.crt>
    client-key-data: <jane.key>
```

Replace the placeholders:
- `<get it from /etc/kubernetes/pki/ca.crt>`: Obtain the CA certificate authority data from `/etc/kubernetes/pki/ca.crt` and replace this placeholder.
- `<API server IP>:<port>`: Replace with the appropriate API

 server IP and port.
- `<cluster name>`: Provide a name for your cluster.
- `<context name>`: Provide a name for the context.

## Step 8: Verify Access

To check if the configuration is working, run the following command:

```bash
kubectl get pod --kubeconfig jane.kubeconfig
```

You may encounter an error like: "Error from server (Forbidden): pods is forbidden: User "jane" cannot list resource "pods" in API group "" at the cluster scope." This error indicates that the user "jane" does not have permission to access the "pods" resource.

## Step 9: Create a Role and Role Binding

To grant specific access to the user, create a Role and Role Binding. Create a file named `role.yaml` and populate it with the following content:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader-role
rules:
- apiGroups: [""]
  resources: ["pods", "deployments"]
  verbs: ["get", "list"]
```

Create a file named `bind.yaml` and populate it with the following content:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader-role
  apiGroup: rbac.authorization.k8s.io
```

Apply the Role and Role Binding YAML files:

```bash
kubectl apply -f role.yaml
kubectl apply -f bind.yaml
```

## Step 10: Verify Role and Role Binding

To verify that the Role and Role Binding were created successfully, run the following commands:

```bash
kubectl get role pod-reader-role
kubectl get rolebindings
kubectl describe rolebindings.rbac.authorization.k8s.io
```

Ensure that the Role and Role Binding are present and correctly bound to the user "jane".

## Step 11: Test Access

To test the configured access, run the following command:

```bash
kubectl get pod --kubeconfig jane.kubeconfig
```

Congratulations! You have successfully created a user in Kubernetes, generated their private and public key pair, configured access using a kubeconfig file, and granted specific access using RBAC.

In this blog post, we covered the essential steps for user creation and access management in Kubernetes. These steps can serve as a starting point for managing user access in your Kubernetes clusters. Remember to follow security best practices and tailor the permissions according to your specific requirements.

Happy Kubernetting!