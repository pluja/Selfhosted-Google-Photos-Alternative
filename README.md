# Replacing Google Photos

To replace google photos I will be using a combination of Programs and scripts that will end by giving me the very same functionallity I got when I was using Google Photos.

These technologies will be:

- PiGallery2 - A simple and lightweight gallery with all you expect. Read only.
  
- Syncthing - A simple, fast, peer-to-peer syncing tool.
  
- Phockup - Script to organize your Photos.
  
- Czkawka - Find duplicates for your photos really fast.
  
- MEGA - Cloud storage.
  

I will be syncing my Phone photos using Syncthing to my Raspberry Pi 4 server. There I will run a serires of scripts to help me organize the photos by Year and Month, nothing more.

Then, once done, I will backup everything to MEGA to have an online and encrypted backup of everything. Let's go.

## Installing the scripts

### Install Dependencies

**RUST:**

`sudo apt install build-essential`

`curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`

## Preparation

### Folders Structure

I will have a 3TB HDD connected to my Raspberry Pi where I will be syncing all my photos. This HDD is automatically mounted on the `/data/3TB` folder on boot. To do so, connect your Storage device on the RPI and run the following command:

`sudo blkid`

You should get a list where you should find your storage device listed:

```shell
/dev/sdb1: LABEL="3TB" UUID="16955984-34c8-435d-b646-6578a3bab016" TYPE="ext4" PARTUUID="ba10c453-2297-5f49-9886-fd76517576cd"
```

You will need to copy the UUID label. Then edit the fstab:

`sudo nano /etc/fstab`

`UUID=16955984-34c8-435d-b646-6578a3bab016 /data/3Tera auto nosuid,nodev,nofail 0 0`

Use this line with the UUID you just copied and the mountpoint you want for your device and save it. Now your External storage will be mounted on boot.

Inside my 3TB storage I have a folder called Photos, just for organization. You can skip this folder if you don't think it is useful. I created it because i may be using the storage for other kinds of media like Music or Films.

Now, inside this folder I have the `tmp` folder for pigallery2 and the `images` folder. Inside images I have two more folders the first one is `Imports` which contains two more folders: `Phone` and `PC`. Here is where all the images will be uploaded from my devices without any kind of Sorting or deduplication. Then I have the `Gallery` folder, here is where all my photos will be kept sorted and perfectly ordered.

### Syncthing Folders

#### Phone Camera

*SERVER*: /data/images/Imports/Phone

*PHONE*: /camera

The *PHONE* folder will be Send-Only and the *SERVER* folder will be Recieve-Only.

We will need to set the IgnoreDelete setting on Syncthing from our Phone device so we can free up space on the Phone. This can be done from the Syncthing Advanced Folders settings (not folder advanced settings!).

#### Computer Folder

If we have enough space on our computer, we can save a copy of the **Sorted** folder we will have on our Server so we can move and create new Folders and Albums. We can use **Digikam** from the computer in order to organize all the photos locally and then propagate these changes to the server using Syncthing.

*SERVER*: /data/images/Gallery

*COMPUTER*: /Pictures/Gallery

We will set both folders as Send-Recieve, we may want to have the "IgnoreDelete" form the PC to the Server just in case, but this is optional.

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
if [ `ls $sourceFolder/Phone | wc -l` -gt 0 ]
then
        phockup $sourceFolder/Phone $sortedFolder --move --date YYYY/MM-M --date-field=FileModifyDate
        ~/czkawka/target/release/czkawka_cli dup -d $sortedFolder -x IMAGE VIDEO -s hashmb -f results-dup.txt --delete-method AEO
fi

if [ `ls $sourceFolder/PC | wc -l` -gt 0 ]
then
        phockup $sourceFolder/PC $sortedFolder --move --date YYYY/MM-M --date-field=FileModifyDate
        ~/czkawka/target/release/czkawka_cli dup -d $sortedFolder -x IMAGE VIDEO -s hashmb -f results-dup.txt --delete-method AEO
fi
```

You can come up with a better script, if you do don't doubt opening an issue and I will replace it. As I am lazy, and for my setup this is OK I won't be thinking a better syntax for this.

This script lets us have imports from both the Computer and the Phone in separate folders to avoid conflicts and strange synchronization issues. You might be able to use a unique folder on the server but I prefer using it this way.

We need to create a cronjob to run this script every hour (for example) to check if there are new imports, if there are we will sort them and move them to our Gallery and then remove any duplicates.
