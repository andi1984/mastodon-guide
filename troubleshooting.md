# Troubleshooting

This page lists possible issues you may encounter while setting up or managing a Mastodon instance, and how to fix them.

## Webpacker compliation error

If you're getting an error during the compilation process (like `The code generator has deoptimised the styling of "/mastodon/node_modules/emoji-mart/dist-es/data/data.js" as it exceeds the max of "500KB"`), you need more RAM. This can be solved by following the steps in the "Create temporary swap file" section.

## Running out of space

If your server is running out of space, it's possible the cron job designed to clean up temporary files is not running. Run this command to see the entire cron file for `root`:

`sudo crontab -l`

This line should be somewhere in the file (if not, add it):

`0 0 * * * /home/mastodon/mastodon_cron > /home/mastodon/mastodon_log`

To clear up files manually, you can just run `/home/mastodon/mastodon_cron` at any time.

## Error uploading profile picture/background/other images

If you see an error message when you try to upload a profile picture, profile background, or other images, you may need to update permissions for the public/system folder. Make sure you are logged into the `mastodon` user, and run this command:

`sudo chown -R 991:991 /home/mastodon/mastodon/public/system`

## The site loads but looks funky / doesn't respond properly

If you are seeing server errors or assets not rendering on the frontend, something might just need a gentle push to rebuild and start working. 

A good first step is to run this block of commands **one line at a time** from your `/home/mastodon/mastodon` directory, making sure none of them fail - then check https://your.domain again to see if things are working.

```
docker-compose down
docker-compose build
docker-compose run --rm web rails assets:precompile
docker-compose run --rm web rails db:migrate
docker-compose up -d
sudo systemctl restart nginx.service
```

**Note:** `docker-compose down` will remove all containers, meaning it will clear out the database. If you'd like to avoid potential data loss, substitute it with `docker-compose stop`. This is less of an issue for a fresh install, but you'll want to switch to the latter once you have registered users.

## Email confirmation isn't working

### Is your host blacklisted?

Some mail hosts will blacklist or bounce emails coming from newly created email addresses or domains. Thankfully Mailgun will try sending these messages again and it should eventually get through.

If you are not receiving emails and need to confirm a user account, go to the "Logs" tab inside the Mailgun UI and click your domain. You should see a green row with the verification email that it attempted to send. Click the cog icon to the left of the row and then "View Message", from here you can manually copy/paste the confirmation URL into your browser to get around this problem.

After your instance is live for ~24 hours, you shouldn't be having this issue any longer. 

### Is your SMTP sending port blocked?

Some ISPs and hosts such as Scaleway block email sending on the standard SMTP port, `25`. To see if that's happening view the web container's logs using `docker logs mastodon_web_1` and look for connection timeouts. To correct this edit the `.env.production` configuration file and change `SMTP_PORT: 25` to port `2525`, which Mailgun and SparkPost support.

After doing so you'll need to rebuild the containers:
```
docker-compose stop
docker-compose build
docker-compose up -d
```
