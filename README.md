# CycleCloud Multi-Region HPC Cluster
This repo contains instructions to create a CycleCloud cluster with Compute VMs in a separate Azure region.  For example, your HPC scheduler/head node could be in East US 2 Region with compute queues/partitions also in East US 2, but also have queues/partitions in East US.  

![CC MultiRegion Overview](https://github.com/themorey/cyclecloud-multiregion-cluster/blob/main/images/ccmr_overview.png "CycleCloud MultiRegion Overview")  

## Why would someone want to span an HPC cluster across Azure Regions?  

- _**Capacity**_.  There are times when a particular Region can not handle the amount of compute cores needed.  In this case the workload can be split across multiple regions to get access to all the compute cores needed.  
- _**Specialty Compute (ie. GPU, FPGA, Infiniband)**_.  An organization created an Azure environment in a specific Region (ie. East US 2) and later requires specialty compute VMs that are not available in that Region.  
- _**[Public Datasets](https://azure.microsoft.com/en-us/services/open-datasets/#overview)**_.  Azure hosts numerous Public Data Sets but they are specific to Regions (ie. West US 2).  You may have an HPC cluster and Data Lake configured in South Central US but need a compute queue/partition in West US 2 to make use of the Public Data Set (ie. US Labor Force Statistics).  

In the examples above, the standard answer is to create a new cluster in the Azure Region.  This collection provides an alternative option to have a single HPC cluster but utilize compute resources in multiple regions.  

# Requirements
  
  ## NETWORKING
  
