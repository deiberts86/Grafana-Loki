# **Install MINIO-Operator**

## **Tools needed**
- kubectl
- helm
- yq ["Download **yq** here from Github"](https://github.com/mikefarah/yq/#install)
  - You can also do a "snap install yq" if on Ubuntu Systems


### **Pull Charts**

```sh
curl -O https://raw.githubusercontent.com/minio/operator/master/helm-releases/operator-5.0.9.tgz
curl -O https://raw.githubusercontent.com/minio/operator/master/helm-releases/tenant-5.0.9.tgz
```

OR

* Clone from repo1 if needed if in a DoD space:

  - **[Repo1-Minio-Operator](https://repo1.dso.mil/big-bang/product/packages/minio-operator.git)**
- Benefit of this includes ISTIO related chart values.

### **Deploy Operator**
```sh
 helm upgrade -i minio-operator ./operator-5.0.9.tgz --namespace minio-operator --create-namespace
```

### **Configure Operator**

* Only use `yq` if you need to adjust the service to NodePort
```sh
kubectl get service console -n minio-operator -o yaml > service.yaml
yq e -i '.spec.type="NodePort"' service.yaml
yq e -i '.spec.ports[0].nodePort = PORT_NUMBER' service.yaml
```
---

* Update replica count
```sh
kubectl get deployment minio-operator -n minio-operator -o yaml > operator.yaml
yq -i -e '.spec.replicas |= 1' operator.yaml
```
---

### **Install Cert-Manager**
- Install **[Cert-manager](https://https://cert-manager.io/docs/installation/helm/)** and create a self-signed Cluster issuer CA

* Create your Cluster CA Issuer
  - Note: Using Self-Signed certificates for this example but custom signing CA certificate can be used to distribute your certificates.

```sh
kubectl apply -f -<<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-issuer-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: self-signed-issuer-ca
  secretName: selfsigned-ca-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer-ca
spec:
  ca:
    secretName: selfsigned-ca-secret
EOF
```

---

- Create your ingress and annotate your `selfsigned-issuer-ca` cluster-issuer

```sh
kubectl apply -f -<<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: selfsigned-issuer-ca
  name: minio-operator-ingress
  namespace: minio-operator
spec:
  ingressClassName: nginx
  rules:
    - host: fqdn-site-name
      http:
        paths:
          - backend:
              service:
                name: console
                port:
                  number: 9090
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - fqdn-site-name
      secretName: string
EOF
```

---

### **Grab Your JWT Token**

* Export the following environment variable with kubectl command to retrieve JWT token

```sh
export SA_TOKEN=$(kubectl -n minio-operator  get secret console-sa-secret -o jsonpath="{.data.token}" | base64 --decode)
echo $SA_TOKEN
```

### **Navigate to your MinIO Operator site through your ingress**

* Go to your new ingress site you created and it should look something like this: https://fqdn-site-name/
  - Note: You should see a nice webpage with the Minio logo and you should have only one field to put in the JWT Token. From there, you can create your tenants through the UI.  If you rather control it with Helm, follow the instructions below.

### **OPTIONAL: Install Tenant with Helm**

* Note: `<tenant-name>` is whatever you want it to be.  If you want to make more tenants, simply re-run this command but ensure the names are not identical and optionally choose different namespace as well! Most likely you will need to point the tenant to the operator service or ingress for this to work properly and be manage by the operator.

```sh
helm upgrade -i <tenant-name> ./tenant-5.0.9.tgz -n <namespace> --values <values.name.yaml>
```

* Expose myminio-console for tenant
  - Note: Follow the similar process for creating an ingress for a longterm solution

```sh
kubectl -n <namespace> port-forward svc/myminio-console 9443:9443
```

---

### **Create your S3 MinIO Endpoint with MinIO-Operator**

Reference: **[Deploy and Manage MinIO Tenants](https://min.io/docs/minio/kubernetes/upstream/operations/deploy-manage-tenants.html)**

#### **Requirements**
  - You NEED a StorageClass defined
    - examples like Longhorn, Local-path, etc.
  - Your JWT login token to access the UI ingress or exposed port
  - Loadbalancer services ready to handle loadbalancer request (this could be ClusterIP if you wanted to lock to cluster only related resources)
  - If you paid for MinIO, this is where you supply your registry key in the Operator UI.

---

1. Inside your operator UI, click `Create Tenant` in the upper right hand corner.
2. Since there are numerous options to choose from, we are going to stick to the MANDATORY fields with the "`*`" next to it.
   - Required fields in the `Setup` tab:
     1. Name
     2. Namespace (you have to be specific to a "known" created namespace)
     3. StorageClass
     4. Number of Servers (how man pods you want to serve S3)
     5. Drives per Server (How many "volumes" to make per server)
     6. Erasure Code Parity (EC:4 is the default but for demo purposes, you can assign EC:0 (No fault tolerance)) but only AFTER you created the tenant. You will have to go back into your tenant and edit the parameter. (bug)
        - More about Erasure Code Parity here: **[Erasure Coding](https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html)**
     7. Optional: Click on the `Identity Provider` tab and you can assign a username / password for your s3 storage console you're about to build. Otherwise, they will be automatically generated.
3. Once you setup these values, and click `Create` in the lower right hand corner and you will get a message that states `New Tenant Created`.  You can now download your .json formated key identifier. 
4. Your s3 minio pod should be provisioning in the desired namespace you specified. This also includes the name of the pods and you can validate your storage by checking your PVCs / PV relationships for your storageClass.
   - If your pod fails to start, check to ensure that your parity settings are configure properly to start the container.  Your container logs should let you know if they need to be adjusted.