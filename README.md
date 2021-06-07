# RKE2 - Notes and Thoughts

The following is really a scratchpad for things that are probably important or that I want to look into.

## Important Default Paths and Files

```bash
# systemD
/usr/local/lib/systemd/system/

# RKE2
/var/lib/rancher/rke2/

# Kube Config
/etc/rancher/rke2/rke2.yaml
```

## Pre-Generating Auth Token

```bash
echo "K$(openssl rand -hex 33)::server:$(openssl rand -hex 16)" > token
Ke4aded7853f03c431571cfc8639ba229d788278a494cf7ca39b8452e28d27661c0::server:101272d6454076139e607b63b74b867d
```

## RKE Server - Important Arguments

```bash
--tls-san "10.45.0.241" --tls-san "rke2.k8s.danmanners.io"      # Relevant for High Availability
--cluster-domain rke2.homelab.local                             # Set a custom cluster-domain
--token-file value                                              # Loads a custom token from a file
--node-name $HOSTNAME                                           # Ensure that system hostnames don't have to match
--node-ip value                                                 # Relevant for ZeroTier/Remote clients
--disable rke2-ingress-nginx                                    # We don't want to use NGINX Ingress
--disable rke2-kube-proxy                                       # We don't want kube-proxy

# Networking
--cni none                                                      # Install Weave/Calico after startup, not built in

# ETCD S3 Backup
--etcd-s3                                                       # Enable S3 ETCD Backups
--etcd-s3-access-key value                                      # S3 Access Key Value
--etcd-s3-secret-key value                                      # S3 Secret Key Value
--etcd-s3-bucket value                                          # S3 Bucket Name
--etcd-s3-region value                                          # AWS Region; Defaults to 'us-east-1'
--etcd-s3-folder value                                          # S3 Directory to store data in
```

## RKE Agent - Important Arguments

```bash
--server value                                                  # High Availability Master DNS Name
--node-name $HOSTNAME                                           # Ensure that system hostnames don't have to match
--node-ip value                                                 # Relevant for ZeroTier/Remote clients
--node-external-ip value                                        # Relevant for cloud hosts
--selinux                                                       # Relevant for CentOS hosts
```

## RKE Server - Hacky Install

```bash
for i in 10.45.0.24{1..3}; do
    # Install RKE2 On the system
    ssh ubuntu@$i -qt 'curl -sfL https://get.rke2.io | sudo sh -'
    # Copy over the relevant files
    scp -q token ubuntu@$i:~/token
    scp -q systemd/rke2-server* ubuntu@$i:/tmp/
    ssh ubuntu@$i -qt \
        "sudo mkdir -p /var/lib/rancher/rke2/ && \
        sudo cp /home/ubuntu/token /var/lib/rancher/rke2/token && \
        sudo chown root:root /var/lib/rancher/rke2/token && \
        sudo chmod 0600 /var/lib/rancher/rke2/token && \
        sudo mv /tmp/rke2-server.* /usr/local/lib/systemd/system/ && \
        sudo sed -i \"s/master-00/master-0$(echo ${i: -1})/g\" /usr/local/lib/systemd/system/rke2-server.service && \
        sudo systemctl daemon-reload";
done
```
