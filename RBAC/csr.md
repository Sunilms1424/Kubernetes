### Create private key ###
```
openssl genrsa -out myuser.key 2048
```
this command is create myuser.key
```
openssl req -new -key myuser.key -out myuser.csr -subj "/CN=myuser"
```
this command is create myuser.csr
note : you can what ever name you want insted of myuser place

### Create a CertificateSigningRequest ###
Create a CertificateSigningRequest and submit it to a Kubernetes Cluster via kubectl. Below is a script to generate the CertificateSigningRequest
```
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: myuser
spec:
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZVzVuWld4aE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQTByczhJTHRHdTYxakx2dHhWTTJSVlRWMDNHWlJTWWw0dWluVWo4RElaWjBOCnR2MUZtRVFSd3VoaUZsOFEzcWl0Qm0wMUFSMkNJVXBGd2ZzSjZ4MXF3ckJzVkhZbGlBNVhwRVpZM3ExcGswSDQKM3Z3aGJlK1o2MVNrVHF5SVBYUUwrTWM5T1Nsbm0xb0R2N0NtSkZNMUlMRVI3QTVGZnZKOEdFRjJ6dHBoaUlFMwpub1dtdHNZb3JuT2wzc2lHQ2ZGZzR4Zmd4eW8ybmlneFNVekl1bXNnVm9PM2ttT0x1RVF6cXpkakJ3TFJXbWlECklmMXBMWnoyalVnald4UkhCM1gyWnVVV1d1T09PZnpXM01LaE8ybHEvZi9DdS8wYk83c0x0MCt3U2ZMSU91TFcKcW90blZtRmxMMytqTy82WDNDKzBERHk5aUtwbXJjVDBnWGZLemE1dHJRSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBR05WdmVIOGR4ZzNvK21VeVRkbmFjVmQ1N24zSkExdnZEU1JWREkyQTZ1eXN3ZFp1L1BVCkkwZXpZWFV0RVNnSk1IRmQycVVNMjNuNVJsSXJ3R0xuUXFISUh5VStWWHhsdnZsRnpNOVpEWllSTmU3QlJvYXgKQVlEdUI5STZXT3FYbkFvczFqRmxNUG5NbFpqdU5kSGxpT1BjTU1oNndLaTZzZFhpVStHYTJ2RUVLY01jSVUyRgpvU2djUWdMYTk0aEpacGk3ZnNMdm1OQUxoT045UHdNMGM1dVJVejV4T0dGMUtCbWRSeEgvbUNOS2JKYjFRQm1HCkkwYitEUEdaTktXTU0xMzhIQXdoV0tkNjVoVHdYOWl4V3ZHMkh4TG1WQzg0L1BHT0tWQW9FNkpsYWFHdTlQVmkKdjlOSjVaZlZrcXdCd0hKbzZXdk9xVlA3SVFjZmg3d0drWm89Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
```
note: 
- usages: it has to be 'client auth'
- expirationSeconds could be made longer (i.e. 864000 for ten days) or shorter (i.e. 3600 for one hour)
- request is the base64 encoded value of the CSR file content. You can get the content using this command:
```
cat myuser.csr | base64 | tr -d "\n"
```
### Approve the CertificateSigningRequest ###
Use kubectl to create a CSR and approve it.
Get the list of CSRs:
```
kubectl get csr
```
Approve the CSR:
```
kubectl certificate approve myuser
```
After approving the csr you need get that certificate

The certificate value is in Base64-encoded format under status.certificate.

Export the issued certificate from the CertificateSigningRequest.
```
kubectl get csr myuser -o jsonpath='{.status.certificate}'| base64 -d > myuser.crt
```
till here this are the step to get the csr

### Create Role and RoleBinding ###
With the certificate created it is time to define the Role and RoleBinding for this user to access Kubernetes cluster resources.
#### role: ####
role : The role is responsible for creating the resources that users can access, and these resources are defined in the `role.yml` file.

role.yml(declaritive way)
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```
resources : here user have the access of pod 
verbs : user can have the access to get, watch, and list the pod

#### rolebinding ####
rolebinding :

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-access-binding
  namespace: default # Specify the namespace where the RoleBinding applies
subjects:
  - kind: User # Can be User, Group, or ServiceAccount
    name: myuser # Replace with the actual username or service account
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role # Refers to the kind of role (Role or ClusterRole)
  name: pod-reader # Replace with the name of the Role or ClusterRole
  apiGroup: rbac.authorization.k8s.io
```
subjects: here you need to provide the user or group details 
roleRef: here you need to porvide the role information which you created early.

### Add to kubeconfig ###
The last step is to add this user into the kubeconfig file
First, you need to add new credentials:
```
kubectl config set-credentials myuser --client-key=myuser.key \
--client-certificate=myuser.crt \
 --embed-certs=true
```
Then, you need to add the context:
```
kubectl config set-context myuser --cluster=kubernetes --user=myuser
```
here you are adding the user to cluster 
