# provision-vmware-vm-instances
This is a collection of playbooks to help provision VMs on VMWare Vcenter.
Initially it is focused on provisioining a bastion instance and installing various clients as well as configuring the bastion. It can also be use to customize (e.g extend) storage for the instance. 

## Usage
To run the various playbooks to provision and configure a bastion host use the following steps.

1. From separate machine already configured to connect to the vcenter, clone the repository using 
```
git clone https://github.com/cadjai/provision-vmware-vm-instances.git && cd provision-vmware-vm-instances
```

2. From within the cloned repository directory run the main playbook to provision the bastion VM instance using
```
ansible-playbook --ask-vault-pass -i inventory -e terraform_dir=~/provision-vmware-vm-instances/ clone-vsphere-bastion-vm-from-template.yml 
```

3. Once the bastion is provisioned use the vcenter web console for the created VM instance to login to the bastion to continue to configure it

4. On the bastion install ansible and git manually to enable the ability to run the remaining playbooks. This step is being done this way because the VM does not have ssh enabled by default so that we can run the remaining playbooks remotely. This is probably because the current template used is a RHELCSB template.
```
sudo yum install -y git ansible
```

5. After installing the two packages above proceed to cloning the same repository onto the VM instance using
```
mkdir -p $HOME/repos && cd $HOME/repos && git clone https://github.com/cadjai/provision-vmware-vm-instances.git && cd provision-vmware-vm-instances
```

6. Now run the various playbooks to configure the bastion using the following 
```
ansible-playbook --ask-vault-pass -i inventory configure-bastion-vm.yml
ansible-playbook --ask-vault-pass -i inventory config-local-vsphere-environment-variables.yml 
ansible-playbook --ask-vault-pass -i inventory install-clients.yml
ansible-playbook --ask-vault-pass -i inventory config-local-vsphere-terraform-provider.yml
```

If you need to run the various plaubooks adhoc you can use these.

To create a VM template from an ova template use the following playbook
```
ansible-playbook --ask-vault-pass -i inventory -vvv configure-vsphere-template.yml
```
To clone a bastion VM from a template use the following playbook
```
ansible-playbook --ask-vault-pass -i inventory -vvv clone-vsphere-bastion-vm-from-template.yml
```
To clone a non bastion VM from a template (e.g. OpenShift UPI instances) use the following playbook
```
ansible-playbook --ask-vault-pass -i inventory -vvv clone-vsphere-vm-from-template.yml
```
TO install various clients on the bastion use the following
```
ansible-playbook --ask-vault-pass -i inventory -vvv install-clients.yml
```
To configure the bastion to use the govc API to communicate with the VCenter usethe following
```
ansible-playbook --ask-vault-pass -i inventory -vvv config-local-vsphere-environment-variables.yml
```
To configure the bastion to use terraform and terraform provider to provision resources against VCenter use the following
```
ansible-playbook --ask-vault-pass -i inventory -vvv config-local-vsphere-terraform-provider.yml 
```


## Roadmap

## Contributing

## License

# Notes from high side deployment process

## Old Cluster Cleanup

### Destroy the existing VMs
```bash
govc vm.destroy bootstrap
govc vm.destroy master-0
govc vm.destroy master-1
govc vm.destroy master-0
```
### Remove old deployment files
```bash
rm -rf /root/platform/cluster
mkdir -p /root/platform/cluster
```
## New Cluster Deployment

### Create new deployment files
```bash
cp /root/UPI-install-config.yml /root/platform/cluster/install-config.yml
openshift-install create manifests --dir=/root/platform/cluster/
```
### Clean up files
```bash
ansible-playbook /home/maintuser/ocp4/clean-manifests.yml

rm -f /root/platform/cluster/openshift/99_openshift-cluster-api_master-machines*.yaml
rm -f /root/platform/cluster/openshift/99_openshift-cluster-api_worker-machineset.yaml
```

### Create ignition configs and convert to base64
```bash
openshift-install create ignition-configs --dir=/root/platform/cluster/

cp /root/platform/cluster/*.ign /root/platform/nginx
cp /root/merge-bootstrap.ign /root/platform/cluster

base64 -w0 /root/platform/cluster/merge-bootstrap.ign > /root/platform/cluster/merge-bootstrap.64
base64 -w0 /root/platform/cluster/master.ign > /root/platform/cluster/master.64
base64 -w0 /root/platform/cluster/worker.ign > /root/platform/cluster/worker.64
```

### Create new VMs via CLI
```bash
govc vm.clone -vm=template-rhcos -folder'$(DataCenter)/vm/DPaaS' -ds=$(DataStore) -dc=$(DataCenter) -on=false bootstrap
govc vm.clone -vm=template-rhcos -folder'$(DataCenter)/vm/DPaaS' -ds=$(DataStore) -dc=$(DataCenter) -on=false master-0
govc vm.clone -vm=template-rhcos -folder'$(DataCenter)/vm/DPaaS' -ds=$(DataStore) -dc=$(DataCenter) -on=false master-1
govc vm.clone -vm=template-rhcos -folder'$(DataCenter)/vm/DPaaS' -ds=$(DataStore) -dc=$(DataCenter) -on=false master-2
```
### Modify the VMs (using base64 data from previous step)
```bash
govc vm.change -vm bootstrap -e "guestinfo.ignition.config.data.encoding=base64" -e "disk.EnableUUID=TRUE" -e "guestinfo.ignition.config.data=$(Data from /root/platform/cluster/merge-bootstrap.64)
govc vm.change -vm master-0 -e "guestinfo.ignition.config.data.encoding=base64" -e "disk.EnableUUID=TRUE" -e "guestinfo.ignition.config.data=$(Data from /root/platform/cluster/master.64)
govc vm.change -vm master-1 -e "guestinfo.ignition.config.data.encoding=base64" -e "disk.EnableUUID=TRUE" -e "guestinfo.ignition.config.data=$(Data from /root/platform/cluster/master.64)
govc vm.change -vm master-2 -e "guestinfo.ignition.config.data.encoding=base64" -e "disk.EnableUUID=TRUE" -e "guestinfo.ignition.config.data=$(Data from /root/platform/cluster/master.64)
```

### Set the static IP config for the VMs
```bash
export IPCFG="ip=$(Bootstrap-IP)::$(Gateway):$(Netmask):$(hostname)::none nameserver=$(nameserver-1) [nameserver=$(nameserver-2)]"
govc vm.change -vm bootstrap -e "guestinfo.afterburn.initrd.network-kargs=${IPCFG}"

export IPCFG="ip=$(Master-0-IP)::$(Gateway):$(Netmask):$(hostname)::none nameserver=$(nameserver-1) [nameserver=$(nameserver-2)]"
govc vm.change -vm master-0 -e "guestinfo.afterburn.initrd.network-kargs=${IPCFG}"

export IPCFG="ip=$(Master-1-IP)::$(Gateway):$(Netmask):$(hostname)::none nameserver=$(nameserver-1) [nameserver=$(nameserver-2)]"
govc vm.change -vm master-1 -e "guestinfo.afterburn.initrd.network-kargs=${IPCFG}"

export IPCFG="ip=$(Master-2-IP)::$(Gateway):$(Netmask):$(hostname)::none nameserver=$(nameserver-1) [nameserver=$(nameserver-2)]"
govc vm.change -vm master-2 -e "guestinfo.afterburn.initrd.network-kargs=${IPCFG}"
```

### Power on the VMs
```bash
govc vm.power -on bootstrap
govc vm.power -on master-0
govc vm.power -on master-1
govc vm.power -on master-2
```





