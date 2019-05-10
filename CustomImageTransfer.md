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
