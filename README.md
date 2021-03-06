# CycleCloud Multi-Region HPC Cluster
This repo contains instructions to create a CycleCloud cluster with Compute VMs in a separate Azure region.  For example, your HPC scheduler/head node could be in East US 2 Region with compute queues/partitions in East US 2, but also have compute queues/partitions in East US.  

![CC MultiRegion Overview](https://github.com/themorey/cyclecloud-multiregion-cluster/blob/main/images/ccmr_overview.png "CycleCloud MultiRegion Overview")  

## Why would someone want to span an HPC cluster across Azure Regions?  

- _**Capacity**_.  There are times when a single Azure Region can not accommodate the quantity of compute cores needed.  In this case the workload can be split across multiple regions to get access to all the compute cores requested.  
- _**Specialty Compute (ie. GPU, FPGA, Infiniband)**_.  An organization created an Azure environment in a specific Region (ie. East US 2) and later requires specialty compute VMs that are not available in that Region.  Examples of these are high end GPU VMs (ie. NDv4), FPGA VMs (ie. NP) or HPC VMs (ie. HB & HC).
- _**[Public Datasets](https://azure.microsoft.com/en-us/services/open-datasets/#overview)**_.  Azure hosts numerous Public Data Sets but they are specific to Regions (ie. West US 2).  You may have an HPC cluster and Data Lake configured in South Central US but need a compute queue/partition in West US 2 to make use of the Public Data Set (ie. US Labor Force Statistics).  

In the examples above, the standard answer is to create a new cluster in the Azure Region.  This repo provides an alternative option to have a single HPC cluster but utilize compute resources in multiple regions.  

# REQUIREMENTS
  
  ## Networking
 * [__VNET Peering__](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview):  The compute nodes in Region 2 need to communicate with the scheduler in Region 1, and vice versa.  The easiest way to accomplish this is to "Peer" the Azure VNETs so resources can communicate between them.  This can also be accomplished with explicit customer routing environments.  
 
 * __Name Resolution__:  Cluster nodes (ie. scheduler and compute) need to resolve each other by hostname.  Traditionally this has been managed by CycleCloud maintaining the `/etc/hosts` file on each node.  This will continue to work for most schedulers with some additons to the CycleCloud cluster template as described below in the Schedulers section.  Some cluster types, ie. Slurm, will make use of Azure DNS (with autoregistration and forward/reverse name lookup) instead of `/etc/hosts` in order to match the Slurm hostname to the Azure hostname.  However, for this to work across Azure Regions will require an [Azure Private DNS](https://docs.microsoft.com/en-us/azure/dns/private-dns-overview) zone and an update to the Network Manager search domain to use the assocaiated private name.

## Job Script Optimizations
* __Working Directory__:  By default the working directory of most schedulers is the directory from which a job is submitted.  Most likely this directory is the user's home directory but it can be specifically set by the job.  This is important for a multiregion cluster as the shared NFS home directory is likely in Region 1 collocated with the "primary" cluster.  If a job runnning in Region 2 writes updates to files in Region 1 across the WAN it will be slow.  It is recommended to use the local SSD on the VM (ie. `/mnt/resource` on CentOS/RHEL) and mv it to the home dir at the end of the job.  Some examples of setting the job working directory in the job script:  

| Scheduler     | Directive                        |
| ------------- |--------------------------------:| 
| SLURM         | #SBATCH --chdir /mnt/resource  | 
| GRIDENGINE    |  #$ -wd /mnt/resource         |  
| OPENPBS       |       N/A     |
| LSF           | -cwd "/mnt/resource          |  

* __Input/Output Files__:  Input and output files should be staged in the Region that runs the job in order to reduce time to run and minimize transfer costs.  

# SCHEDULER CONFIGS  
This section discusses configs applied to the Cluster template files for different schedulers.  The full template files are located in the `templates` directory of the repo.  

> ***NOTE***:  The CycleCloud UI forms will not allow you to configure a multiregion cluster.  All the Cluster Parameters normally configured in the UI *must* be hardcoded in the template file for a multiregion cluster.  Placeholder values are in the template files included in this repo but must updated with your specific information.


## GRIDENGINE
* GridEngine will use `/etc/hosts` (aka *cyclecloud.hosts.standalone_dns*) with multiple subnets and an alternate prefix
* Cluster Template changes:
  * Added a `cloud-init` section to open the permissions of `/mnt/resource`
  * Added configs to `node defaults` to specify subnets used and a common dns name (alt_suffix)
    * `cyclecloud.hosts.standalone_dns.alt_suffix = private.ccmr.net` can be set to any name (you can use the default name in the template)
    * `cyclecloud.hosts.standalone_dns.subnets = 10.40.0.0/24, 10.41.0.0/24` should list all subnets of the cluster in all Azure Regions
  * `SubnetId` and `Region` settings are moved from `[[node defaults]]` to each defined `node` and `nodearray`; all values are hardcoded in the template
    * `SubnetId` is of the format `resource-group-name/vnet-name/subnet-name` with a placeholder name in the template
    * `Region` can be determined by the azure-cli command `az account list-locations -o table`
  * Removed `parameters Region` and `parameters Networking` from the Parameters section as they are hardcoded in the sections above it
  * Hardcoded Default OS values to avoid issues with UI validations
  * Hardcode changes to VM types, core counts, etc
* Create new cluster named `GridEngine-Multiregion` as follows:  
> cyclecloud import_cluster GridEngine-Multiregion -c "Grid Engine" -f gridengine-multiregion-git.txt  

## SLURM  
* Slurm will require a Private DNS zone with all cluster VNETs linked as described [here](https://docs.microsoft.com/en-us/azure/dns/private-dns-getstarted-portal)
* Cluster Template changes:
  * `SubnetId` and `Region` settings are moved from `[[node defaults]]` to each defined `node` and `nodearray`; all values are hardcoded in the template
    * `SubnetId` is of the format `resource-group-name/vnet-name/subnet-name` with placeholder name in the template
    * `Region` can be determined by the azure-cli command `az account list-locations -o table`
  * Added a `cloud-init` in `node defaults` to update Network Manager search-domain to the Private DNS name. Update the default value with your Private DNS name.
  * Removed `parameters Region` and `parameters Networking` from the Parameters section as they are hardcoded in the sections above it
  * Hardcoded Default OS values to avoid issues with UI validations
  * Hardcode changes to VM types, core counts, etc
* Create new cluster named `Slurm-Multiregion` as follows:  
> cyclecloud import_cluster Slurm-Multiregion -c Slurm -f slurm-multiregion-git.txt  


