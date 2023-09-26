# **`Install MINIO-Operator`**

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

* Clone from repo1 if needed:

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
---------

* Update replica count
```sh
kubectl get deployment minio-operator -n minio-operator -o yaml > operator.yaml
yq -i -e '.spec.replicas |= 1' operator.yaml
```
---------

* Create your Cluster CA Issuer
  - Note: Using Self-Signed certificates for this example but custom signing CA certificate can be used to distribute your certificates.

- Install **[Cert-manager](https://https://cert-manager.io/docs/installation/helm/)** and create a self-signed Cluster issuer CA

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

---------

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

---------

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

