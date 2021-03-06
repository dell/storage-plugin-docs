---
title: PowerFlex
linktitle: PowerFlex
description: >
  Installing PowerFlex CSI Driver via Helm
---

The CSI Driver for Dell EMC PowerFlex can be deployed by using the provided Helm v3 charts and installation scripts on both Kubernetes and OpenShift platforms. For more detailed information on the installation scripts, review the script [documentation](https://github.com/dell/csi-powerflex/tree/master/dell-csi-helm-installer).

The controller section of the Helm chart installs the following components in a _Deployment_ in the specified namespace:
- CSI Driver for Dell EMC PowerFlex
- Kubernetes External Provisioner, which provisions the volumes
- Kubernetes External Attacher, which attaches the volumes to the containers
- Kubernetes External Snapshotter, which provides snapshot support
- Kubernetes External Resizer, which resizes the volume

The node section of the Helm chart installs the following component in a _DaemonSet_ in the specified namespace:
- CSI Driver for Dell EMC PowerFlex
- Kubernetes Node Registrar, which handles the driver registration

## Prerequisites

The following are requirements that must be met before installing the CSI Driver for Dell EMC PowerFlex:
- Install Kubernetes or OpenShift (see [supported versions](../../../dell-csi-driver/#features-and-capabilities))
- Install Helm 3
- Enable Zero Padding on PowerFlex
- Mount propagation is enabled on container runtime that is being used
- Install PowerFlex Storage Data Client 
- If using Snapshot feature, satisfy all Volume Snapshot requirements
- A user must exist on the array with a role _>= FrontEndConfigure_


### Install Helm 3.0

Install Helm 3.0 on the master node before you install the CSI Driver for Dell EMC PowerFlex.

**Steps**

  Run the `curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash` command to install Helm 3.0.

### Enable Zero Padding on PowerFlex

Verify that zero padding is enabled on the PowerFlex storage pools that will be used. Use PowerFlex GUI or the PowerFlex CLI to check this setting. For more information to configure this setting, see [Dell EMC PowerFlex documentation](https://cpsdocs.dellemc.com/bundle/PF_CONF_CUST/page/GUID-D32BDFF7-3014-4894-8E1E-2A31A86D343A.html).

### Install PowerFlex Storage Data Client

The CSI Driver for PowerFlex requires you to have installed the PowerFlex Storage Data Client (SDC) on all Kubernetes nodes which run the node portion of the CSI driver. 
SDC could be installed automatically by CSI driver install on Kubernetes nodes with OS platform which support automatic SDC deployment, 
currently Fedora CoreOS (FCOS) and Red Hat CoreOS (RHCOS). 
On Kubernetes nodes with OS version not supported by automatic install, you must perform the Manual SDC Deployment steps [below](#manual-sdc-deployment).
Refer https://hub.docker.com/r/dellemc/sdc for supported OS versions.

#### SDC Deployment

The CSI Driver for PowerFlex requires you to have installed the PowerFlex Storage Data Client (SDC) on all Kubernetes nodes which run the node portion of the CSI driver. SDC could be installed automatically by CSI driver install on Kubernetes nodes with OS platform which support automatic SDC deployment, currently Fedora CoreOS (FCOS) and Red Hat CoreOS (RHCOS). 

On Kubernetes nodes with OS version not supported by automatic install, you must perform the Manual SDC Deployment steps [below](#manual-sdc-deployment). 
Refer https://hub.docker.com/r/dellemc/sdc for your OS versions.

**Optional:** For a typical install, you will pull SDC kernel modules from the Dell EMC FTP site, which is set up by default. Some users might want to mirror this repository to a local location. The PowerFlex KB article (https://www.dell.com/support/kbdoc/en-us/000184206/how-to-use-a-private-repository-for) has instructions on how to do this. 

#### Manual SDC Deployment

For detailed PowerFlex installation procedure, see the _Dell EMC PowerFlex Deployment Guide_. Install the PowerFlex SDC as follows:

**Steps**

1. Download the PowerFlex SDC from [Dell EMC Online support](https://www.dell.com/support). The filename is EMC-ScaleIO-sdc-*.rpm, where * is the SDC name corresponding to the PowerFlex installation version.
2. Export the shell variable _MDM_IP_ in a comma-separated list using `export MDM_IP=xx.xxx.xx.xx,xx.xxx.xx.xx`, where xxx represents the actual IP address in your environment. This list contains the IP addresses of the MDMs.
3. Install the SDC per the _Dell EMC PowerFlex Deployment Guide_:
    - For Red Hat Enterprise Linux and CentOS, run `rpm -iv ./EMC-ScaleIO-sdc-*.x86_64.rpm`, where * is the SDC name corresponding to the PowerFlex installation version.
4. To add more MDM_IP for multi-array support, run `/opt/emc/scaleio/sdc/bin/drv_cfg --add_mdm --ip 10.xx.xx.xx.xx,10.xx.xx.xx`


### (Optional) Volume Snapshot Requirements

Applicable only if you decided to enable snapshot feature in `values.yaml`

```yaml
snapshot:
  enabled: true
```

#### Volume Snapshot CRD's
The Kubernetes Volume Snapshot CRDs can be obtained and installed from the external-snapshotter project on Github.
- If on Kubernetes 1.19 (beta snapshots) use [v3.0.x](https://github.com/kubernetes-csi/external-snapshotter/tree/v3.0.3/client/config/crd)
- If on Kubernetes 1.20/1.21 (v1 snapshots) use [v4.0.x](https://github.com/kubernetes-csi/external-snapshotter/tree/v4.0.0/client/config/crd)

#### Volume Snapshot Controller
The beta Volume Snapshots in Kubernetes version 1.17 and later, the CSI external-snapshotter sidecar is split into two controllers:
- A common snapshot controller
- A CSI external-snapshotter sidecar

The common snapshot controller must be installed only once in the cluster irrespective of the number of CSI drivers installed in the cluster. On OpenShift clusters 4.4 and later, the common snapshot-controller is pre-installed. In the clusters where it is not present, it can be installed using `kubectl` and the manifests are available:
- If on Kubernetes 1.19 (beta snapshots) use [v3.0.x](https://github.com/kubernetes-csi/external-snapshotter/tree/v3.0.3/deploy/kubernetes/snapshot-controller)
- If on Kubernetes 1.20 and 1.21 (v1 snapshots) use [v4.0.x](https://github.com/kubernetes-csi/external-snapshotter/tree/v4.0.0/deploy/kubernetes/snapshot-controller)

*NOTE:*
- The manifests available on GitHub install the snapshotter image: 
   - [quay.io/k8scsi/csi-snapshotter:v3.0.x](https://quay.io/repository/k8scsi/csi-snapshotter?tag=v3.0.3&tab=tags)
   - [quay.io/k8scsi/csi-snapshotter:v4.0.x](https://quay.io/repository/k8scsi/csi-snapshotter?tag=v4.0.0&tab=tags)
- The CSI external-snapshotter sidecar is still installed along with the driver and does not involve any extra configuration.

#### Installation example 

You can install CRDs and default snapshot controller by running following commands:
```bash
git clone https://github.com/kubernetes-csi/external-snapshotter/
cd ./external-snapshotter
git checkout release-<your-version>
kubectl create -f client/config/crd
kubectl create -f deploy/kubernetes/snapshot-controller
```

*NOTE:*
- It is recommended to use 3.0.x version of snapshotter/snapshot-controller when using Kubernetes 1.19
- When using Kubernetes 1.20/1.21 it is recommended to use 4.0.x version of snapshotter/snapshot-controller.
- The CSI external-snapshotter sidecar is still installed along with the driver and does not involve any extra configuration.

## Install the Driver

**Steps**
1. Run `git clone https://github.com/dell/csi-powerflex.git` to clone the git repository.

2. Ensure that you have created a namespace where you want to install the driver. You can run `kubectl create namespace vxflexos` to create a new one.

3. Check `helm/csi-vxflexos/driver-image.yaml` and confirm the driver image points to a new image.

4. Collect information from the PowerFlex SDC by executing the `get_vxflexos_info.sh` script located in the top-level helm directory.  This script shows the _VxFlex OS system ID_ and _MDM IP_ addresses. Make a note of the value for these parameters as they must be entered in the `config.yaml` file in the top-level directory.

5. Prepare the config.yaml for driver configuration. The following table lists driver configuration parameters for multiple storage arrays.

    | Parameter | Description                                                  | Required | Default |
    | --------- | ------------------------------------------------------------ | -------- | ------- |
    | username  | Username for accessing PowerFlex system.                      | true     | -       |
    | password  | Password for accessing PowerFlex system.                      | true     | -       |
    | systemID  | System name/ID of PowerFlex system.                           | true     | -       |
    | endpoint  | REST API gateway HTTPS endpoint for PowerFlex system.         | true     | -       |
    | skipCertificateValidation  | Determines if the driver is going to validate certs while connecting to PowerFlex REST API interface. | true     | true    |
    | isDefault | An array having isDefault=true is for backward compatibility. This parameter should occur once in the list. | false    | false   |
    | mdm       | mdm defines the MDM(s) that SDC should register with on start. This should be a list of MDM IP addresses or hostnames separated by comma. | true     | -       |

    Example: config.yaml

    ```yaml
    - username: "admin"
      password: "password"
      systemID: "ID1"
      endpoint: "https://127.0.0.1"
      skipCertificateValidation: true
      isDefault: true
      mdm: "10.0.0.1,10.0.0.2"
    - username: "admin"
      password: "Password123"
      systemID: "ID2"
      endpoint: "https://127.0.0.2"
      skipCertificateValidation: true
      mdm: "10.0.0.3,10.0.0.4"
    ```

    After editing the file, run the following command to create a secret called `vxflexos-config`
    `kubectl create secret generic vxflexos-config -n vxflexos --from-file=config=config.yaml`

    Use the following command to replace or update the secret:

    `kubectl create secret generic vxflexos-config -n vxflexos --from-file=config=config.yaml -o yaml --dry-run=client | kubectl replace -f -`

    *NOTE:* 

    - The user needs to validate the YAML syntax and array-related key/values while replacing the vxflexos-creds secret.
    - If you update the secret, you will have to reinstall the driver.
    - Old `json` format of the array configuration file is still supported in this release. If you already have your configuration in `json` format, you may continue to maintain it or you may transfer this configuration to `yaml`
    format and replace/update the secret.  
    
6. Create the config map for use by the driver. The config map controls the level and format of the driver's logging. To create it:  
   `kubectl create -f logConfig.yaml`  
   To see possible configuration options, see the 'Dynamic Logging Configuration" section in Features.  

7. If using automated SDC deployment:
   - Check the SDC container image is the correct version for your version of PowerFlex. 
   
8. Copy the default values.yaml file `cd helm && cp csi-vxflexos/values.yaml myvalues.yaml`

9. Edit the newly created values file and provide values for the following parameters `vi myvalues.yaml`:

| Parameter                | Description                                                                                                                                                                                                                                                                                                                                                                                                    | Required | Default |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ------- |
| volumeNamePrefix         | Set so that volumes created by the driver have a default prefix. If one PowerFlex/VxFlex OS system is servicing several different Kubernetes installations or users, these prefixes help you distinguish them.                                                                                                                                                                                                 | No       | "k8s"   |
| controllerCount          | Set to deploy multiple controller instances. If the controller count is greater than the number of available nodes, excess pods remain in a pending state. You can increase the number of available nodes by configuring the "controller" section in your values.yaml. For more details on the new controller pod configurations, see the Features section for Powerflex specifics.                                | Yes      | 2       |
| enablelistvolumesnapshot | Set to have snapshots included in the CSI operation ListVolumes. Disabled by default.                                                                                                                                                                                                                                                                                                                          | No       | FALSE   |
| allowRWOMultiPodAccess   | Setting allowRWOMultiPodAccess to "true" will allow multiple pods on the same node to access the same RWO volume. This behavior conflicts with the CSI specification version 1.3. NodePublishVolume description that requires an error to be returned in this case. However, some other CSI drivers support this behavior and some customers desire this behavior. Customers use this option at their own risk. | No       | FALSE   |
| **controller**           | This section allows the configuration of controller-specific parameters. To maximize the number of available nodes for controller pods, see this section. For more details on the new controller pod configurations, see the [Features section](../../../features/powerflex/) for Powerflex specifics.                                                                                                             | -        | -       |
| nodeSelector             | Defines what nodes would be selected for pods of controller deployment. Leave as blank to use all nodes. Uncomment this section to deploy on master nodes exclusively.                                                                                                                                                                                                                                         | No       | " "     |
| tolerations              | Defines tolerations that would be applied to controller deployment. Leave as blank to install the controller on worker nodes only. If deploying on master nodes is desired, uncomment out this section.                                                                                                                                                                                                            | No       | " "     |
| **monitor**              | This section allows the configuration of the SDC monitoring pod.                                                                                                                                                                                                                                                                                                                                                   | -        | -       |
| enabled                  | Set to enable the usage of the monitoring pod.                                                                                                                                                                                                                                                                                                                                                                 | No       | FALSE   |
| hostNetwork              | Set whether the monitor pod should run on the host network or not.                                                                                                                                                                                                                                                                                                                                             | No       | TRUE    |
| hostPID                  | Set whether the monitor pod should run in the host namespace or not.                                                                                                                                                                                                                                                                                                                                           | No       | TRUE    |
| **podmon**               | Podmon is an optional feature under development and tech preview. Enable this feature only after contact support for additional information.   |  -        |  -       |
| enabled                  |                                                                                                                                                                                                                                                                                                                                                                                                                |  No        |   FALSE      |
| **vgsnapshotter**        | Volume Group Snapshotter(vgsnapshotter) is an optional feature under development and tech preview. Enable this feature only after contact support for additional information.   |  -        |  -       |
| enabled                  |                                                                                                                                                                                                                                                                                                                                                                                                                |  No        |   FALSE      |


11. Install the driver using `csi-install.sh` bash script by running `cd ../dell-csi-helm-installer && ./csi-install.sh --namespace vxflexos --values ../helm/myvalues.yaml`

 *NOTE:*
- For detailed instructions on how to run the install scripts, refer to the README.md  in the dell-csi-helm-installer folder.
- This script runs the `verify-csi-vxflexos.sh` script that is present in the same directory. It will validate MDM IP(s) in `vxflexos-config` secret and creates a new field consumed by the init container and sdc-monitor container
- This script also runs the `verify.sh` script. You will be prompted to enter the credentials for each of the Kubernetes nodes. 
  The `verify.sh` script needs the credentials to check if SDC has been configured on all nodes. 
- It is mandatory to run the first installation and installation after changes to MDM configuration in `vxflexos-config` secret
  **without** skipping the verification. After that, you can use `--skip-verify-node` or `--skip-verify` .
- (Optional) Enable additional Mount Options - A user is able to specify additional mount options as needed for the driver. 
   - Mount options are specified in storageclass yaml under _mkfsFormatOption_. 
   - *WARNING*: Before utilizing mount options, you must first be fully aware of the potential impact and understand your environment's requirements for the specified option.

## Storage Classes

For CSI driver for PowerFlex version 1.4 and later, `dell-csi-helm-installer` does not create any storage classes as part of the driver installation. A wide set of annotated storage class manifests have been provided in the `helm/samples` folder. Use these samples to create new storage classes to provision storage. See this [note](../../../../v2/installation/helm/powerflex/#storage-classes) for the driving reason behind this change.

### What happens to my existing storage classes?

*Upgrading from CSI PowerFlex v1.4 driver*
The storage classes created as part of the installation have an annotation - "helm.sh/resource-policy": keep set. This ensures that even after an uninstall or upgrade, the storage classes are not deleted. You can continue using these storage classes if you wish so.

*Upgrading from an older version of the driver*
The storage classes will be deleted if you upgrade the driver. If you wish to continue using those storage classes, you can patch them and apply the annotation "helm.sh/resource-policy": keep before performing an upgrade.

> *NOTE:* If you continue to use the old storage classes, you may not be able to take advantage of any new storage class parameter supported by the driver.

**Steps to create storage class:**
There are samples storage class yaml files available under `helm/samples/storageclass`.  These can be copied and modified as needed. 

1. Edit `storageclass.yaml` if you need ext4 filesystem and `storageclass-xfs.yaml` if you want xfs filesystem.
2. Replace `<STORAGE_POOL>` with the storage pool you have.
3. Replace `<SYSTEM_ID>` with the system ID you have. Note there are two appearances in the file.
4. Edit `storageclass.kubernetes.io/is-default-class` to true if you want to set it as default, otherwise false. 
5. Save the file and create it by using `kubectl create -f storageclass.yaml` or `kubectl create -f storageclass-xfs.yaml`

 *NOTE*: 
- At least one storage class is required for one array.
- If you uninstall the driver and reinstall it, you can still face errors if any update in the `values.yaml` file leads to an update of the storage class(es):

```
    Error: cannot patch "<sc-name>" with kind StorageClass: StorageClass.storage.k8s.io "<sc-name>" is invalid: parameters: Forbidden: updates to parameters are forbidden
```

In case you want to make such updates, ensure to delete the existing storage classes using the `kubectl delete storageclass` command.  
Deleting a storage class has no impact on a running Pod with mounted PVCs. You cannot provision new PVCs until at least one storage class is newly created.

## Volume Snapshot Class

Starting CSI PowerFlex v1.5, `dell-csi-helm-installer` will not create any Volume Snapshot Class during the driver installation. There is a sample Volume Snapshot Class manifest present in the _helm/samples/_ folder. Please use this sample to create a new Volume Snapshot Class to create Volume Snapshots.
