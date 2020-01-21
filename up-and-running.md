# Running a Mastodon instance on Ubuntu 16.04 with Docker & nginx

This guide assumes you have:

1. Root access to a fresh Ubuntu 16.04 x64 machine.
2. A domain name (preferably a wacky TLD) pointed towards the IP address of the machine
3. Some patience... please work all the way through to the "Troubleshooting" section before calling it a day 😃

These instructions have been tested and verified with the standard Ubuntu 16.04 x64 image on a $10/mo [Vultr](https://vultr.com) instance, a $20/mo [DigitalOcean](https://digitalocean.com) instance, and a $5/mo [Linode](https://linode.com) instance. 2GB memory is recommended, but 1GB will work for small instances. You'll also find things faster if you can select the region nearest you geographically. 

## Step 1. Create a `mastodon` user with root privileges

First login via SSH as the root user, then add a new user:

`adduser mastodon`

Then give the user root privileges:

`usermod -aG sudo mastodon`

Switch to the new user:

`su - mastodon`

## Install Docker

First, update the package database:

`sudo apt-get update`

Then install APT repository management tools and support for `https` repositories:

`sudo apt-get install apt-transport-https software-properties-common`

Now let's install Docker. Add the GPG key for the official Docker repository to the system:

`sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D`

Add the Docker repository to APT sources:

`sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"'`

Update the package database with the Docker packages from the newly added repo:

`sudo apt-get update`

Tell the system to install from the Docker repo instead of the default Ubuntu 16.04 repo:

`sudo apt-cache policy docker-engine`

Finally, install Docker:

`sudo apt-get install docker-ce`

Docker should now be installed, the daemon started, and the process enabled to start on boot. Check that it's running:

`sudo systemctl status docker`

Look for the text "active (running)" under the docker.service

So you don't have to prepend `sudo` to every `docker` command in the future, add yourself to the `docker` group that was just created:

`sudo usermod -aG docker $(whoami)`

Exit the SSH session and then log back in before moving to the next step.

## Install Docker Compose

Check the latest release number [here](https://github.com/docker/compose/releases) and run the following command, swapping out the release number ("1.18.0" below): 

`sudo curl -o /usr/local/bin/docker-compose -L "https://github.com/docker/compose/releases/download/1.18.0/docker-compose-$(uname -s)-$(uname -m)"`

Set the necessary permissions:

`sudo chmod +x /usr/local/bin/docker-compose`

Verify it installed by checking the version:

`docker-compose -v`

If you see the version number, we're in business.

## Create temporary swap file (optional)

If you're using a machine with a small amount of RAM (like a 1GB VPS), you will probably need to create a temporary swap file in order to complete the  rest of the guide. These commands will create a 1GB swap file:

```
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

## Install Mastodon

We're now reading to clone the git repo and start setting up Mastodon. Make sure you're in your home directory by running:

`cd /home/mastodon`

Clone the repo into this directory:

`git clone https://github.com/tootsuite/mastodon.git`

Next up, navigate to the cloned app directory:

`cd mastodon`

Copy the production environment configuration file:

`cp .env.production.sample .env.production`

We need to generate some secret keys before we configure everything, so we'll build the Docker image and get it running unconfigured first:

`docker-compose build` ***<== this may take a while, go make yourself a cup of tea*** ☕️

Now that the image is built, we're going to generate 3 secret keys that are required for the configuration. Run this command 3 times and note down each of the keys generated, we'll need them in a minute:

`docker-compose run --rm web rake secret`

Once you have your 3 secret keys noted somewhere, it's time to edit the configuration file:

`nano .env.production`

Edit the settings, and use the 3 secret keys you generated to populate the `PAPERCLIP_SECRET`, `SECRET_KEY_BASE` and `OTP_SECRET` settings. The order in which you copy/paste the keys doesn't matter, so long as each value is different. 

Next, we need to update the following values:

- **(Required)** Change the `LOCAL_DOMAIN` setting to match the domain you have pointing at the machine.
- **(Required)** Email settings:
	- [Setup a Mailgun account](https://app.mailgun.com/new/signup/) (make sure to enter your CC details for the 10k/mo free email tier)
	- Go through the domain verification process and make sure the domain is listed as green and "active"
	- From the [domains page](https://app.mailgun.com/app/domains) click your domain and then copy/paste the "Default SMTP Login" into your config file as the `SMTP_LOGIN` value and the "Default Password" as the `SMTP_PASSWORD`
	- Change the `SMTP_FROM_ADDRESS` value to something like notifications@<your domain>
- **Optional** If you want to store images and media on S3 instead of your machine (ie. if you don't have a lot of disk space) then generate the required settings via AWS and populate the S3 details (recommended for production use)

Once done, press ^X, type `Y` and hit Enter to save your changes. You can also use any text editor you'd like, if you're not a fan of `nano`

As we've changed the config, we'll need to build again:

`docker-compose build`

Now it's time to run migrations: 

`docker-compose run --rm web rails db:migrate`

Give the appropriate permissions:

`docker-compose run --rm -u root web chown -R mastodon:mastodon public/assets public/packs public/system`

We can pre-compile assets too to make things snappier:

`docker-compose run --rm web rails assets:precompile`

Once completed, run the container:

`docker-compose up -d`

## Setup nginx in front of Docker

First, let's install nginx:

`sudo apt-get install nginx`

We'll remove the default site profile:

`sudo rm /etc/nginx/sites-available/default`

...and its symbolic link:

`sudo rm /etc/nginx/sites-enabled/default`

Then create a new profile for our Mastodon instance:

`sudo touch /etc/nginx/sites-available/mastodon`

...and a symbolic link enabling it:

`sudo ln -s /etc/nginx/sites-available/mastodon /etc/nginx/sites-enabled/mastodon`

Now let's configure nginx:

`sudo nano /etc/nginx/sites-available/mastodon`

Copy the nginx config from [here](https://github.com/tootsuite/documentation/blob/master/Running-Mastodon/Production-guide.md#mastodon-application-configuration) and paste it into the `nano` editor window. 

Find all intances of `example.com` and replace with the domain name you have pointed to your machine and replace "live" by "mastodon" as we cloned Mastodon to a "mastodon" folder within "mastodon" user. These are the only changes you should make.

Once done, press ^X, type `Y` and hit Enter to save your changes.

## Setup SSL/HTTPS with Certbot

On Ubuntu systems, the Certbot team maintains a PPA. Once you add it to your list of repositories all you'll need to do is apt-get the following packages:

```
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
```

Now we can install Certbot:

`sudo apt-get install python-certbot-nginx`

Once installed, we can generate the SSL certificates. First you'll need to stop your nginx server so that Certbot can run its standalone authenticator:

`sudo systemctl stop nginx.service`

Then simply run the following command replacing `example.com` with the domain you have pointed at your machine:

`sudo certbot --nginx -d example.com`

Follow the prompts to complete the process. 

## Run Mastodon

We've made a lot of progress, let's get everything running. First, make sure you're in the Mastodon install directory:

`cd /home/mastodon/mastodon`

Now let's stop Docker and rebuild everything to be safe. Run these commands **one line at a time**, making sure none fail:

```
docker-compose down
docker-compose build
docker-compose run --rm web rails assets:precompile
docker-compose run --rm web rails db:migrate
docker-compose up -d
```

Lastly, let's bring nginx back online:

`sudo systemctl restart nginx.service`

You should be able to visit your domain now, be auto-directed to the HTTPS protocol and see your running Mastodon instance!

**NOTE**: If you created a temporary swap file, you can now delete it by running `sudo swapoff /swapfile` and `sudo rm -rf /swapfile`.

## Setup cron jobs

### Automate Mastodon tasks

To keep Mastodon running smoothly, we need to set up some cron jobs to regularly tidy things up in the background. First, make sure you're in the home directory of your `mastodon` user:

`cd /home/mastodon`

Next up, we'll create a file with the commands we'd like to schedule:

`nano mastodon_cron`

Copy/paste in everything below:

```
cd /home/mastodon/mastodon
docker-compose run --rm web rake mastodon:media:clear
docker-compose run --rm web rake mastodon:media:remove_remote
docker-compose run --rm web rake mastodon:push:refresh
docker-compose run --rm web rake mastodon:push:clear
docker-compose run --rm web rake mastodon:feeds:clear
```

Once done, press ^X, type `Y` and hit Enter to save your changes. Now we need to tell cron to run this periodically:

`sudo chmod +x mastodon_cron && sudo crontab -e`

If you're prompted to select a text editor, just hit `2` then Enter to select `nano`. The crontab file should open up in edit mode, copy/paste and add this to the end of the file:

`0 0 * * * /home/mastodon/mastodon_cron > /home/mastodon/mastodon_log`

Once again, press ^X, type `Y` and hit Enter to save your changes. This has just told cron to run the specified commands at 12 midnight each day. You can adjust the `/home/mastodon/mastodon_cron` file to change what gets run daily, or change the scheduling by editing the crontab file and saving it again.

### Auto-renew Let's Encrypt SSL certificate

Let's Encrypt SSL certificates expire every 90 days, so it's important that we schedule a job to automatically renew this periodically in the background - otherwise our instance will eventually lose SSL.

Let's fire up crontab again:
 
 `sudo crontab -e`
 
Press the down arrow until you reach the end of the file, you should see your mastodon_cron job that we setup previously. Below that, copy/paste the following:

```
0 1 * * 1 /usr/bin/letsencrypt renew >> /home/mastodon/letsencrypt.log
5 1 * * 1 /bin/systemctl reload nginx
```

Once done, press ^X, type Y and hit Enter to save your changes. 

We just created a new cron job that will execute the letsencrypt-auto renew command every Monday at 1:00am, and reload Nginx at 1:05am (so the renewed certificate will be used). If the certificate is less than 60 days old, letsencrypt will not try to renew it.

The output produced by the command will be piped to a log file located at /home/mastodon/letsencrypt.log

## Next Steps: Managing your instance

Information on managing your instance can be found [here](https://github.com/tootsuite/documentation/blob/master/Running-Mastodon/Administration-guide.md) on the Mastodon repository itself. Before you start editing site settings etc. however, you need to make yourself an admin. 

### Granting admin permissions

After registering on your newly created instance and confirming your account, you'll need to navigate to your Mastodon directory:

`cd /home/mastodon/mastodon`

Then run this command:

`docker-compose run --rm web bin/tootctl accounts modify <yourusername> --role admin`

Logout and log back in again, then visit your.domain/admin/settings to start customizing your instance.

## Troubleshooting

If you have any problems during (or after setup), please see the [Troubleshooting page](https://github.com/ummjackson/mastodon-guide/blob/master/troubleshooting.md).
