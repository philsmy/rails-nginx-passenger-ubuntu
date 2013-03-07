My rails-NGINX-ree-passenger-ubuntu stack
============================

My notes on setting up a simple production server with Ubuntu 10.04 LTS, NGINX, Passenger, Ruby Enterprise Edition and Mysql for Rails 3.

This guide assumes you have installed Ubuntu 10.04 LTS (with no modules (optional)) on a server somewhere and you have access via sudo with the default ubuntu user.

AS UBUNTU - EARLY SETUP
-------
create admin user:

	useradd -m -k /etc/skel -d /home/admin -G sudo -s /bin/bash admin

visudo and put admin as rootable with no password. Put this at the end of the file!
	
	admin   ALL=(ALL) NOPASSWD: ALL

Setup the keys

	sudo mkdir /home/admin/.ssh
	sudo chown -R admin:admin /home/admin/.ssh
	sudo chmod 0700 /home/admin/.ssh
	sudo cp .ssh/authorized_keys /home/admin/.ssh/. # copy keys from root... maybe you need something different
	sudo chown -R admin:admin /home/admin/.ssh

Logoff and reconnect as admin

Aliases
-------

    echo "alias ll='ls -l'" >> ~/.bash_aliases
    
edit .bashrc and uncomment the loading of .bash_aliases

Update and upgrade the system
-------------------------------

    sudo apt-get update
    sudo apt-get upgrade

Configure timezone (If you didn't already when installing Ubuntu)
-------------------

    sudo dpkg-reconfigure tzdata
    sudo apt-get install ntp
    sudo ntpdate ntp.ubuntu.com # Update time
    
Verify that you have to correct date and time with

    date

Configure locale
----------------

First check the current locale

    /usr/bin/locale

Set if necessary

    sudo /usr/sbin/locale-gen en_US.UTF-8
    sudo /usr/sbin/update-locale LANG=en_US.UTF-8

Configure hostname
-------------------

    sudo hostname your-hostname

Add 127.0.0.1 your-hostname

    sudo vim /etc/hosts
    
Write your-hostname in 
    
    sudo vim /etc/hostname
    
Verify that hostname is set
    
    hostname

Install mysql
---------------

This should be installed before Ruby Enterprise Edition because that will install the mysql gem.

    sudo apt-get install mysql-server libmysqlclient15-dev   

Install Packages
----------------

	sudo apt-get install build-essential libssl-dev libreadline-dev libcurl4-openssl-dev libpcre3-dev

Gemrc
-------

Add the following lines to ~/.gemrc, this will speed up gem installation and prevent rdoc and ri from being generated, this is not nessesary in the production environment.

    ---
	:verbose: true
	:bulk_threshold: 1000
	install: --no-ri --no-rdoc --env-shebang
	:sources:
	- http://gemcutter.org
	- http://gems.rubyforge.org/
	- http://gems.github.com
	:benchmark: false
	:backtrace: false
	update: --no-ri --no-rdoc --env-shebang
	:update_sources: true


Capistrano environment fixes
----------------------------

If your deploying with Capistrano, you must modify a few things to get it to use Ruby Enterprise and load the local users environment.

Edit /etc/ssh/sshd_config and add the following to the bottom of the file:

    PermitUserEnvironment yes
Then reboot sshd by running:

    /etc/init.d/ssh reload

Configuring the PATH

Edit ~/.ssh/environment, and put something like this inside:

    PATH=/opt/ruby/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/opt/ruby/bin

**OR is it doesn't exist you can do this to add it with one command:**

    echo "PATH=/opt/ruby/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/opt/ruby/bin" > ~/.ssh/environment

Installing RVM
--------------

We already have rvm install scripts in our capistrano deploy. They came from [wayneeseguin on github](https://github.com/wayneeseguin/rvm-capistrano).
FYI to your deploy.rb just add these:

	set :rvm_ruby_string, ENV['GEM_HOME'].gsub(/.*\//,"")
	require "rvm/capistrano"                               # Load RVM's capistrano plugin.
    before 'deploy:setup', 'rvm:install_rvm'

Then you can do
	
	cap rvm:install_rvm
	cap rvm:install_ruby
	
To install first rvm and then the ruby you need

	vi ~/.rvmrc

To have

	rvm_trust_rvmrcs_flag=1
	
And

	sudo vi /etc/environment
	
To add

	RAILS_ENV=production
	
Finally

	rvm use --default ruby-1.9.3-p362
	
(Or whatever version you need)

Update RubyGems
---------------
    
    sudo gem update --system
	gem update

Installing git
----------------

    sudo apt-get install git-core

Passenger
---------

	gem install passenger

NGINX
-------

Automatically install NGINX compiled with Passenger & SSL into /opt/NGINX/

    rvmsudo passenger-install-nginx-module --auto --prefix=/opt/nginx/ --auto-download --extra-configure-flags="--with-http_ssl_module --with-http_gzip_static_module --without-mail_pop3_module --without-mail_smtp_module --without-mail_imap_module --with-http_stub_status_module"


NGINX init script
-------------------

More information on http://wiki.NGINX.org/NGINX-init-ubuntu

This command will download the latest version of my init script, copy it to /etc/init.d/nginx and update permissions.

    cd
	git clone git://github.com/jnstq/rails-nginx-passenger-ubuntu.git
	sudo mv rails-nginx-passenger-ubuntu/nginx/nginx /etc/init.d/nginx
	sudo chown root:root /etc/init.d/nginx
		
I always edit this file and add

	sleep 2
	
at line 233 (in restart)
    
Verify that you can start and stop NGINX with init script

    sudo /etc/init.d/nginx start
    
      * Starting nginx Server...
      ...done.
    
    sudo /etc/init.d/nginx status
    
      NGINX found running with processes:  11511 11510
    
    sudo /etc/init.d/nginx stop
    
      * Stopping nginx Server...
      ...done.

Add it to the startup routine:

    sudo /usr/sbin/update-rc.d -f nginx defaults
    
If you want, reboot and see so the webserver is starting as it should.

Installing ImageMagick and RMagick (Optional)
-----------------------------------

If you want to install the latest version of ImageMagick. I used MiniMagick that shell-out to the mogrify command, worked really well for me.

    # If you already installed imagemagick from apt-get
    sudo apt-get remove imagemagick

    sudo apt-get install libperl-dev gcc libjpeg62-dev libbz2-dev libtiff4-dev libwmf-dev libz-dev libpng12-dev libx11-dev libxt-dev libxext-dev libxml2-dev libfreetype6-dev liblcms1-dev libexif-dev perl libjasper-dev libltdl3-dev graphviz gs-gpl pkg-config

Use wget to grab the source from ImageMagick.org.

		ftp://ftp.imagemagick.org/pub/ImageMagick/ImageMagick.tar.gz

Once the source is downloaded, uncompress it:


    tar xvfz ImageMagick.tar.gz


Now configure and make:

    cd ImageMagick-6.ZZZZ
    ./configure
    make
    sudo make install

To avoid an error such as:

convert: error while loading shared libraries: libMagickCore.so.2: cannot open shared object file: No such file or directory

    sudo ldconfig

Install RMagick
 
    sudo /opt/ruby/bin/ruby /opt/ruby/bin/gem install rmagick

Install Bundler (Optional)
---------------

This is done automatically by passenger.

Install Nokogiri (Optional)
----------------

Nokogiri dependencies

    sudo apt-get install libxslt1-dev libxml2-dev

Install Nokogiri gem

    sudo gem install nokogiri

Update Path
-----------
If you have trouble with PATH that changes when doing sudo, see [http://stackoverflow.com/questions/257616/sudo-changes-path-why](http://stackoverflow.com/questions/257616/sudo-changes-path-why) then add the following line to the same file

    echo "alias sudo='sudo env PATH=$PATH'" >> ~/.bash_aliases

    
    
Config NGINX
----------------------------------

   
Add a new virtual host
	sudo vi /opt/nginx/conf/nginx.conf
	
Add our info. Grab the nginx.conf file from the doc/ of the source
    
Restart NGINX

    sudo /etc/init.d/nginx restart
    
Check your ip address and see if you can access the rails application
        

Create a Console File
---------------------

For convenience create a console file.

	vi ~/console
		
With this content

	#!/bin/sh

	export RAILS_ENV=production
	cd /u/apps/media_kontrol/current
	bundle exec rails c
		
You can no just log in and do ./console

Generate ssh keys
---------------------

Copy in the id_rsa and id_rsa.pub to ~/.ssh
		
check it works:

	git ls-remote ssh://git@mediacontrol.unfuddle.com/mediacontrol/mcr3.git master

Gems
----

We are now using bundler. So as a pre-emptive strike, create the gem directory:

	mkdir ~/.gems
	
Install Memcached
-----------------

Install and start:

	sudo apt-get install memcached
	memcached -d -u root
	
Log Rotate
----------

create a file in /etc/logrotate.d/passenger

	/u/apps/media_kontrol/shared/log/production.log {
	  daily
	  missingok
	  rotate 30
	  compress
	  delaycompress
	  sharedscripts
	  postrotate
	    touch /u/apps/media_kontrol/current/tmp/restart.txt
	  endscript
	}

create a file in /etc/logrotate.d/nginx

	/opt/nginx/logs/*.log {
	  daily
	  missingok
	  rotate 30
	  compress
	  delaycompress
	  sharedscripts
	  postrotate
	    /etc/init.d/nginx restart
	  endscript
	}

You can execute a debug run of logrotate with:

	logrotate -d /etc/logrotate.d/passenger
	logrotate -d /etc/logrotate.d/nginx

You can force an rotations with:

	logrotate -f /etc/logrotate.d/passenger
	logrotate -f /etc/logrotate.d/nginx

Install Whenever
-----------------
If this machine is to have whenever schedules

	sudo gem install whenever
	sudo ln -s /opt/ruby/bin/whenever /usr/local/bin/.
