# How-To-Deploy-Express-To-APACHE-Server

## 1. Install and Setup PM2
```dash
npm install -g pm2

sudo pm2 start app.js (run it as root)
pm2 startup systemd
sudo env PATH=$PATH < PM2 path >
```
The benifit of using PM2 is:
1. Once started, your app is forever alive, auto-restarting across crashes and machine restarts.
2. All your applications are run in the background and can be easily managed.PM2 creates a list of processes, that you can access with:
```dash
pm2 ls
```
3. Application logs are saved in the hard disk of your servers into ~/.pm2/logs/.
Access your realtime logs with:
```dash
pm2 logs <app_name>
```
4. In-terminal monitoring
```dash
pm2 monit
```
5. Automate your deployment and avoid to ssh in all your servers one by one
```dash
pm2 deploy
```
6. stop pm2
```dash
pm2 
```
## 2 Setting up a LaunchDaemon with pm2, so the server will kepp running even after logged-out on this computer
### 2.1 Create a new LaunchDaemon config file for your app - 
````sudo vi /Library/LaunchDaemons/org.node.xyz.plist````
### 2.2 Add the contents of ```org.node.xyz.plist```, altering it with your configuration, and save (````:wq````) and exit
```dash
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>Label</key>
        <string>org.node.xyz</string>
        <key>UserName</key>
        <string>username</string>
        <key>KeepAlive</key>
        <true/>
        <key>ProgramArguments</key>
        <array>
            <string>/usr/local/Homebrew/lib/node_modules/pm2/bin/pm2</string>
            <string>start</string>
            <string>/Users/lucy_l/Sites/<Your Project xyz>/app.js</string>
            <string>--name</string>
            <string><Your Project Name XYZ></string>
            <string>--env</string>
            <string>production</string>
        </array>
        <key>RunAtLoad</key>
        <true/>
	<key>EnvironmentVariables</key>
    <dict>
      <key>PATH</key>
      <string>/usr/local/Homebrew/Cellar/node/10.3.0/bin</string>
      <key>PM2_HOME</key>
      <string>/Users/<username>/.pm2</string>
    </dict>
	  <key>StandardErrorPath</key>
	  <string>/tmp/com.PM2.err</string>
	  <key>StandardOutPath</key>
	  <string>/tmp/com.PM2.out</string>

    </dict>
</plist>
```
### 2.3 ````sudo chown root:wheel /Library/LaunchDaemons/org.node.xyz.plist````
### 2.4 ````sudo chmod 644 /Library/LaunchDaemons/org.node.xyz.plist````
### 2.5 Find your path by typing ````echo $PATH````
### 2.6 ````sudo vi /etc/launchd.conf````
### 2.7 Add the contents of ````launchd.conf````, but use whatever ````$PATH```` you just echoed, then save and exit
```dash
setenv PATH /bin:/sbin:/usr/bin:/usr/sbin:/opt/local/bin:.:/opt/local/bin/:/usr/local/bin/
```
### 2.8 ````sudo shutdown -r now```` - this will reboot the system. If you skip this step, the changes to ````launchd.conf```` will not take effect.
### 2.9 Log back into the machine
### 2.10 ````sudo launchctl load /Library/LaunchDaemons/org.node.xyz.plist````
### 2.11 Verify that it loaded correctly - ````sudo launchctl list | grep xyz````. Three columns will be returned - ````PID````, ````status````, and ````label````. If there is no value for ````PID````, but there is a value for ````status````, something went wrong.
### 2.12 If everything went according to plan, you should now be able to ````pm2 list```` and see your Node app running. If you get an error, you might need to ````sudo chmod 777 /Users/<<me>>/.pm2/pm2.log````.

## 3 Setup Apache Server and Setup Virutalhost
### 3.1 on /etc/apache2/httpd.conf, uncomment following:
```bash
LoadModule php5_module libexec/apache2/libphp5.so
LoadModule proxy_module libexec/apache2/mod_proxy.so
LoadModule proxy_connect_module libexec/apache2/mod_proxy_connect.so
LoadModule proxy_ftp_module libexec/apache2/mod_proxy_ftp.so
LoadModule proxy_http_module libexec/apache2/mod_proxy_http.so
LoadModule proxy_fcgi_module libexec/apache2/mod_proxy_fcgi.so
LoadModule proxy_scgi_module libexec/apache2/mod_proxy_scgi.so
```
```bash     
     #
     # DocumentRoot: The directory out of which you will serve your
     # documents. By default, all requests are taken from this directory, but
     # symbolic links and aliases may be used to point to other locations.
     #
	DocumentRoot "/Users/<To Your Project>/Sites"
	<Directory "/Users/l<To Your Project>/Sites">
	    #
	    # Possible values for the Options directive are "None", "All",
	    # or any combination of:
	    #   Indexes Includes FollowSymLinks SymLinksifOwnerMatch ExecCGI MultiViews
	    #
	    # Note that "MultiViews" must be named *explicitly* --- "Options All"
	    # doesn't give it to you.
	    #
	    # The Options directive is both complicated and important.  Please see
	    # http://httpd.apache.org/docs/2.4/mod/core.html#options
	    # for more information.
	    #
	    Options FollowSymLinks Multiviews
	    MultiviewsMatch Any

	    #
	    # AllowOverride controls what directives may be placed in .htaccess files.
	    # It can be "All", "None", or any combination of the keywords:
	    #   AllowOverride FileInfo AuthConfig Limit
	    #
	    AllowOverride All

	    #
	    # Controls who can get stuff from this server.
	    #
	    Require all granted
	</Directory>
```

### 3.2 on /etc/apache2/extra/httpd-vhosts.conf    
```bash  
<VirtualHost *:80>
    DocumentRoot "/Users/<To Your Project>"
    DirectoryIndex "views/index.ejs"
    ServerName "*****.com"
    ServerAlias "www.*****.com"
    ErrorLog "/private/var/log/apache2/******_log"
    CustomLog "/private/var/log/apache2/******_log" common

    ProxyRequests Off
    ProxyPreserveHost On
    ProxyVia Full
    <Proxy *>
      Order deny,allow
      Allow from all
    </Proxy>

    <Location />
      ProxyPass http://127.0.0.1:2200/
      ProxyPassReverse http://127.0.0.1:2200/
    </Location>

    <Directory "/Users/<Path To Your Project>/">
    #
    # Possible values for the Options directive are "None", "All",
    # or any combination of:
    #   Indexes Includes FollowSymLinks SymLinksifOwnerMatch ExecCGI MultiViews
    #
    # Note that "MultiViews" must be named *explicitly* --- "Options All"
    # doesn't give it to you.
    #
    # The Options directive is both complicated and important.  Please see
    # http://httpd.apache.org/docs/2.4/mod/core.html#options
    # for more information.
    #
    Options All
    #MultiviewsMatch Any

    #
    # AllowOverride controls what directives may be placed in .htaccess files.
    # It can be "All", "None", or any combination of the keywords:
    #   AllowOverride FileInfo AuthConfig Limit
    #
    AllowOverride All

    #
    # Controls who can get stuff from this server.
    #
    #Order allow,deny
    #Allow from all

    Require all granted
    </Directory>

</VirtualHost>
```

