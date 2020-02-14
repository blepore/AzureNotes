# Introduction

Using Azure for the basic building blocks, let’s demonstrate a Reservoir Simulation pipeline.

Open Porous Media (OPM) is being used for our tool and dataset. 
OPM is an open source project for the development of applications aimed at the modeling and simulation of 
porous media processes. Two popular applications to come out of this project are Flow and ResInsight.

Read more in the “Reservoir Simulation on Azure” 
(https://techcommunity.microsoft.com/t5/azure-compute/reservoir-simulation-on-azure-hpc-for-oil-amp-gas/ba-p/791986) 
article published in MS Tech Community.

# CycleCloud Login
```
URL: https://cycleserverd42c4f.southcentralus.cloudapp.azure.com
Username: demo
Password: HPCOptimal20
```

# Create HPC Cluster Template
```
Region: South Central US
Master VM Type: Standard_D12v2
Execute Type: Standard_H16 (uncheck F2s_v2)
Subnet ID: hpcvnet-compute

BaseOS: OpenLogic:CentOS-HPC:7.6:latest
Custom image: check
Master/Execute Cluster-Init: Browse button -> Demo -> 1.0.0 -> default
Public Head Node: Check ‘Access master node from the Internet’
```
# Create Cluster
Clusters -> choose your newly-created cluster -> click ‘Start’

# SSH to master & kick off job
Click ‘master’. In Details pane, choose ‘Connect’ to see ssh/IP address of master node. Copy/Paste to terminal to log in.
```
User: demo
Password: HPCOptim@l20

Run: qsub -l select=2:ncpus=16:mpiprocs=16,place=scatter:excl $HOME/apps/opm/flow_norne.sh
Run 'qstat –f' to check status
```

# Visualize output
RDP to NV Windows VM:
```
IP address: 
Username: hpcadmin
Password: Tiered20202020
```
Launch ResInsight and view output:
Launch ResInsight from the Y: drive 
```
  Y:\resinsight\ResInsight-2019.04.0_oct-4.0.0_souring_win64\ResInsight
  ```
From ResInsight, click import eclipse case. The result file is located on the Z: drive
 ``` Z:\opm-data\norne\out_parallel\NORNE_ATW2013.EGRID
```
