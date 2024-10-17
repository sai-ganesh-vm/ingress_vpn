
# Configuring NGINX Ingress Controller for Multiple Applications with VPN-Based IP Restriction

## 1. Introduction

This document explains how to configure a **single NGINX Ingress Controller** to manage multiple applications. In this setup:
- **Application A**: Open to public traffic from any IP address.
- **Application B**: Restricted to users connected through a **VPN** that assigns a static public IP.
- The VPN-assigned IP is **whitelisted at the Ingress level**, ensuring secure access to Application B, without needing security group rules.

By using the NGINX Ingress Controller and appropriate ingress resources, this approach avoids the complexity of configuring network-level security groups and offers more flexible application-level access control.

---

## 2. How to Do It

The solution involves the following steps:
1. **Deploy NGINX Ingress Controller**: This controller will manage all traffic for multiple applications.
2. **Create Kubernetes Deployments and Services**: For each application to expose them within the cluster.
3. **Define Separate Ingress Resources**:
   - **Application A**: Accessible by the public.
   - **Application B**: Accessible only via the VPN IP, using the `nginx.ingress.kubernetes.io/whitelist-source-range` annotation.

This setup ensures each application is managed by the same Ingress Controller but with **different access rules applied at the Ingress Resource level**.

---

## 3. This Method vs. Alternatives

| **Approach**                              | **Description**                                           | **Drawbacks**                                              |
|-------------------------------------------|-----------------------------------------------------------|------------------------------------------------------------|
| **Ingress Resource with VPN IP Whitelisting** (This Method) | Control access via NGINX Ingress annotations for VPN-restricted access to specific applications. | Requires a reliable VPN with a fixed public IP. |
| **Security Group Restrictions**           | Use network-level rules to limit access to a specific IP.  | Complex management if the VPN IP changes frequently.       |
| **Separate Ingress Controllers**          | Deploy multiple controllers for different apps.           | Higher resource consumption and management overhead.       |

This method offers a **simple and centralized approach** by using only one Ingress Controller to handle different access needs.

---

## 4. Advantages of This Method
- **Single Ingress Controller**: Reduces operational overhead and resource consumption.
- **Application-Level Control**: IP whitelisting is managed at the Ingress Resource level, offering flexibility.
- **VPN-Only Access without Security Group Rules**: No need to manage network-level restrictions or modify security groups.
- **Scalable and Maintainable**: Easily extendable to new applications by adding more ingress resources.

---

## 5. Step-by-Step Implementation

### **Step 1: Install NGINX Ingress Controller**

Install NGINX Ingress Controller using Helm.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
```

Verify the installation:

```bash
kubectl get pods -n ingress-nginx
```

### **Step 2: Create Deployments and Services for the Applications**

**Deployment and Service for Application A (Public)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-a
  template:
    metadata:
      labels:
        app: app-a
    spec:
      containers:
        - name: app-a-container
          image: nginx:latest
          ports:
            - containerPort: 80
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-a-service
  namespace: default
spec:
  selector:
    app: app-a
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

**Deployment and Service for Application B (VPN Restricted)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-b
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-b
  template:
    metadata:
      labels:
        app: app-b
    spec:
      containers:
        - name: app-b-container
          image: nginx:latest
          ports:
            - containerPort: 80
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-b-service
  namespace: default
spec:
  selector:
    app: app-b
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### **Step 3: Create Ingress Resources**

**Ingress Resource for Application A (Open to Public)**:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-a-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: app-a.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-a-service
                port:
                  number: 80
```

**Ingress Resource for Application B (VPN Restricted)**:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-b-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/whitelist-source-range: "192.168.1.10/32"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: app-b.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-b-service
                port:
                  number: 80
```

### **Step 4: Apply the Configurations**

```bash
kubectl apply -f app-a-deployment.yaml
kubectl apply -f app-b-deployment.yaml
kubectl apply -f app-a-service.yaml
kubectl apply -f app-b-service.yaml
kubectl apply -f app-a-ingress.yaml
kubectl apply -f app-b-ingress.yaml
```

### **Step 5: Verify the Setup**

1. **Check Ingress Resources**:

   ```bash
   kubectl get ingress
   ```

2. **Test Access**:
   - **Application A**: Accessible from any IP at `http://app-a.example.com`.
   - **Application B**: Accessible only via the VPN-assigned IP at `http://app-b.example.com`.

3. **Monitor Logs for Troubleshooting**:

   ```bash
   kubectl logs -f <nginx-ingress-pod-name> -n ingress-nginx
   ```

---

## 6. Conclusion

This setup allows you to manage multiple applications with different access requirements using a **single NGINX Ingress Controller**. It ensures:
- **Application A** is accessible to everyone.
- **Application B** is securely restricted to users connected through a VPN, without requiring security group rules.

The approach simplifies management by centralizing control at the application layer, ensuring flexible and scalable traffic management while keeping the setup lightweight.

