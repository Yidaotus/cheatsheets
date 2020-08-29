# A step by step cheat sheet on how I usually deploy my fullstack application to a linux server for the initial alpha test.

# Update the package list

```
sudo apt update
```

# Install the mysql-server

```
sudo apt install mysql-server
```

*   Leave the root pw empty when asked (later)
*   execute mysql_secure_installation and set parameter
    *   auth_socket (Only allow local users to connect. Use the same mysql username as the unix username)
    *   PAM : Allows for example an LDAP backed system and use that to authenticate.
    *   caching_sha2_password is the most secure but needs updated client drivers
*   `CREATE USER '<user>'@'localhost' IDENTIFIED BY 'password';`
*   `GRANT ALL PRIVILEGES ON *.* TO '<user>'@'localhost' WITH GRANT OPTION;`
*   Import dump `mysql < dump.sql`
    *   Useful commands:
        *   `SHOW DATABASES;`
        *   `USE tablename;`
        *   `SHOW tables;`
        *   `SHOW columns FROM user;`
    *   Create db user for the app
        *   `CREATE USER '<appuser>'@'localhost' IDENTIFIED BY 'password';`
        *   `GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, INDEX, DROP, ALTER, CREATE TEMPORARY TABLES, LOCK TABLES ON <projectDb>.* TO '<appuser>'@'localhost';`
    *  Verify with
        *   `SELECT user,authentication_string,plugin,host FROM mysql.user;`


# Setup GIT deployment repository

*   Create the server folders
    *   `sudo mkdir -p /var/www/<project>/<branch>/[client|server]/serve`
    *   `sudo mkdir -p /var/www/<project>/<branch>/[client|server]/tmp`
*   Set permissions so our users can access these folders. The git hook is run with the current user so if we want to deploy here we need to have write permission.
    *   `sudo chgrp -R dev /var/www/<project>/<branch>`
    *   `sudo chmod g+w /var/www/<project>/<branch>`
*   Setup the deployment git repository
    *   `sudo mkdir -p /var/git/<project>-[client|server].git`
    *   `cd /var/git/<project>-[client|server].git`
    *   `sudo git init --bare`
*   Setup permissions so developers can push without elevated access rights
    *   `cd /var/git/<project>-[client|server].git`
    *   `sudo chmod -R g+rw .`
        *   Add **r**ead **w**rite and a**cc**ess rights for the **g**roup to all folders in the current directory **-R**ecursively
    *   `sudo chmod g-w objects/pack/*`
        *   [Pack files](https://git-scm.com/book/en/v2/Git-Internals-Packfiles) should be immutable.
    *   `sudo find . -type d -exec chmod g+s {} +`
        *   We get all folders (-type d) in the current directory (.) and set the groupID/setgid bit of the folder (chmod g+s). All directories found by find are passed as one ({} +) instead of executing the command on every found directory seperatly ({} \;). See [GNU](https://www.gnu.org/software/coreutils/manual/html_node/Directory-Setuid-and-Setgid.html). Setting the gid flag on a directory has the effect of every newly created file/folder inheriting its group from the currently set group istead of the one from the user creating the file/folder
    *   `sudo git config core.sharedRepository group`
        *   Allow users of the same group as the repo to push


# Creating a deploy hook

*   `cd /var/git/<project>-[client|server].git/hooks`
*   `sudo touch post-receive`
*   `sudo chmod +x post-receive`

For our client:

```sh
#!/bin/sh

USER=`whoami`
SERVER=`hostname`

TARGET="/var/www/<project>/<branch>/client/serve"
TEMP="/var/www/<project>/<branch>/client/tmp"
DIST="/var/www/<project>/<branch>/client/tmp/dist"
REPO="/var/git/<project>.git"

echo "Hello $USER at $SERVER"
echo "You are about to publish from:"
echo "  $REPO"
echo "to"
echo "  $TARGET"

mkdir -p $TEMP

echo "Cecking out repo"
git --work-tree=$TEMP --git-dir=$REPO checkout -f beta
if [ "$?" -ne "0" ]; then
    echo "Git checkout failed!"
    exit 1
fi

cd $TEMP
echo "Running NPM install"
npm install --silent
if [ "$?" -ne "0" ]; then
    echo "NPM install failed!"
    exit 1
fi

echo "Building application"
npm run --silent build
if [ "$?" -ne "0" ]; then
    echo "Build failed!"
    exit 1
fi

cd /
rm -rf $TARGET
mv $DIST $TARGET
rm -rf $TEMP

echo "Fixing up group permissions"
chgrp -R www-data $TARGET
if [ "$?" -ne "0" ]; then
    echo "Changing group to www-data failed!"
    exit 1
fi

echo "Fixing up index.html to 404.html, for SPA usage"
cp $TARGET/index.html $TARGET/404.html
if [ "$?" -ne "0" ]; then
    echo "Error while copying index to 404 for SPA usage"
    exit 1
fi

echo "Successfully published"
```

And respectevly for our server:

```sh
#!/bin/sh

USER=`whoami`
SERVER=`hostname`

TARGET="/var/www/<project>/<branch>/server/serve"
TEMP="/var/www/<project>/<branch>/server/tmp"
REPO="/var/git/<project>-<server>.git"

echo "Hello $USER at $SERVER"
echo "You are about to publish from:"
echo "  $REPO"
echo "to"
echo "  $TARGET"

mkdir -p $TEMP

echo "Cecking out repo"
git --work-tree=$TEMP --git-dir=$REPO checkout -f beta
if [ "$?" -ne "0" ]; then
    echo "Git checkout failed!"
    exit 1
fi

cd $TEMP
echo "Running NPM install"
npm install --silent
if [ "$?" -ne "0" ]; then
    echo "NPM install failed!"
    exit 1
fi

echo "Building application"
npm run --silent build
if [ "$?" -ne "0" ]; then
    echo "Build failed!"
    exit 1
fi

cd /
rm -rf $TARGET
mv $TEMP $TARGET
rm -rf $TEMP

echo "Copying production config"
cp $CFG $TARGET/config/
if [ "$?" -ne "0" ]; then
    echo "Copying server config failed!"
    exit 1
fi

echo "Copying ORM config"
cp $ORMCFG $TARGET/
if [ "$?" -ne "0" ]; then
    echo "Copying orm config failed!"
    exit 1
fi

echo "Generate JWT Secret"
cat /dev/urandom | base64 | head -c 512 > $TARGET/config/secret.key
if [ "$?" -ne "0" ]; then
    echo "Error while generating JWT secret"
    exit 1
fi

echo "Fixing up group permissions"
chgrp -R www-data $TARGET
if [ "$?" -ne "0" ]; then
    echo "Changing group to www-data failed!"
    exit 1
fi

echo "Restarting server"
sudo /bin/systemctl restart <perojcet>-server-<branch>
if [ "$?" -ne "0" ]; then
    echo "Error while restarting the service!"
    exit 1
fi

echo "Successfully published"
```

Add a simple systemd unit file.
Client:

```
[Unit]
Description=Serve static client package
After=network.target

[Service]
Type=simple
Restart=always
RestartSec=1
User=www-data
Group=www-data
WorkingDirectory=/var/www/<project>/<branch>/client/serve
ExecStart=/usr/bin/http-server -p 3005 --log-ip

[Install]
WantedBy=multi-user.target
```

Server:

```
[Unit]
Description=Backend for <project>
After=mysql.service

[Service]
Type=simple
Restart=always
RestartSec=1
User=www-data
Group=www-data
Environment=NODE_ENV=production
Environment=PORT=3249
WorkingDirectory=/var/www/<project>/<branch>/server/serve
ExecStart=/usr/local/bin/node /var/www/<project>/<branch>/server/serve/build/app.js

[Install]
WantedBy=multi-user.target
```

Add the remote to our local repository

*   `git remote add deploy-dev ssh://<user>@<hostname>/var/git/<project>-[client|server].git/`


This is for your development environment only. On a production server you should probably use nginx or apache. For development builds I usually just set up a simple http server. Either the python simple http server or in this case the http-server npm package.

*   `npm install --global http-server`

Create a systemd unit to manage the http-server

*   `touch /etc/systemd/system/<project>-<branch>.service`

```
[Unit]
Description=Serve static client package
After=network.target

[Service]
Type=simple
Restart=always
RestartSec=1
User=www-data
Group=www-data
WorkingDirectory=/var/www/<project>/<branch>/client/serve
ExecStart=/usr/bin/http-server -p 3005

[Install]
WantedBy=multi-user.target
```

In order for our users to start/stop and restart this service we need to add certain rights for the group to our sudoers file /etc/sudoers:

```
%developers 	ALL= NOPASSWD: /bin/systemctl restart <project>-<branch>
%developers 	ALL= NOPASSWD: /bin/systemctl start <project>-<branch>
```

This tells sudo that everyone in the group `developers` is allowed to execute `/bin/systemctl [start|restart] <project>-<branch>` without the need to enter their password `NOPASSWD`. It's still important to execute those paths with `sudo` so this rule comes to effect.

Now we see why I copied the index.html to 404.html. If you develope a SPA and your preferred router works with the session history stack i.E. changes the URL instead of the URL hash you need to server the index.html on any path. http-server serves 404.html if the path does not exist which solves our problem. On nginx we usually use this setting:

```
location / {
  try_files $uri $uri/ /index.html;
}
```

Lastly I like to set up a reverse proxy to serve my applications over TLS. This way you have a central point of managing your certificates. This certificate is valid for all subdomains so you can add applications without the hassle of setting up a new certificate. 
For my dev server I like to use redbird because of how easy it is to get running. It even has automated zero configuration letsencrypt capability. I manage a few more certificates so I chose to generate my certificate manually and pass them to redbird.

*   `npm install redbird`

```sh
#!/usr/bin/env node
var redbird = new require('redbird')({
    	port: 80,
    	ssl: {
            	port: 443,
            	key: "/etc/certificates/private.key",
            	cert: "/etc/certificates/full.crt",
            	ca: "/etc/certificates/full.crt",
    	}
});

redbird.register('<project_1>.<hostname>.com', 'http://127.0.0.1:3004', {ssl: true});
redbird.register('<project_2>.<hostname>.com', 'http://127.0.0.1:3005', {ssl: true});
```

And again a systemd unit file to start up the reverse proxy.
```
[Unit]
Description=Reverse Proxy
After=network.target

[Service]
Environment=NODE_ENV=production
Environment=PATH=/usr/local/bin
ExecStart=<full_path_to_proxy>/main.js
Type=simple
User=www-data
Group=www-data
Restart=always

[Install]
WantedBy=multi-user.target
```

Lastly this is how I update my certificates. You should porbably use certbots autorenewal but since my domain name isn't pointed to this server I generate them manually for my subdomains.

*   `./certbot-auto certonly --manual -d <domain>.<tld>`


Subnote: Push with Windows 
Put your private key in C:\Users\<user>\.ssh\id_rsa. On Windows the easiest way is probably using putty-gen and exporting the private key as OpenSSH key.

Subnote: Webpack
I often see  developers importing static assets directly in their source. Don't do this! Use imports/require for those files and let webpack file-loader plugin handle those files. This way you don't need to copy any assets folders and have hashed filenames for your cache control!

Nextup: Going from dev to beta which include changes from redbird and http-server to nginx!
