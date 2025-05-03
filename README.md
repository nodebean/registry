# 📦 Private Docker Registry with MinIO Backend on Kubernetes

This guide outlines the deployment of a private Docker Registry within a Kubernetes cluster, utilizing MinIO as the S3-compatible storage backend. The setup incorporates Istio for network security and routing, ensuring controlled access and efficient management.

---

## 🗂️ Kubernetes Configuration Overview

### 1. **Namespace**

* **File**: `namespace.yaml`
* **Purpose**: Establishes the `registry` namespace to logically separate and manage the Docker Registry components within the Kubernetes cluster.

### 2. **Deployment**

* **File**: `deployment.yaml`
* **Purpose**: Deploys the Docker Registry container.
* **Key Features**:

  * **Image**: Utilizes the official `registry:2` Docker image.
  * **Environment Variables**: Sources S3 credentials from a Kubernetes `Secret` named `s3-credentials`.
  * **Configuration**: Mounts a custom `config.yml` from the `registry-config` ConfigMap to `/etc/docker/registry/config.yml`.

### 3. **Service**

* **File**: `service.yaml`
* **Purpose**: Exposes the Docker Registry internally within the cluster on port `5000`, facilitating communication between services.

### 4. **ConfigMap**

* **File**: `registry-cm.yaml`
* **Purpose**: Provides the Docker Registry with its configuration settings.
* **Highlights**:

  * **Storage Backend**: Configures MinIO as the S3-compatible storage backend.
  * **Connection Details**: Specifies the MinIO endpoint, region, and bucket name.
  * **Security Settings**: Disables SSL (`secure: false`) for internal communication; ensure this aligns with your security requirements.

### 5. **AuthorizationPolicy (Istio)**

* **File**: `authorization-policy.yaml`
* **Purpose**: Implements network-level access control.
* **Details**:

  * **Access Restriction**: Allows only traffic originating from the `10.10.0.0/16` subnet to access the Docker Registry service within the `registry` namespace.

### 6. **HTTPRoute (Istio Gateway API)**

* **File**: `httproute.yaml`
* **Purpose**: Defines external HTTP routing to the Docker Registry.
* **Configuration**:

  * **Hostname**: Routes requests from `registry.fava.casa`.
  * **Path Matching**: Directs all paths (`/`) to the Docker Registry service on port `5000`.

### 7. **Kustomization**

* **File**: `kustomization.yaml`
* **Purpose**: Aggregates all Kubernetes manifests for streamlined deployment using `kubectl kustomize` or similar tools.

---

## 🗄️ MinIO Configuration for Docker Registry

To integrate MinIO as the storage backend for the Docker Registry, ensure the following configurations are in place:

### 1. **Docker Registry Configuration (`config.yml`)**

Located within the `registry-config` ConfigMap, this file should include:

```yaml
version: 0.1
log:
  fields:
    service: registry
http:
  addr: :5000
storage:
  cache:
    layerinfo: inmemory
  s3:
    region: us-east-1
    regionendpoint: http://minio.fava.casa:9000
    bucket: registry
    encrypt: false
    secure: false
    v4auth: true
    chunksize: 5242880
    rootdirectory:
```

**Note**: Replace `http://minio.fava.casa:9000` with your MinIO service endpoint if different. Ensure that the `bucket` specified (`registry`) exists in MinIO.

### 2. **MinIO Bucket Policy**

To grant the Docker Registry full access to the `registry` bucket, apply the following bucket policy in MinIO:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*"
      ],
      "Resource": [
        "arn:aws:s3:::registry",
        "arn:aws:s3:::registry/*"
      ]
    }
  ]
}
```

**Implementation Steps**:

1. **Access MinIO Console**: Navigate to your MinIO web interface (e.g., `http://minio.fava.casa:9000`).

2. **Login**: Use your MinIO root credentials.

3. **Navigate to Buckets**: Select the `registry` bucket.

4. **Edit Bucket Policy**: Apply the above JSON policy to grant necessary permissions.

**Security Consideration**: Granting `s3:*` permissions provides full access to the bucket. For enhanced security, consider restricting actions to only those necessary for the Docker Registry's operation, such as `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`, etc.

---

## 🚀 Deployment Instructions

1. **Prepare Secrets**:

   * Create a Kubernetes `Secret` named `s3-credentials` containing your MinIO access and secret keys.

   ```bash
   kubectl create secret generic s3-credentials \
     --from-literal=REGISTRY_STORAGE_S3_ACCESSKEY=<your-access-key> \
     --from-literal=REGISTRY_STORAGE_S3_SECRETKEY=<your-secret-key> \
     -n registry
   ```

   Replace `<your-access-key>` and `<your-secret-key>` with your actual MinIO credentials.

2. **Apply Kubernetes Manifests**:

   * Deploy all resources using Kustomize:

   ```bash
   kubectl apply -k .
   ```

   Ensure you're in the directory containing the `kustomization.yaml` file.

3. **Verify Deployment**:

   * Check the status of the deployed pods:

   ```bash
   kubectl get pods -n registry
   ```

   * Confirm that the Docker Registry is accessible via the specified hostname (`registry.fava.casa`) and port (`5000`).

---

## 🔐 Security Recommendations

* **TLS Encryption**: Currently, the configuration disables SSL (`secure: false`). For production environments, it's recommended to enable TLS to encrypt data in transit. This involves configuring TLS certificates in both MinIO and the Docker Registry.

* **Access Control**: Review and adjust the MinIO bucket policy to adhere to the principle of least privilege, granting only necessary permissions to the Docker Registry.

* **Authentication**: Implement authentication mechanisms for accessing the Docker Registry, such as HTTP basic authentication or token-based authentication, to prevent unauthorized access.

---

By following this guide, you can successfully deploy a secure and efficient private Docker Registry backed by MinIO within your Kubernetes cluster.
