# Calico study

> To begin we will be installing Multipass. Multipass is a utility from Canonical that allows you to create Ubuntu VMs across a range of platforms in a uniform fashion. We recommend using the latest stable version of Multipass (version 1.5.0 at the time of writing). If any difficulty is encountered deploying the labs - please try this version of Multipass and let us know in the #academy slack channel that you encountered issues with a newer version. 

> Multipass installation instructions can be found here: https://multipass.run

> Note: If you're running on Windows, you must use the default Hyper-V option.  In addition, note that if you are running an old version of VMware Workstation/Player on Windows, you may find that VMware can not start VMs after using multipass due to its use of Hyper-V. If you experience this, please see the workaround instructions later in the "Managing Your Lab" module.

> Note 2: If you have issues with the Multipass VMs, consider disabling VPN clients that might be running on your system. We have seen scenarios where VPN clients can cause problems with VMs started by Multipass.

> Restart your workstation after installing Multipass and before installing the lab. It is essential you do not skip this step.

## Quick start
If you’re on Linux, Mac, or have access to a Bash shell on Windows you can follow these steps to get up and running quickly:

```
curl https://raw.githubusercontent.com/tigera/ccol1/main/control-init.yaml | multipass launch -n control -m 2048M 20.04 --cloud-init -
curl https://raw.githubusercontent.com/tigera/ccol1/main/node1-init.yaml | multipass launch -n node1 20.04 --cloud-init -
curl https://raw.githubusercontent.com/tigera/ccol1/main/node2-init.yaml | multipass launch -n node2 20.04 --cloud-init -
curl https://raw.githubusercontent.com/tigera/ccol1/main/host1-init.yaml | multipass launch -n host1 20.04 --cloud-init -
```

## Starting the Instances
On some platforms, multipass requires you to start the VMs after they have been launched. We can do this by using the multipass start command.
```
multipass start --all
```
Throughout the deployments for the labs, the instances will reboot once provisioning is complete. As a result, you may have to wait a minute until the instance has fully provisioned. A quick way to check the current state of the cluster is to use the multipass list command.
```
multipass list
```
Example output:
<img width="908" alt="Screen Shot 2022-08-30 at 23 25 28" src="https://user-images.githubusercontent.com/66551005/187463110-686b43af-6aa2-4175-9003-579bc4a55f5c.png">



## Validating the Environment
To validate the lab has successfully started after all four instances we will enter the host1 shell:
```
multipass shell host1
```
Once you reach the command prompt of host1, run kubectl get nodes.
```
kubectl get nodes -A
```
Example output:
<img width="908" alt="Screen Shot 2022-08-30 at 23 26 00" src="https://user-images.githubusercontent.com/66551005/187463261-a3700400-813b-493d-90e0-52fafc43f45f.png">
  
Note the “NotReady” status. This is because we have not yet installed a CNI plugin to provide the networking.

The instance we will be using for the following labs will be host1 unless otherwise specified. Think of host1 as your primary entry point into the kubernetes ecosystem, with the other instances acting as the cluster in the cloud.
