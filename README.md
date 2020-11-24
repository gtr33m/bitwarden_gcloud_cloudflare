# Bitwarden self-hosted on Google Cloud for Free

Bitwarden installation optimized for Google Cloud's 'always free' f1-micro compute instance

> _Note: if you follow these instructions the end product is a self-hosted instance of Bitwarden running in the cloud and will be free **unless** you exceed the 1GB egress per month or have egress to China or Australia. I talk about best practices to help avoid China/AUS egress, but there's a chance you can get charges from that so please keep that in mind._

This is a quick-start guide. Read about this project in more detail [here](https://bradford.la/2020/self-host-bitwarden-on-google-cloud).

---

## Features

* Bitwarden self-hosted
* Automatic https certificate management through cloudflare proxy
* Blocking brute-force attempts with fail2ban(not finished yet)
* Automatic backup your files to google drive periodicity(not finished yet).
## Pre-requisites

Before you start, ensure you have the following:

1. A Google Cloud account
2. A Cloudflare-managed DNS site with an A record ready for Bitwarden

## Step 1: Set up Google Cloud `f1-micro` Compute Engine Instance

Google Cloud offers an '[always free](https://cloud.google.com/free/)' tier of their Compute Engine with one virtual core and ~600 MB of RAM (about 150 MB free depending on which OS you installed). [Bitwarden RS](https://github.com/dani-garcia/bitwarden_rs) runs well under these constraints; it's written in Rust and an ideal candidate for a micro instance. 

Go to [Google Compute Engine](https://cloud.google.com/compute) and open a Cloud Shell. You may also create the instance manually following [the constraints of the free tier](https://cloud.google.com/free/docs/gcp-free-tier). In the Cloud Shell enter the following command to build the properly spec'd machine: 

```bash
# create firewall rules that only accept connections from cloudflare
$ gcloud compute firewall-rules create cloudflare-webs \
  --allow=tcp:80,tcp:8080,tcp:8880,tcp:2052,tcp:2082,tcp:2086,tcp:2095,tcp:443,tcp:2053,tcp:2083,tcp:2087,tcp:2096,tcp:8443,udp:80,udp:8080,udp:8880,udp:2052,udp:2082,udp:2086,udp:2095,udp:443,udp:2053,udp:2083,udp:2087,udp:2096,udp:8443 \
  --source-ranges $(curl https://www.cloudflare.com/ips-v4 | sed -z 's/\n/,/g') \
  --target-tags=cloudflare-webs 
  
# update firewall-rules with latest cloudflare ip
$ gcloud compute firewall-rules update cloudflare-webs --source-ranges $(curl https://www.cloudflare.com/ips-v4 | sed -z 's/\n/,/g')

# create vm
$ gcloud compute instances create bitwarden-rs \
    --machine-type f1-micro \
    --zone us-west1-b \
    --image-project cos-cloud \
    --image-family cos-stable \
    --boot-disk-size=30GB \
    --tags cloudflare-webs \
    --scopes compute-rw
```

You may change the zone to be closer to you or customize the name (`bitwarden`), but most of the other values should remain the same. 

## Step 2: Pull and Configure Project

Enter a shell on the new instance and clone this repo:

```bash
$ git clone https://github.com/dadatuputi/bitwarden_gcloud.git
$ cd bitwarden_gcloud
```

Set up the docker-compose alias by using the included script:

```bash
$ sh utilities/install-alias.sh
$ source ~/.bashrc
$ docker-compose --version
docker-compose version 1.25.5, build 8a1c60f
```

### Configure Environmental Variables with `.env`

I provide `.env.template` which should be copied to `.env` and filled out; filling it out is self-explanitory and requires certain values such as a domain name, Cloudflare API tokens, etc. 

### Configure `fail2ban` (_Not finished yet_)

`fail2ban` need to be integrated with cloudflare API.

https://guides.wp-bullet.com/integrate-fail2ban-cloudflare-api-v4-guide/


### Configure Automatic Rebooting After Updates (_optional_)

Container-Optimized OS will automatically update itself, but the update will only be applied after a reboot. In order to ensure that you are using the most current operating system software, you can set a boot script that waits until an update has been applied to schedule a reboot.

Before you start, ensure you have `compute-rw` scope for your bitwarden compute vm. If you used the `gcloud` command above, it includes that scope. If not, go to your Google Cloud console and edit the "Cloud API access scopes" to have "Compute Engine" show "Read Write". You need to shut down your compute vm in order to change this.

Modify the script to set your local timezone and the time to schedule reboots: set the `TZ=` and `TIME=` variables in `utilities/reboot-on-update.sh`. By default the script will schedule reboots for 06:00 UTC. 

From within your compute vm console, type the command `toolbox`. From within `toolbox`, find the `utilities` folder within `bitwarden_gcloud`. `toolbox` mounts the host filesystem under `/media/root`, so go there to find the folder. It will likely be in `/media/root/home/<google account name>/bitwarden_gcloud/utilities` - `cd` to that folder.

Next, use `gcloud` to add the `reboot-on-update.sh` script to your vm's boot script metadata with the `add-metadata` [command](https://cloud.google.com/compute/docs/startupscript#startupscriptrunninginstances):

```bash
gcloud compute instances add-metadata bitwarden-rs --zone=us-west1-b --metadata-from-file startup-script=reboot-on-update.sh
```

You can confirm that your startup script has been added in your instance details under "Custom metadata" on the Compute Engine Console. 

Next, restart your vm with the command `$ sudo reboot`. Once your vm has rebooted, you can confirm that the startup script was run with the command:

```bash
$ sudo journalctl -u google-startup-scripts.service
```

Now the script will wait until a reboot is pending and then schedule a reboot for the time configured in the script.

## Step 3: Start Services

To start up, use `docker-compose`:

```bash
$ docker-compose up
```

You can now use your browser to visit your new Bitwarden site. 
