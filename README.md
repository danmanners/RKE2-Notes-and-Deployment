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

Not actually sure this works the way I think it does, so let's ignore this for now.

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

## RKE Server - Hacky Install (Doesn't quite work)

```bash
# Install RKE2 on all 5 hosts
for ip in 10.45.0.24{1..5}; do
    # Install RKE2 On the system
    ssh ubuntu@$ip -qt 'curl -sfL https://get.rke2.io | sudo sh -';
done

# Setup the First Master
for ip in 10.45.0.241; do
    # Copy over the relevant files
    scp -q config/server/master-config.yaml ubuntu@$ip:/tmp/config.yaml
    ssh ubuntu@$ip -qt \
        "sudo mkdir -p /var/lib/rancher/rke2/ && \
        sudo mv /tmp/config.yaml /etc/rancher/rke2/config.yaml && \
        sudo chown root:root /etc/rancher/rke2/config.yaml && \
        sudo chmod 0600 /etc/rancher/rke2/config.yaml && \
        sudo sed -i \"s/master-00/master-0$(echo ${ip: -1})/g\" /etc/rancher/rke2/config.yaml && \
        sudo systemctl enable --now rke2-server";
done

# Get the master token
ROOT_NODE_TOKEN="$(ssh ubuntu@10.45.0.241 -qt 'sudo cat /var/lib/rancher/rke2/server/node-token')"

# Setup the 2nd and 3rd masters
for ip in 10.45.0.24{2..3}; do
    # Copy over the relevant files
    scp -q config/server/ha-node-config.yaml ubuntu@$ip:/tmp/config.yaml
    ssh ubuntu@$ip -qt \
        "sudo mkdir -p /var/lib/rancher/rke2/ && \
        sudo mv /tmp/config.yaml /etc/rancher/rke2/config.yaml && \
        sudo chown root:root /etc/rancher/rke2/config.yaml && \
        sudo chmod 0600 /etc/rancher/rke2/config.yaml && \
        sudo sed -i \"s/0X/0$(echo ${ip: -1})/g\" /etc/rancher/rke2/config.yaml && \
        sudo systemctl enable --now rke2-server";
done

# Setup the 2nd and 3rd masters
for ip in 10.45.0.24{2..3}; do
    # Copy over the relevant files
    scp -q config/server/ha-node-config.yaml ubuntu@$ip:/tmp/config.yaml
    ssh ubuntu@$ip -qt \
        "sudo mkdir -p /var/lib/rancher/rke2/ && \
        sudo mv /tmp/config.yaml /etc/rancher/rke2/config.yaml && \
        sudo chown root:root /etc/rancher/rke2/config.yaml && \
        sudo chmod 0600 /etc/rancher/rke2/config.yaml && \
        sudo sed -i \"s/0X/0$(echo ${ip: -1})/g\" /etc/rancher/rke2/config.yaml && \
        sudo systemctl enable --now rke2-server";
done
```

Still To-Do:

* Need to add one more line to replace the `"Fake::server:token"` string in the config files with the actual token from the Primary Master.
