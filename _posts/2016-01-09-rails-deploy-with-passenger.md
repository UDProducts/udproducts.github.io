---
layout: post
title: "Rails with passenger and nginx"
description: ""
category: articles
tags: 
image:
  feature: texture-feature-03.jpg
  comments: true
  share: true
---
### Intro

I have been using nginx and unicorn combination in the vps server for my rails production application, and recently when working in a new project i found out that passenger was running on it and i wanted to give it a try. So lets start from first. Create a  new rails application

    rails new deploy_with_passenger
	rails g controller home index

change routes

    root home#index

### Setup rails app for deployment

Add passenger gem to your gem file

    gem 'passenger'

Add capistrano to setup your app for deployment

    gem 'capistrano', '~> 3.4.0'
    gem 'capistrano-rails'
    gem 'capistrano-passenger'

Then we have to execute 

    bundle exec cap install

### Configure deploy.rb

add this lines

    set :passenger_restart_with_touch, true

and uncomment 

    set scm ...
    linked_files .................
    linked_dirs lines .................


Now configure deploy/production.rb

    set :bundle_without, %w{development test staging}.join(' ')

	set :stage, :production

	set :repo_url, 'git@github.com:aaa/ccc.git'

	set :branch, 'master'

	set :deploy_to, '~/deploy_with_passenger' # added for production server

	server = %w{ubuntu@x.x.x.x}

	role :app, server
	role :web, server
	role :db,  server
	set :rails_env, 'production'


### Setup production server

Now configure fresh production server. Installing dependencies for ruby

    sudo apt-get update
    sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3 -dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev python-software-properties libffi-dev

Install rvm 

    sudo apt-get install libgdbm-dev libncurses5-dev automake libtool bison libffi-dev
    curl -L https://get.rvm.io | bash -s stable
    source ~/.rvm/scripts/rvm
    rvm install 2.2.4
    rvm use 2.2.4 --default
    ruby -v

    echo "gem: --no-ri --no-rdoc" > ~/.gemrc
    gem install bundler

Installing Nginx and passenger

	sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7
	sudo apt-get install -y apt-transport-https ca-certificates

	# Add Passenger APT repository
	sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger trusty main > /etc/apt/sources.list.d/passenger.list'
	sudo apt-get update

	# Install Passenger & Nginx
	sudo apt-get install -y nginx-extras passenger

Open up nginx configuration ```/etc/nginx/nginx.conf  and uncomment out the following

    passenger_root /usr/lib/ruby/vendor_ruby/phusion_passenger/locations.ini;

    passenger_ruby /home/deploy/.rbenv/shims/ruby; # If you use rbenv


Now install postgres

    sudo apt-get install postgresql postgresql-contrib libpq-dev

Create the required database

    CREATE ROLE user_name;
    ALTER ROLE user_name WITH LOGIN PASSWORD 'password' NOSUPERUSER NOCREATEDB NOCREATEROLE;
    CREATE DATABASE database_name OWNER user_name;
    REVOKE ALL ON DATABASE database_name FROM PUBLIC;
    GRANT CONNECT ON DATABASE database_name TO database_user;
    GRANT ALL ON DATABASE database_name TO database_user;


required for installing json gem 

    sudo apt-get install libgmp3-dev

To fix runtime js error

    sudo apt-get install nodejs


Open up /etc/nginx/sites-enabled/default in your text editor and we will replace the file's contents with the following:

    server {
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

        server_name mydomain.com;
        passenger_enabled on;
        rails_env    production;
        root         /home/deploy/myapp/current/public;

        # redirect server error pages to the static page /50x.html
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }



 
#### Now deploy the app

Using capistrano deploy your app

    cap production deploy

