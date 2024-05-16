# Kubernetes Cluster Monitoring via Kubernetes Service Discovery with External Prometheus Server

This repository provides the steps to monitor a Kubernetes (K8S) cluster using Kubernetes Service Discovery (kubernetes_sd) with an external Prometheus server. The setup includes configuring `kubernetes_sd` with `role: endpoints` and `role: nodes`, utilizing the `/api/v1/nodes/<node-name>/proxy/metrics/cadvisor` endpoint.

## Components Used
- **Local Kubernetes Cluster**: Set up using k3s with 2 nodes.
- **Prometheus Server**: Hosted on Rocky Linux 9 with Grafana installed.
- **Kubernetes Service Discovery (kubernetes_sd)**: Used for Prometheus service discovery.

## Setup Steps

1. **Copy Certificates to Prometheus Server**
   - Transfer the CA certificate and a privileged user certificate and key to the Prometheus server using SFTP:
     - `ca.crt`
     - `client-admin.crt`
     - `client-admin.key`
   - Place these files in the `/etc/prometheus` directory on the Prometheus server.

2. **Change Ownership of Certificate Files**
   - Change the owner of the certificate files to the Prometheus user:
     ```bash
     chown prometheus /etc/prometheus/ca.crt /etc/prometheus/client-admin.crt /etc/prometheus/client-admin.key
     ```

3. **Edit Prometheus Configuration**
   - Edit the Prometheus configuration file located at `/etc/prometheus/prometheus.yml` to include the necessary `kubernetes_sd` configuration.

4. **Restart Prometheus**
   - Restart the Prometheus service to apply the changes:
     ```bash
     systemctl restart prometheus
     ```

5. **Import Grafana Dashboard**
   - Import the Grafana dashboard with the ID `15282` to visualize the metrics.

## Example Prometheus Configuration Snippet

```yaml
  - job_name: "kubernetes-nodes"
    kubernetes_sd_configs:
      - api_server: https://192.168.184.152:6443
        role: node
        tls_config:
          ca_file: "/etc/prometheus/server-ca.crt"
          cert_file: "/etc/prometheus/client-admin.crt"
          key_file: "/etc/prometheus/client-admin.key"
          insecure_skip_verify: true
    scheme: https
    tls_config:
      ca_file: "/etc/prometheus/server-ca.crt"
      cert_file: "/etc/prometheus/client-admin.crt"
      key_file: "/etc/prometheus/client-admin.key"
      insecure_skip_verify: true
    relabel_configs:
    - action: labelmap
      regex: __meta_kubernetes_node_label_(.+)
    - target_label: __address__
      replacement: "192.168.184.152:6443"
    - source_labels:
      - __meta_kubernetes_node_name
      regex: (.+)
      target_label: __metrics_path__
      replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

  - job_name: 'kubernetes-apiservers'
    kubernetes_sd_configs:
    - role: endpoints
      api_server: https://192.168.184.152:6443
      tls_config:
        ca_file: "/etc/prometheus/server-ca.crt"
        cert_file: "/etc/prometheus/client-admin.crt"
        key_file: "/etc/prometheus/client-admin.key"
        insecure_skip_verify: true
    scheme: https
    tls_config:
      ca_file: "/etc/prometheus/server-ca.crt"
      cert_file: "/etc/prometheus/client-admin.crt"
      key_file: "/etc/prometheus/client-admin.key"
      insecure_skip_verify: true
    relabel_configs:
    - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
      action: keep
      regex: default;kubernetes;https
