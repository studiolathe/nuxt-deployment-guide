## Nuxt.js Deployment

This guide assumes you have already setup an Ubuntu server running LAMP stack.

## Step 01 - Setting up your deployment scripts

Create `scripts/depoy.sh` and update key values:


```
#!/usr/bin/env sh

USER=root
HOST=123.123.123.123
WEBROOT=/var/www/your-site-repo

ssh $USER@$HOST /bin/bash <<EOF
  cd $WEBROOT
  git fetch
  git reset --hard origin/master
  yarn install
  yarn run build
  pm2 restart all
EOF
```

Then make sure to update the `package.json` and add scripts:

```
"scripts": {
  "build": "nuxt build",
  "start": "nuxt start"
}
```

Then enable the script by running the below locally:

```
$ chmod +x scripts/*.sh
```

## Step 02 - Configuring the server

Nuxt needs Node.js to run, so make sure Node and Yarn are installed. Follow the steps below to install Yarn on your Ubuntu 18.04 system:

The first step is to enable the Yarn repository. Start by importing the repository’s GPG key using the following curl command:

```
$ curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
```

Add the Yarn APT repository to your system’s software repository list by typing:

```
$ echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
```

Once the repository is added to the system, update the package list and install Yarn, with:

```
$ sudo apt update
$ sudo apt install yarn
```
If you already don’t have Node.js installed on your system, the command above will install it.

To verify that Yarn installed successfully, run the following commands which will print the Yarn version number:

```
$ yarn --version
```

We then need to install pm2 which will helps manage the server later down the track.

## Step 03 - Cloning your local repo to your server

Install git on the server and clone your repo into a directory. First off, we need to make sure you have a token for authenticating Github on the server. If not follow here:  https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line

Once you have your token, ssh into the server and cd into the web root and clone your repo onto the server:

```
$ cd /var/www\
$ git clone https://YOUR_TOKEN@github.com/username/repo.git
```

## Step 04 – Building the Nuxt App on the server 

So now we have our repo cloned onto the server we want to build our app. To do this run:

```
$ yarn install
$ yarn run build
$ yarn run start
```

Once that's completed try visiting http://SERVER_IP:3000. NOTE: if its not loading, stop the Next server and restart it with:

```
HOST=0.0.0.0 yarn run start
```

Then try visiting http://SERVER_IP:3000 again and you should see the site live.

## Step 05 - Keeping the Nuxt server running
Basically the idea is that you have the Nuxt server running in the background (we can use pm2 for this)

Your server will sit there on port 3000 or whatever, and you'll set up apache to "reverse proxy" from your domain name to the nuxt server.

So lets install pm2 by running from within the repo folder on the server:

```
$ cd /var/www/repo
$ yarn global add pm2
```

Now instead of running `HOST=0.0.0.0 yarn run start` we can run the below to have our Nuxt server running in the background:

```
pm2 start "yarn start"
```


## Step 06 - Pointing our Nuxt server to our domain

In the example below we will be pointing our Nuxt server to a sub-domain. First lets create a config file for the sub-domain.

```
$ nano /etc/apache2/sites-available/staging.ccbabcoq.co.conf
```

And add the below:

```
<VirtualHost *:80>
    ServerAdmin admin@example.com
    ServerName staging.example.com
    DocumentRoot /var/www/repo/dist

    ProxyPass / http://localhost:3000/
    ProxyPassReverse / http://localhost:3000 /

    <Directory /var/www/repo/dist>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        allow from all
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

We then need to enable our config:

```
$ a2ensite staging.example.com.conf
```

Now we need to enable our proxy and restart apache:

```
$ sudo a2enmod proxy
$ sudo a2enmod proxy_http
$ sudo service apache2 restart
```

Now head to http://staging.example.com and your site should be live!
