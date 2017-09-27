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
   sudo mv bosh-cli-2.0.28-linux-amd64 /usr/local/bin/bosh
   sudo chmod +x /usr/local/bin/bosh
   ```

1. Deploy BOSH director from jumpbox. See here: http://bosh.io/docs/init-google.html

   ```
   mkdir -p ~/deployments/bosh-101
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
1. Create `bosh-101-vars.yml` file with BOSH deployment variables (available in lastpass).

1. Deploy VMs with BOSH + warden CPI to that BOSH director:
   ```
   bosh create-release
   bosh -e bosh-101 upload-release

   ./scripts/deploy-classroom 2
   ```
   * where 2 is the number of students, VMs with BOSH installed on them.

1. Right before the class on a jumpbox VM (`bosh-bastion`) create a temporary user:

   ```
   sudo useradd --create-home -e 2013-07-30 jumpbox
   passwd jumpbox
   ```
   
   * where `2013-07-30` is the expiration date of user.
   * set password to something that will be shared during the class (this VM is open to public, so make it hard, but something users can retype).

1. Update /etc/ssh/sshd_config:
   ```
   PasswordAuthentication yes 
   ```
   and then restart the sshd:
   ```
   sudo service ssh restart
   ```
   Make sure to switch this back after the class or do full cleanup (see [After the class](README.md#after-the-class) section below).
   
   Disable sshguard that will block user after N failed attempts for 15 mins:
   
   ```
   sudo service sshguard stop
   ```

1. Save jumpbox SSH private key on jumpbox VM (from `bosh-101-vars.yml` from lastpass), will be used to SSH to students sandbox VM: 

   ```
   sudo -u jumpbox mkdir /home/jumpbox/.ssh
   sudo -u jumpbox vim /home/jumpbox/.ssh/jumpbox.key
   sudo chmod 600 /home/jumpbox/.ssh/jumpbox.key
   ```   

1. Now students can SSH to their BOSH sandbox VM (bosh-lite with CLI) (e.g. 10.0.0.4) as following:

   ```
   ./scripts/ssh <JUMPBOX_IP> 10.0.0.4
   ```

## After the class

On jumpbox:

```
bosh -e bosh-101 -d bosh-101-classroom delete-deployment
```
   
It is advisable to cleanup your whole GCP environment following these instructions: https://github.com/cloudfoundry-incubator/bosh-google-cpi-release/blob/master/docs/bosh/README.md#delete-resources
   
It should be easy and fast to spin up new environment following this guide (~15 mins). If you stuck somewhere please open an issue. This guide might become outdates so contributions are welcome.
