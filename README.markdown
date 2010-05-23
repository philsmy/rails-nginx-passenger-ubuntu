rails-nginx-passenger-ubuntu stack
============================

My notes on setting up a simple production server with Ubuntu 10.04 LTS, Nginx, Passenger, Ruby Enterprise Edition and Mysql for Rails 2.3.5

Aliases
-------

    echo "alias ll='ls -l'" >> ~/.bash_aliases
    
edit .bashrc and uncomment the loading of .bash_aliases

If you have trouble with PATH that changes when doing sudo, see http://stackoverflow.com/questions/257616/sudo-changes-path-why then add the following line to the same file

    echo "alias sudo='sudo env PATH=$PATH'" >> ~/.bash_aliases

Edit /etc/ssh/sshd_config and add the following to the bottom of the file:

    PermitUserEnvironment yes
Then reboot sshd by running:

    /etc/init.d/ssh reload
    

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

This should be installed before Ruby Enterprise Edition becouse that will install the mysql gem.

    sudo apt-get install mysql-server libmysqlclient15-dev    
Gemrc
-------

Add the following lines to ~/.gemrc, this will speed up gem installation and prevent rdoc and ri from being generated, this is not nessesary in the production environment.

    ---
    :sources:
    - http://gems.rubyforge.org
    - http://gems.github.com
    gem: --no-ri --no-rdoc


Ruby Enterprise Edition
------------------------

Check for newer version at [http://www.rubyenterpriseedition.com/download.html](http://www.rubyenterpriseedition.com/download.html)

Install package required by ruby enterprise, C compiler, Zlib development headers, OpenSSL development headers, GNU Readline development headers

    sudo apt-get install build-essential zlib1g-dev libssl-dev libreadline5-dev

Download and install Ruby Enterprise Edition

    wget http://rubyforge.org/frs/download.php/66162/ruby-enterprise-X.X.X-ZZZZ.ZZ.tar.gz
    tar xvfz ruby-enterprise-X.X.X-ZZZZ.ZZ.tar.gz 
    rm ruby-enterprise-X.X.X-ZZZZ.ZZ.tar.gz 
    cd ruby-enterprise-X.X.X-ZZZZ.ZZ/
    sudo ./installer
    
    
Change target folder to /opt/ruby for easier upgrade later on

Add Ruby Enterprise bin to PATH

    echo "export PATH=/opt/ruby/bin:$PATH" >> ~/.profile && . ~/.profile
    
Verify the ruby installation

    ruby -v
    ruby 1.8.7 (2009-06-12 patchlevel 174) [x86_64-linux], MBARI 0x6770, Ruby Enterprise Edition 20090928


Installing git
----------------

    sudo apt-get install git-core

Nginx
-------

Automatically install Nginx compiled with Passenger & SSL into /opt/nginx/

    /opt/ruby/bin/passenger-install-nginx-module --auto --prefix=/opt/nginx/ --auto-download --extra-configure-flags=--with-http_ssl_module


Nginx init script
-------------------

More information on http://wiki.nginx.org/Nginx-init-ubuntu

    cd ~/src
    git clone git://github.com/jnstq/rails-nginx-passenger-ubuntu.git
    sudo mv rails-nginx-passenger-ubuntu/nginx/nginx /etc/init.d/nginx
    sudo chown root:root /etc/init.d/nginx
    
Verify that you can start and stop nginx with init script

    sudo /etc/init.d/nginx start
    
      * Starting Nginx Server...
      ...done.
    
    sudo /etc/init.d/nginx status
    
      nginx found running with processes:  11511 11510
    
    sudo /etc/init.d/nginx stop
    
      * Stopping Nginx Server...
      ...done.
    
    sudo /usr/sbin/update-rc.d -f nginx defaults
    
If you want, reboot and see so the webserver is starting as it should.

Installning ImageMagick and RMagick
-----------------------------------

If you want to install the latest version of ImageMagick. I used MiniMagick that shell-out to the mogrify command, worked really well for me.

    # If you already installed imagemagick from apt-get
    sudo apt-get remove imagemagick

    sudo apt-get install libperl-dev gcc libjpeg62-dev libbz2-dev libtiff4-dev libwmf-dev libz-dev libpng12-dev libx11-dev libxt-dev libxext-dev libxml2-dev libfreetype6-dev liblcms1-dev libexif-dev perl libjasper-dev libltdl3-dev graphviz gs-gpl pkg-config

Use wget to grab the source from ImageMagick.org.

Once the source is downloaded, uncompress it:


    tar xvfz ImageMagick.tar.gz


Now configure and make:

    cd ImageMagick-6.5.0-0
    ./configure
    make
    sudo make install

To avoid an error such as:

convert: error while loading shared libraries: libMagickCore.so.2: cannot open shared object file: No such file or directory

    sudo ldconfig

Install RMagick
 
    sudo /opt/ruby/bin/ruby /opt/ruby/bin/gem install rmagick

Install Bundler
---------------

Bundler is arguably the best ruby gem manager ever written. Install it!

    sudo gem install bundler

Install Nokogiri
----------------

Nokogiri dependencies

    sudo apt-get install libxslt-dev libxml2-dev

Install Nokogiri gem

    sudo gem install nokogiri

Test a rails applicaton with nginx
----------------------------------

    rails -d mysql testapp
    cd testapp
    
Enter your mysql password
    
    vim database.yml
    rake db:create:all
    ruby script/generate scaffold post title:string body:text
    rake db:migrate RAILS_ENV=production
    
Check so the rails app start as normal
    
    ruby script/server

    sudo vim /opt/nginx/conf/nginx.conf
    
Add a new virutal host

    server {
        listen 80;
        # server_name www.mycook.com;
        root /home/deploy/testapp/public;
        passenger_enabled on;
    }
    
Restart nginx

    sudo /etc/init.d/nginx restart
    
Check you ipaddress and see if you can acess the rails application
        

