# Updating your Mastodon instance

**Note**: This guide assumes you created your Mastodon instance using the instructions from this documentation. If you set up your instance using a different guide, some steps could be different.

## Announce the update (optional)

If you're running an instance that other people use, you should let people know when you are planning to update. That way, they'll know the downtime is intentional, and not something they should be worried about. Ideally, you should announce the downtime several hours in advance, so everyone has time to see the message.

## Make a backup (optional)

It's a good idea to make a backup before updating, in case something goes wrong. General information about backing up Mastodon instances can be found [on the official documentation](https://github.com/tootsuite/documentation/blob/master/Maintaining-Mastodon/Backups-Guide.md).

If you're running your instance on a VPS, your VPS provider may offer backup services. For example, Linode offers automatic and manual backups for a few dollars extra per month (more info [here](https://www.linode.com/backups)).

## Stopping the instance

Before we update, it's important to shut everything down, to prevent possible data loss and/or corruption. Make sure you're logged in as the `mastodon` user, and enter the mastodon directory:

```
cd ~/mastodon
```

Next, shut down the ngix server:

```
sudo systemctl stop nginx.service
```

Then shut down Docker using the `stop` command:

```
docker-compose stop
```

## Updating the local files

Next, we need to download the new version of Mastodon, using Git. First, make sure you're on the `master` branch:

```
git status
```

You should see a message like `On branch master`. Now, we need to update our local files to match the master branch:

```
git pull origin master
```

Now you've finished downloading the latest version of Mastodon! It's time to compile it.

## Create temporary swap file (optional)

If you're using a machine with a small amount of RAM (like a 1GB VPS), you will need to create a temporary swap file (1GB in this example) in order to compile Mastodon.

If you already created a swapfile during the initial setup, you just have to re-enable it again. First, type this command to check if it's already active:

```
cat /proc/swaps
```

If `/swapfile` is not listed, that means you have to re-enable it. Just run this command and you're done:

```
sudo swapon /swapfile
```

If you did not create a swap file in the past, use these commands to make one and enable it:

```
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

## Compiling Mastodon

Now we're ready to compile Mastodon. First, you should check [the latest release documentation](https://github.com/tootsuite/mastodon/releases/), since there might be special steps you need to do. You only need to follow the directions that apply to Docker.

**Note:** Some of the following steps may not be necessary for every release, but it doesn't hurt to run them all. Since this is a general guide for every update, that's what we'll do.

Run each of these commands, one after the other:

```
docker-compose build
docker-compose run --rm web rails assets:precompile
docker-compose run --rm web rails db:migrate
```

If all of the commands worked without errors, then pat yourself on the back, because your instance has been updated!

## Turning everything back on

Now that the update is finished, we need to turn Docker and the ngix server back on. First, restart Docker:

```
docker-compose up -d
```

Then restart ngix:

```
sudo systemctl restart nginx.service
```

After a minute or two, your instance should be back online!
