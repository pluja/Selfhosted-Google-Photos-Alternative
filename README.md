## <p align="center"> Replacing Google Photos </p>
# <p align="center"> Self-Hosted Photo Storage Solution </p>
After testing a lot (almost all of them) of alternatives and solutions. Finally I came up with the definitive solution, the one that gives me everything I want in an easy (once set up is done), cheap, secure and lightweight way.

At the end of this tutorial we will have a Photo Storage solution that will allow us to:

- [x] Sync our photos from any device.  
- [x] Remove exact-photo duplicates efficiently.  
- [x] Automatically organize our Gallery by Year and Month.  
- [x] Manually run Face Recognition and Face Scanning.  
- [x] Manage our gallery from DigiKam.  
- [x] Have an online gallery with a map, faces, tags, ratings and other filters.  
- [x] Upload an encrypted backup of our gallery to any cloud service.  
- [x] Locally generate a backup on a separated drive.  
- [x] Securelly access our Gallery from anywhere outside our network.  

This can be achieved using a Raspberry Pi 4 and a HDD Storage drive. Pretty cheap!

The stack we will be using is the following:

- [PiGallery2](https://github.com/bpatrik/pigallery2) - A simple and lightweight gallery with all that you expect. Read only.  
  - You can also use [PhotoView](https://github.com/photoview/photoview) instead if you want to be able to do the Face Recognition on the Raspberry itself. I personally prefer not to carry this workload on the RPI.
- [Syncthing](https://syncthing.net/) - A simple, fast, peer-to-peer syncing tool.  
- [Phockup](https://github.com/ivandokov/phockup) - Script to organize your Photos.  
- [Czkawka](https://github.com/qarmin/czkawka) - Find duplicates for your photos really fast.
- [SSHFS](https://github.com/libfuse/sshfs) - Manage remote Gallery folder locally.  
- [DigiKam](https://www.digikam.org/) - Face Recognition and Metadata management.  
- rsync - Efficient backup tool.
- rclone - Rsync for cloud services.

## The Guide

1. [Setup the server](https://github.com/pluja/simple-selfhosting/blob/main/01-Setting-Up-The-Server.md#setup-rpi4-server)
2. [Setting up the Google Photos replacement](https://github.com/pluja/simple-selfhosting/blob/main/02-Photo-Storage-Solution.md#replacing-google-photos)
3. [Outside access: PiVPN](https://github.com/pluja/simple-selfhosting/blob/main/03-PiVPN-Convert-Raspi-Into-VPN.md#convert-your-raspberry-into-a-vpn)
