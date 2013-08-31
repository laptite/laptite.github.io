---
layout: post
title: "rails deployment to digital ocean ubuntu via capistrano"
author: "Alisa Chang"
date: 2013-08-13 18:58
comments: true
categories: rails, deployment
---

{% img left /images/holy-cow.png 194 187 [Pixel Img] %}
We are in the 11th week of Flatiron School. It has been a whirlwind of learning, and it feels great to finally deploy our first Rails app today. Deployment was painful, so I'm posting this walk-through in hopes that it might save someone some deployment time. My team started this process with high hopes of crushing it in 45 minutes. Instead we ran into all sorts of annoying problems to debug for the next 3 hours (ugh!).

Some hurdles that were overcome and that are discussed in this post are: asset pre-compilation errors, Sunspot solr gem headaches, Database not seeding, and image_tag 'isn't precompiled' error message.

This walk-through builds on an excellent tutorial by guest speaker <a href="https://github.com/spikegrobstein/flatironschool-deployment_lecture/blob/master/lecture.md">Spike Grobstein</a>. My intent is not to plagiarize any of that work, but instead to add in some steps that were needed for me to get our application deployed and working. 

We were provided the following information by our TA Blake: (a) Digital Ocean droplet to serve our app, and (b) credentials (IP Address / Username: root / Password). He then sent us packing to work on our own because he's a meanie (just kidding). One skill you'll sharpen while in this program is your ability to walk in the dark. Not to worry though. There is light to be found for those who persevere. Anyway, I digress.

Before you start anything you'll want to make sure that all working git branches are up-to-date, reviewed, merged and pushed up to master accordingly.  This may not be a big concern if you're working solo, but a critical step for our team of four sharing workload. 

Open terminal, cd into your Rails app and let's get ready to rumble.

<strong>PHASE 1: Connect to your spankin' new server</strong><br>
<em>In the code base a hashtag # represents a comment, so don't type it - ok?</em>

Connect to your new server via SSH:
    ssh root@XXX.XXX.XXX.XXX # plug your IP address in the place of the X's

Answer questions politely:
    Are you sure you want to continue connecting (yes/no)? # type yes and press enter

Root is a super user, so you should really create a user for the app since it brings bad mojo on top of 7 years of bad luck to do otherwise - it's bad practice, so don't be that person:
    useradd -s /bin/bash -G sudo -m outlinked # note: our app is called "outlinked", enter username.

Then you have to create a password for this user, but you won't be entering the password right away. You're going to tell the server which username you want to apply the password to:
    passwd outlinked

When prompted for a password, choose one and confirm it.

Groovy, now exit the server and connect as your alter-ego (aka your new username):

    ssh outlinked@XXX.XXX.XXX.XXX # you know what to do with these X's now

Enter your password, and get ready to install the packages your app will need to run on your Ubuntu server.

<strong>PHASE 2: Get your server ready for your awesome application</strong>

Update your apt-repository and dependencies with this code:
    sudo apt-get update

Then upgrade all installed packages with this code:
    sudo apt-get upgrade

You'll be shown a list of packages that need upgrading, and then asked to confirm. Be polite and answer in the affirmative:

    Y # press enter

Install build essential package:
    sudo apt-get install build-essential

Install Ruby packages:
    sudo apt-get install ruby1.9.3 sqlite3 libsqlite3-ruby1.9.1 libsqlite3-dev

Install Git:
    sudo apt-get install git

Install XML libraries (this one is for gems using the stated libs e.g. Nokogiri):
    sudo apt-get install libxml2-dev libxslt1-dev

Install Gem Bundler:
    sudo gem install bundler

Install SQLite3:
    sudo gem install sqlite3

Install nodejs (avoids compile errors with the Javascript framework):
    sudo apt-get install nodejs

Install Capistrano, which you'll be using to deploy your app:
    gem install capistrano

Install Passenger, which is an nginx module and your Rack webserver:
    sudo gem install passenger
    sudo passenger-install-nginx-module
    sudo apt-get install libcurl4-openssl-dev libssl-dev zlib1g-dev
    sudo nano /opt/nginx/conf/nginx.conf # configure nginx
    # scroll through the nano file looking for #user  nobody;
    # uncomment that, and change nobody; to www-data;
    # keep scrolling and find the location block now
    # Replace the whole block:
      location / {
       root   html;
      index  index.html index.htm;
      }
    # with:
      root /home/USERNAME/APPNAME/current/public;
      passenger_enabled on;
    # Exit the editor (Ctrl + X)
    sudo ln -s /opt/nginx/sbin/nginx /usr/local/sbin/
    sudo nginx # Start nginx
    (To stop ngingx: sudo nginx -s stop)

Congratulations, you're well on your way to making your app dreams a reality!

<strong>PHASE 3: Application config files &amp; Deployment</strong>

Go ahead an open a new terminal tab so that you can cd back into your application's repo on your local drive. All Capistrano needs for you to type in the command line to start the deployment process is:
    capify .

Excellent, let's update a few settings in our application to enable deployment.

Navigate to the config directory, open deploy.rb and edit it as follows:
    set :application, "outlinked" # Set application to USERNAME

    # Enter your github ssh handle (the https one didn't work for us)
    set :repository,  "git@github.com:flatiron-school/library-redux.git"

    set :user, 'outlinked' # Set user to USERNAME

    set :deploy_to, "/home/outlinked/outlinked" # Set deploy_to to /home/USERNAME/APPLICATION

    set :user_sudo, false
    
    set :rails_env, "production" # sets your server environment to Production mode

    set :scm, :git  # sets version control

    default_run_options[:pty] = true
    
    role :web, "XXX.XXX.XXX.XXX" # Your HTTP server, Apache/etc
    role :app, "XXX.XXX.XXX.XXX" # We made the app role the same as our `Web` server 
    
    # We made the database role the same as our our `Web` server 
    role :db,  "XXX.XXX.XXX.XXX", :primary => true # This is where Rails migrations will run

    # We're using Passenger mod_rails, so we uncommented this:
    namespace :deploy do
      task :start do ; end
      task :stop do ; end
      task :restart, :roles => :app, :except => { :no_release => true } do
        run "#{try_sudo} touch #{File.join(current_path,'tmp','restart.txt')}"
      end
    end

Cool. We're getting closer to the promise land.

Now go into your Capfile in your application, and uncomment the following:
    load 'deploy/assets' # this will precompile your assets every time you deploy

Oh yeah, don't forget to add and commit your changes:
    git add .
    git commit -am "edit deploy.rb"
    git push

My team had to generate a deploy key because our application sits in a private Github repo. If this doesn't apply to you, keep reading because one day you'll need to do this. Switch your terminal tab so that you're back into your server command line and enter this code:
    ssh-keygen -t rsa -b 4096 # Leave all requests for passwords blank! 
                              # Hit enter until no more questions are asked.

Now you'll need to open your browser and navigate to your repo in Github. Click on 'Settings' and then on 'Deploy key', and then finally on 'Add deploy key'. Enter your github password, and go back to your server terminal to retrieve the key to copy/paste into github:
    cat ~/.ssh/id_rsa.pub # copy everything from the ssh-rsa to username@host.
                          # I first opened this file with nano, which only showed a partial key.
                          # Don't enter a title for the key. Github will use your username.

Awesome. Now jump back into your other terminal window where you're in your local repo and run:
    cap deploy:setup

This connects you to your server, prompts you for your password and creates your directory structure on the server. When you cd into your USERNAME directory in your server, you'll see three sub-directories: current, releases and shared. The current directory is the server equivalent of your application directory in your local directory. 

I should mention that a log folder wasn't immediately available in the current directory. This became an issue when I ran into a problem deploying and I couldn't get to my production.log to review error messages. I got help from our TA Joe for this, and there was some light-fingered magic going on, so I don't have exact code to post. My understanding is that the Production.log file in the current directory points via symlink to a log file in the shared directory, which did not exist. You can check this status by doing an <code>ls -l</code> and looking for line items in red. 

<strong>Guess what? It's time to deploy - Heck yeah!</strong><br>
So you've done like a million steps already and we're just deploying now. Pat yourself on the back and give yourself a high five. 

All your changes are committed, right? Cool. You're ready to deploy. From your local repo terminal, enter the magical Capistrano command while singing O Sole Mio:
    cap deploy

Open your browser, type in your IP address in the url bar <code>http://[your IP]</code>

Celebration time, right? Wrong. This is where error messages plagued me. 

**Important note:** you know all that stuff you have to do in your local Rails app like bundle install and all that jazz? That doesn't magically happen in the server. All the routines required in your local drive, you're going to have to repeat here.

**Problem no. 1 - Sunspot gem**
We installed this gem so that users can run searches in our app. This gem was a pain in the ass from day one. It runs a Java server and it's been a challenge to update every time we changed seats during pair programming. A good rule of thumb is that whatever you've had to do to fix problems in your local drive, you'll likely have to repeat in your server environment. For one you have to install a Java environment:
    sudo apt-get install openjdk-6-jdk
Next you'll need to cd into your app USERNAME/current and do all that other junk that fixes rake errors:
    rails g sunspot_rails:install RAILS_ENV='production'
    rake sunspot:solr:start RAILS_ENV='production'
    rake sunspot:reindex RAILS_ENV='production'

The reason I'm tacking on RAILS_ENV='production' at the end of each rake command is that I haven't set up rake tasks in my config file in Rails to default to production mode. I figured this out after numerous failed attempts at seeding. According to my rails console the data was seeding, but it turns out the simple rake command populates the development database and not the production one. Did you know that typing <code>rails c</code> in the server gets you the development database and <code>rails c production</code> gets you the production database? Holy batman, it's a thing! Lesson learned after beating my head against the wall for about an hour. 

Time to setup the tables and seed the files:
    bundle exec rake db:seed RAILS_ENV='production'
    rake db:migrate RAILS_ENV='production'

Restart the server now by typing:
    touch tmp/restart.txt # if you get a permissions error: sudo touch tmp/restart.txt

Reload your browser and voilà! Did it work? For me it still didn't!

Merde! Pardon my french, but this is wholly aggravating by this time. Thank goodness for TA Ashley for helping me retain my sanity. 

Here's the error message I was getting:
    ActionView::Template::Error ( isn't precompiled):
        17: (...)
        20:   <%= image_tag("#{topic_link.link.user.image}", size: "25x25") %>

I was stuck on this Mo-Fo for at least an hour because every new person that swooped in concluded that it was a pre-compiling error. So round and round in circles I went. It turns out that this is a legit Rails issue that was brought up on github/Rails. Thank you Ashley, I'm not a complete dope! The story here is this:

My team didn't set a default image for avatars. Instead we just had an erb link in line 20. When there is no image available, there's a nil in this field. This was no big deal in development in our local drive, but in Production mode: no-likey. So we tried to put 'if statements' to by-pass the error, but it didn't like that either. Long story short, we deleted the line in the short-term so that we could complete deployment. Tomorrow, I'll add a default image to my assets, which we can link to if image is nil. 

In the meantime? Success is sweet!
