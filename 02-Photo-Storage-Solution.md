# Replacing Google Photos

To replace google photos I will be using a combination of programs and scripts that will end by giving me the very same functionallity I got when I was using Google Photos.

These technologies will be:

- [PiGallery2](https://github.com/bpatrik/pigallery2) - A simple and lightweight gallery with all that you expect. Read only.  
  - You can also use [PhotoView](https://github.com/photoview/photoview) instead if you want to be able to do the Face Recognition on the Raspberry itself. I personally prefer not to carry this workload on the RPI.
- [Syncthing](https://syncthing.net/) - A simple, fast, peer-to-peer syncing tool.  
- [Phockup](https://github.com/ivandokov/phockup) - Script to organize your Photos.  
- [Czkawka](https://github.com/qarmin/czkawka) - Find duplicates for your photos really fast.
- [SSHFS](https://github.com/libfuse/sshfs) - Manage remote Gallery folder locally.  
- [DigiKam](https://www.digikam.org/) - Face Recognition and Metadata management.  
- rsync - Efficient backup tool.
- rclone - Rsync for cloud services.
  

I will be synching my Phone photos using Syncthing to my Raspberry Pi 4 server. There I will run a serires of scripts to help me organize the photos by Year and Month. Then I will make use of DigiKam locally to have the ability to run Face tagging and recogntion along with metadata tweaking.

Then, once done, I will encrypt and backup everything to MEGA using `rclone` so I have an online and encrypted backup of everything. I will also keep a local copy of everything using `rsync` on a separate HDD that will automatically turn on when needed. Let's go.

## Preparation

### Folder Structure

I will have a 3TB HDD connected to my Raspberry Pi where I will be synching all my photos. This HDD is automatically mounted on the `/data/3TB` folder on boot. To do this automation, connect your Storage device on the RPI and run the following command:

`sudo blkid`

You will get a list where you should find your storage device listed:

```shell
/dev/sdb1: LABEL="3TB" UUID="16955984-34c8-435d-b646-6578a3bab016" TYPE="ext4" PARTUUID="ba10c453-2297-5f49-9886-fd76517576cd"
```

You will need to copy the **UUID** label. Then edit the fstab:

- `sudo nano /etc/fstab`

And add the following line usign the UUID and mountpoint for your drive and system:

`UUID=16955984-34c8-435d-b646-6578a3bab016 /data/3Tera auto nosuid,nodev,nofail 0 0`

Save it. Now your External storage will be mounted on boot.

Inside my 3TB storage I have a folder called Photos, just for organization. You can skip this folder if you don't think it is useful. I created it because i may be using the storage for other kinds of media like Music or Films.

Now, inside this folder I have the `tmp` folder for pigallery2 and the `images` folder. Inside images I have two more folders the first one is `Imports` which contains two more folders: `Phone` and `PC`. Here is where all the images will be uploaded from my devices without any kind of Sorting or deduplication. Then I have the `Gallery` folder, here is where all my photos will be kept sorted and perfectly ordered.

The folder structure insidte the 3 TB drive looks like this:

```
Photos/
├── tmp/
└── images/
    ├── Imports/
    │   ├── Phone/
    │   └── PC/
    └── Gallery/
```

## Installing programs

First we need to install the main programs that are needed for everything to work.

### Rust:

Rust is a secure compiled programming language which allows for very efficient applications. We will use it for deleting exact-image duplicates on our Gallery.

```bash
sudo apt install build-essential
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### Syncthing

Syncthing is a very powerful open-source synchronization tool:
> Syncthing is a continuous file synchronization program. It synchronizes files between two or more computers in real time, safely protected from prying eyes. Your data is your data alone and you deserve to choose where it is stored, whether it is shared with some third party, and how it's transmitted over the internet. - From [Syncthing.net](https://syncthing.net)

On your raspberry server run:

- `sudo apt install syncthing`

Now you have installed syncthing. On a web browser on any computer on your network visit `192.168.1.222:8384` (if you changed the static IP for your server previously use the one you chose).

[Install Syncthing on all the devices you want to connect](https://syncthing.net/downloads/)

It is important that you read [this 'Getting Started' page](https://docs.syncthing.net/intro/getting-started.html) if you have never used Syncthing before. Using this guide connect your devices (for example your Phone and your Raspberry server) and understand how all works. I also recommend you to [read this page](https://docs.syncthing.net/intro/gui.html) to get introduced to Syncthing GUI. It is important that you understand how it works as you don't want to mess things up as you could end up losing your files somehow.

In the following section we will configure the folder synchronization between hosts.

#### Syncthing Folders

As we have seen before, I will show you how I configured the above folder structure with syncthing so I can sync my photo sources (Phone and computer). If you have more sources you can just add more folders.

##### Phone Camera

This folder must sync one way. I want to be able to delete the photos from my phone without losing them on my server too. This is why we will set up IgnoreDelete and Send-Only folders.

- *SERVER* Destination Folder: /data/images/Imports/Phone
  
- *PHONE* Source Folder: /DCIM/Camera
  

The *PHONE* folder will be of type **Send-Only** and the *SERVER* folder will be **Recieve-Only.**

It is also very important that we set the **IgnoreDelete** setting to **True** on the server for our Phone folder so we can free up space on the Phone by deleting the photos without them getting delete on the server too. This can be done from the **Syncthing Advanced Folders settings** (See the following image to find out where this option resides):
![](https://i.imgur.com/Bv5eD4B.png)

##### Computer Folder

If we have enough space on our computer, we can save a copy of the **Sorted** folder we will have on our Server so we can move and create new Folders and Albums. We can use **Digikam** from the computer in order to organize all the photos locally and then propagate these changes to the server using Syncthing.

*SERVER*: /data/images/Gallery

*COMPUTER*: /Pictures/Gallery

We will set both folders as Send-Recieve, we may want to have the "IgnoreDelete" form the PC to the Server just in case, but this is optional.

### PiGallery2

This will be the app we will be using to display our Gallery folder. Installing PiGallery2 on our Raspberry is pretty easy, you just need to follow the [app instructions on their README.md](https://github.com/bpatrik/pigallery2/blob/master/docker/README.md). I really recommend using Docker-Compose.

---

Now we have our Syncronization method set up. It was quite easy, we will have our photos from our Phone synced to our server in a matter of minutes and in a peer-to-peer way. How nice and how private!

## The Scripts

Now we will work on different scripts. The IMPORT script, that will handle the sorting, de-duplication and deletion of files and the BACKUP that, as the name already clarifies, will handle the cloud-backup of our photos to any cloud-provider of our choice.

Let's go. I want to have my photos sorted by Year/MM-Month folder structure (eg. 2021/01-January) automatically. This will let me have a very organized gallery from the very beggining. To do this I will be using a very nice script called [Phockup](https://github.com/ivandokov/phockup). This scripts allows us to do exactly what we want to do. To install it we will use the following commands on our server:

```bash
sudo apt-get install python3 libimage-exiftool-perl -y
curl -L https://github.com/ivandokov/phockup/archive/latest.tar.gz -o phockup.tar.gz
tar -zxf phockup.tar.gz
sudo mv phockup-* /opt/phockup
sudo ln -s /opt/phockup/phockup.py /usr/local/bin/phockup
```

Now we have installed Phockup. The command I will be using to Sort my folder will be the following:

`phockup /source/folder /destination/folder --move --date YYYY/MM-M --date-field=FileModifyDate`

This command will take the contents of the source folder and **move** them to the destination folder. If you want to keep a copy of the original uploaded files without being sorted, you can remove the `--move` flag. The `--date YYYY/MM-M` specifies the folder structure we want in this case YEAR/XX-Month. we also have an additional flag called `date-field` that specifies the date field from exiftool that needs to be taken if there is no date specified.

Now we need to install Czkawka in order to achieve the de-duplication of exact same images, videos, etc. To do this we will need to have previously correctly installed Rust and be able to make use of the **cargo** command. Once done, we will:

```bash
git clone https://github.com/qarmin/czkawka.git
cd czkawka
cargo run --release --bin czkawka_cli
```

This will compile Czkawka CLI version so we can use it.

The script for the Import of the files will be the following:

```bash
#!/bin/bash
sourceFolder="/data/3Tera/photos/Imports"
sortedFolder="/data/3Tera/photos/Gallery"
if [ `ls $sourceFolder/Phone $sourceFolder/PC | wc -l` -gt 0 ]
then
        phockup $sourceFolder $sortedFolder --move --date YYYY/MM-M --date-field=FileModifyDate
        ~/czkawka/target/release/czkawka_cli dup -d $sortedFolder -x IMAGE VIDEO -s hashmb -f results-dup.txt --delete-method AEO
fi
```

I am sure you can come up with a better script, if you do don't doubt opening an issue and I will replace it. As I am lazy, and for my setup this is OK I won't be thinking for a better syntax for this. Maybe you could just

This script lets us have imports from both the Computer and the Phone in separate folders to avoid conflicts and strange synchronization issues. You might be able to use a unique folder on the server but I prefer using it this way.

It checks if the Imports directory is empty. If not, then it does the import. If is empty it does nothing.

We need to create a cronjob to run this script every hour (for example) to check if there are new imports, if there are we will sort them and move them to our Gallery and then remove any duplicates. To do this open your crontab with

- `crontab -e`

And paste this (change the IMPORT.sh script path) for running the IMPORT script every 15 minutes.

```
*/15 * * * * /home/pi/IMPORT.sh
```

I run it every 15 minutes as I don't take too much photos along a day (I take ~10 photos a day). If you take a lot of photos you may want to do it more regularly. Or if you are not interested on having your photos in your gallery almost instantly, you can just do it once a day, for example overnight.

### Remote Folder on local machine: Using Digikam for Metadata completion and Face Recognition

You can mount the remote Gallery folder on your local machine so you can work with it. To do it you can make use of `SSH FILESYSTEM` which allows you to mount a remote filesystem to your machine and use it as if it was a regular folder.

This will also allow us to set up DigiKam wiht this mounted folder as a Local Collection and then, for example, scan the collection for faces. Using this method we can keep our Raspberry free of the heavy-workload that face recoginition means and derive this work on our local machine. The only drawback is that we will need to manually run the face recognition from DigiKam.

If you have a **large collection** of photos, I recommend you to get your photos on an SSD on your local machine and scan your collection for faces from there. Once scanned, tagged and written to the metadata upload them to the raspberry pi (or use that same SSD there) as this will make the process much faster.

For me, I started a new gallery from the ground up this year. My old (2TB) photos are kept in an old HDD but I don't need them on my daily gallery for now.

I scanned over 1000 full-quality photos from `sshfs` on the folder mounted on the RPI 3TB HDD in about 25 minutes. I assume that this same process on a local SSD would have taken half the time.

If you want to set the Face Tags and other useful metadata (or edit metada), or just edit the remote Gallery from your desktop computer You need to install `sshfs`:

- `sudo apt install sshfs`

And then mount the remote folder to a local folder like this:

- `sshfs pi@192.168.1.222:/data/3Tera/images/Gallery /home/pluja/Pictures/PiGallery -o idmap=user -o ui d=1000 -o gid=1000`

You can [Download DigiKam AppImage](https://www.digikam.org/download/). DigiKam is an awesome photo management tool. You can do many things there like creating albums or editing metadata, but the most interesting one is the face detection and recognition. Using this you can scan your gallery for faces. You will be able to tag some faces and once done, you can recognize them based on the already tagged faces:

![](https://i.imgur.com/8hqpRlE.jpeg)

Now go to **Settings > Configure DigiKam** and navigate to the **Metadata** section. In the first section on the **Behaviour** tab called "Write this information to the Metadata" select the information you want. If you are tagging faces, select the **Face Tags** so this information is written to the files.

After you have tagged the faces, select a face (for example "Albert Einstein" in the photo) and now you can go to **Album > Write Metadata To Files** to write face Metadata to the files so PiGallery2 can read it.

If you want to unmount the folder, you can use:

- `fusermount -u /home/pluja/Pictures/PiGallery`

## Backing up everything

I want to backup everything and have several copies for all. I want a local copy on a separate HDD which will automatically turn on at night with a timer switch and an additional encrypted copy to be uploaded to a Cloud service like MEGA.

For now, I will just use the local backup and in the near future I will update this file with how to backup to a cloud service. For the cloud backups you can use **rclone** (the tool I will use in the future tutorial), you can investigate about it. In a few words, is just like **rsync** but for cloud services.

### Local Backup

I will be using a very simple yet efficient method for backing everything up. I won't do any compression or packing of files. I will just sync the folders using `rsync`. This command lets you have incremental copies for a folder to a remote host (or locally). As I have a second Raspberry Pi, I will be backing up to a HDD connected to my second server for increased backup security.

The `rsync` command reads a source folder and only writes the differences to the destination folder. The first time it will copy everything, so if you have a big gallery it will take a considerable ammount of time depending on if you are using an HDD or SSD or if you are copying to a remote host. But after that first upload, every backup will only upload the files that have changed or that are new. So it will be much more efficient and fast than copying the whole folder every time.

Here is the BACKUP.sh file:

```bash
#!/bin/bash
sourceFolder = /data/3Tera/pigallery2/images/Gallery
destinationFolder = pi@192.168.1.223:/data/backup/backup/
logPath = /data/3Tera
(echo "------`date`------" && rsync -ai $sourceFolder $destinationFolder) &>> $backupPath/backup.log
```

This will generate a `backup.log` file at the `logPath` destination. It will print the date of the backup and then thanks to the `-i` flag it will only write the newly backed up files. The `-a` flag will copy the Gallery folder structure as is and will keep all files properties among other things. You should definitely use the `-a` flag.

As we said, after the first time, it will only backup the new and the changed images so for this reason you can make the backup more regularly. For example every 3 or 6 hours. Edit you crontab with `crontab -e` to suit your needs. I used the following syntax:

```
20 7-23/4 * * * /home/pi/BACKUP.sh
```

This will make a backup *at minute 30 past every 4th hour from 7 through 23*. I make it start at the 45th minute so it does not conflict with the IMPORT.sh script that takes place every 15 minutes.
