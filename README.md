# BOSH-101

## BOSH-101 Architecture

BOSH-101 environment is deployed to GCP. It is managed by single BOSH director. 

That director is accessible via jumpbox (`bosh-bastion`) created via terraform.

Bosh-101 director manages `bosh-101-classroom` deployment that consists of `N` VMs, where `N` is the number of students in classroom. Each VM has pre-installed local BOSH director with Warden CPI as well as downloaded BOSH CLI binary, simple release and stemcell.

## Deploying BOSH-101

1. Prepare GCP environment and deploy jumpbox (`bosh-bastion`). See: https://github.com/cloudfoundry-incubator/bosh-google-cpi-release/blob/master/docs/bosh/README.md

   **NOTE**: Replace `hashicorp/terraform:light` with `hashicorp/terraform:0.9.9` as per this issue: https://github.com/cloudfoundry-incubator/bosh-google-cpi-release/issues/222
   
1. SSH to jumpbox
  ```
  gcloud compute ssh bosh-bastion
  mkdir -p ~/workspace
  cd ~/workspace
  git clone https://github.com/mariash/bosh-101-release
  git clone https://github.com/cloudfoundry/bosh-deployment
  ```

1. Get latest bosh cli: http://bosh.io/docs/cli-v2.html

   ```
   wget https://s3.amazonaws.com/bosh-cli-artifacts/bosh-cli-2.0.28-linux-amd64

   sudo gem uninstall bosh_cli
   sudo rm /usr/bin/bosh2
   mv bosh-cli-2.0.28-linux-amd64 /usr/local/bin/bosh
   chmod +x /usr/local/bin/bosh
   ```

1. Deploy BOSH director from jumpbox. See here: http://bosh.io/docs/init-google.html

   ```
   cd bosh-101-release
   
   ./scripts/create-env
   ```

1. Update cloud config:

   ```
   ./scripts/update-cloud-config
   ```
1. Upload latest stemcell:

   ```
   bosh -e bosh-101 upload-stemcell https://bosh.io/d/stemcells/bosh-google-kvm-ubuntu-trusty-go_agent
   ```

1. Deploy VMs with BOSH + warden CPI to that BOSH director:
   ```
   bosh create-release
   bosh -e bosh-101 upload-release

   ./scripts/deploy-classroom 2
   ```
   * this script requires `bosh-101-vars.yml` file to exists with the list of deployment variables (available in lastpass).
   * where 2 is the number of students, VMs with BOSH installed on them.

1. Save jumpbox SSH private key on jumpbox VM (from lastpass in `bosh-101-vars.yml`): 

   ```
   vim ~/.ssh/jumpbox.key
   chmod 600 ~/.ssh/jumpbox.key
   ```

1. To ssh to BOSH-lite VM from your anywhere (e.g. 10.0.0.4):

   ```
   GATEWAY_USER=<BASTION_USER> GATEWAY_SSH_KEY=<BASTION_SSH_KEY> GATEWAY_HOST=<BASTION_IP> ./scripts/ssh 10.0.0.4
   ```
