# Transferring custom image from GCE to Azure

When talking to customers about trying out Azure, one sticking point that we hear from customers often is that rebuilding
their workflow (again) in a different cloud is painful. One of the pain points is regarding their VMs - it takes alot of time
and energy to build a new image so even getting started with their apps, scripts, etc is just... hard.... 

There doesn't seem to be an end-to-end guide on how to move a custom image from GCE into Azure if I were a Azure newbie.
In an effort to make the transition from Google GCE to Microsoft Azure easier, I thought it made some sense to test how one 
might do that. 

It's pretty straight-forward and only takes about an hour once you know what to do. Here's how I did it:

# Prepare image

This page gives you just about all you need to prepare the image for Azure. Follow these steps 
https://docs.microsoft.com/en-us/azure/virtual-machines/linux/create-upload-ubuntu

A GCE image, of course, also comes loaded with a bunch of GCE-related goodies that should proably be removed. 

SSH-Guard caused me some trouble so I removed that:
```
sudo apt-get remove --purge sshguard
```

Removing these files helped to clean up and quite-down the logs:
```cd /
rm etc/apt/apt.conf.d/01autoremove-gce 
rm etc/apt/apt.conf.d/99-gce-ipv4-only 
rm etc/cloud/cloud.cfg.d/91-gce.cfg
rm etc/cloud/cloud.cfg.d/91-gce-system.cfg
rm etc/apt/apt.conf.d/99ipv4-only 
rm etc/modprobe.d/gce-blacklist.conf
rm etc/rsyslog.d/90-google.conf
rm etc/sysctl.d/11-gce-network-security.conf 
rm etc/sysctl.d/99-gce.conf 
rm lib/systemd/system-preset/90-google-compute-engine.preset
rm lib/systemd/system/google-accounts-daemon.service
rm lib/systemd/system/google-clock-skew-daemon.service
rm lib/systemd/system/google-network-daemon.service
rm lib/systemd/system/google-instance-setup.service
rm lib/systemd/system/google-shutdown-scripts.service
rm lib/systemd/system/google-startup-scripts.service
rm lib/udev/rules.d/64-gce-disk-removal.rules
rm lib/udev/rules.d/65-gce-disk-naming.rules
rm lib/udev/rules.d/99-gce.rules
rm usr/bin/google_accounts_daemon
rm usr/bin/google_clock_skew_daemon
rm usr/bin/google_instance_setup
rm usr/bin/google_metadata_script_runner
rm usr/bin/google_network_daemon
rm usr/bin/google_optimize_local_ssd
rm usr/bin/google_set_multiqueue
rm usr/share/doc/gce-compute-image-packages/TODO.Debian
rm usr/share/doc/gce-compute-image-packages/changelog.Debian.gz
rm usr/share/doc/gce-compute-image-packages/copyright
rm usr/share/lintian/overrides/gce-compute-image-packages
```


You will want to disable cloudinit and let waagent do the provisioning from the waagent config file:
```
vi /etc/waagent
```
 I disabled the firewall to simplify the installation:
```
# Enable instance creation
Provisioning.Enabled=y
...

# Rely on cloud-init to provision
Provisioning.UseCloudInit=n
...

# Add firewall rules to protect access to Azure host node services
OS.EnableFirewall=n
```

Stop the VM

#Capture the image

Login to the GCE console:
```
$ gcloud auth login
```

List disks in project:
```
$ gcloud compute disks list
NAME                LOCATION    LOCATION_SCOPE  SIZE_GB  TYPE         STATUS
testimage-ubuntu-2  us-east4-c  zone            10       pd-standard  READY
ubuntu-demo         us-east4-c  zone            10       pd-standard  READY
```

Create image in GCE
https://cloud.google.com/sdk/gcloud/reference/beta/compute/images/create
```
$ gcloud compute images create ubuntu-demo-image --source-disk=ubuntu-demo --source-disk-zone=us-east4-c
```

Make the object publicly accessible:
https://cloud.google.com/storage/docs/access-control/making-data-public
```
$ gsutil iam ch allUsers:objectViewer gs://mybucket-public/
```

Export image in GCE
https://cloud.google.com/sdk/gcloud/reference/beta/compute/images/export
```
$ gcloud compute images export --destination-uri=gs://mybucket-public/test-image_fromgce.vhdx --image=test-image --async --export-format=vhdx
```
