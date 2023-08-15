# awx-k3s

# Disable firewalld
sudo systemctl disable firewalld --now

# Disable nm-cloud-setup if exists and enabled
sudo systemctl disable nm-cloud-setup.service nm-cloud-setup.timer
sudo reboot

sudo dnf install -y git curl

curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644

kubectl apply -k operator

export AWX_HOST="awx.example.com"

openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -out ./base/tls.crt -keyout ./base/tls.key -subj "/CN=${AWX_HOST}/O=${AWX_HOST}" -addext "subjectAltName = DNS:${AWX_HOST}"

Modify hostname in base/awx.yaml.

...
spec:
  ...
  ingress_type: ingress
  ingress_tls_secret: awx-secret-tls
  hostname: awx.example.com     👈👈👈
...

Modify two passwords in base/kustomization.yaml. Note that the password under awx-postgres-configuration should not contain single or double quotes (', ") or backslashes (\) to avoid any issues during deployment, backup or restoration.

...
  - name: awx-postgres-configuration
    type: Opaque
    literals:
      - host=awx-postgres-13
      - port=5432
      - database=awx
      - username=awx
      - password=Ansible123!     👈👈👈
      - type=managed

  - name: awx-admin-password
    type: Opaque
    literals:
      - password=Ansible123!     👈👈👈
...

Prepare directories for Persistent Volumes defined in base/pv.yaml. These directories will be used to store your databases and project files. Note that the size of the PVs and PVCs are specified in some of the files in this repository, but since their backends are hostPath, its value is just like a label and there is no actual capacity limitation.

sudo mkdir -p /data/postgres-13

sudo mkdir -p /data/projects

sudo chmod 755 /data/postgres-13

sudo chown 1000:0 /data/projects

Deploy AWX, this takes few minutes to complete.

kubectl apply -k base

To monitor the progress of the deployment, check the logs of deployments/awx-operator-controller-manager:

kubectl -n awx logs -f deployments/awx-operator-controller-manager


When the deployment completes successfully, the logs end with:

$ kubectl -n awx logs -f deployments/awx-operator-controller-manager
...
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=83   changed=0    unreachable=0    failed=0    skipped=79   rescued=0    ignored=1

