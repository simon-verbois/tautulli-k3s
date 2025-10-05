# Deploying Tautulli on K3s/Kubernetes

This repository contains a set of Kubernetes manifests for deploying [Tautulli](https://tautulli.com/), a popular monitoring and tracking tool for Plex Media Server, on a K3s cluster (tested on K3s with single node) (or any other Kubernetes cluster).

The configuration is designed to be simple, clean, and easily maintainable.

## Features

- Simple deployment via `kubectl apply`.
- Data persistence managed by a `PersistentVolumeClaim`.
- Centralized configuration in a `ConfigMap`.
- Secure exposure via an `Ingress` (tested with Traefik).

## Prerequisites

1.  A working Kubernetes cluster (e.g., K3s, k0s, RKE2).
2.  The `kubectl` command-line tool configured to access your cluster.
3.  An **Ingress Controller** installed in the cluster (e.g., Traefik, NGINX Ingress).
4.  A **StorageClass** configured to dynamically provision storage volumes. K3s includes `local-path-provisioner`, which works for this configuration.
5.  A running Plex Media Server instance that Tautulli can connect to.

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/simon-verbois/tautulli-k3s
cd tautulli-k3s
```

### 2. Copy the files

Make your own copy of the templates files with this command.

```bash
for file in *.yaml.template; do mv "$file" "${file%.template}"; done
```

### 3. Customize the configuration

You **must** adapt some files to your own environment before applying them.

- **`02-configmap.yaml`**:
    - Change the time zone `TZ` if needed (e.g., `"America/New_York"`).
    - Adjust `PUID` and `PGID` to match the user/group ID that owns your configuration files on the persistent storage. This helps avoid permission issues.

    ```yaml
    data:
      TZ: "Europe/Paris" # <-- EDIT THIS
      PUID: "YOUR-USER-UID"        # <-- EDIT THIS
      PGID: "YOUR-USER-GID"        # <-- EDIT THIS
    ```

<br>

- **`03-deployment.yaml`**:
    - Adjust the `fsGroup` in `securityContext` to match the `PGID` you set in the `ConfigMap`.

    ```yaml
    # ...
    spec:
      securityContext:
        fsGroup: YOUR-USER-GID # <-- EDIT THIS to match your PGID
    # ...
    ```

<br>

- **`05-ingress.yaml`**:
    - Modify the `host` to use your own domain name.
    - Adapt the ingress `annotations` for your Ingress Controller. The example uses Traefik and assumes your TLS certificate resolver is named `ovhresolver`. Change this to match your setup.

    ```yaml
    metadata:
      # ...
      annotations:
        traefik.ingress.kubernetes.io/router.tls.certresolver: your-certresolver-name # <-- EDIT THIS
    spec:
      rules:
      - host: "tautulli.your-domain.com" # <-- EDIT THIS
        # ...
      tls:
      - hosts:
        - "tautulli.your-domain.com" # <-- EDIT THIS
    ```

### 4. Deploy Tautulli

Apply all YAML manifests in a single command from the project root:

```bash
kubectl apply -f .
```

This will create the `tautulli` namespace, the PersistentVolumeClaim, the ConfigMap, the Deployment, the Service, and the Ingress.

### 5. Access Tautulli

After a few moments, the container image will be downloaded and the pod will start. You should then be able to access your Tautulli instance via the URL you configured in the ingress (e.g., `https://tautulli.your-domain.com`).

The initial setup, including connecting Tautulli to your Plex Media Server, will be done through this web interface.

## Maintenance

### Updating the Tautulli Image

The deployment uses the `lscr.io/linuxserver/tautulli:latest` image. To update to the latest version, you can trigger a rolling update of the deployment, which will force Kubernetes to pull the newest image.

```bash
kubectl rollout restart deployment/tautulli-deployment -n tautulli
```

You can monitor the progress of the update with:

```bash
kubectl rollout status deployment/tautulli-deployment -n tautulli
```

## Uninstallation

To remove all the resources created by these manifests, run the following command:

```bash
kubectl delete -f .
```

## License

This project is licensed under the MIT License. See the `LICENSE` file for details.