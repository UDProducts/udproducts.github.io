---
layout: post
title: "Rails deployment with unicorn and nginx"
description: ""
category: articles
tags: 
image:
  feature: texture-feature-03.jpg
  comments: true
  share: true
---

### Intro:

Most of the Rails application deployed are with nginx and unicorn configuration, So sharing nginx and unicorn configuration we will need to make the rails application serve at HTTP port 80.

Here im not going to create rails app or configure any VPS server,We can get all the details like capistrano, setting up the server details from this [post](http://udproducts.github.io/articles/rails-deploy-with-passenger/). So we are going to have just nginx configuration and unicorn configuration here.

From previous post we will have to add unicorn to your Gemfile, bundle it and deploy.

    gem 'unicorn'

Now we will have a unicorn configuration inside the app/config/unicorn.rb

	root = "/home/ubuntu/apps/app_name/current"
	working_directory root
	pid "#{root}/tmp/pids/unicorn.pid"
	stderr_path "#{root}/log/unicorn.log"
	stdout_path "#{root}/log/unicorn.log"

	listen "/tmp/unicorn.app_name.sock"
	worker_processes 10
	timeout 30

	preload_app true

	before_fork do |server, worker|
	  if defined?(ActiveRecord::Base)
	    ActiveRecord::Base.connection.disconnect!
	  end
	  old_pid = "/tmp/unicorn.my_site.pid.oldbin"
	  if File.exists?(old_pid) && server.pid != old_pid
	    begin
	      Process.kill("QUIT", File.read(old_pid).to_i)
	    rescue Errno::ENOENT, Errno::ESRCH
	    end
	  end
	end

	after_fork do |server, worker|
	  if defined?(ActiveRecord::Base)
	    ActiveRecord::Base.establish_connection
	  end
	end

To start the rails app we will need to create the init file and place it in /etc/init.d/appname



	#!/bin/sh
	### BEGIN INIT INFO
	# Provides:          unicorn
	# Required-Start:    $remote_fs $syslog
	# Required-Stop:     $remote_fs $syslog
	# Default-Start:     2 3 4 5
	# Default-Stop:      0 1 6
	# Short-Description: Manage unicorn server
	# Description:       Start, stop, restart unicorn server for a specific application.
	### END INIT INFO
	set -e

	# Feel free to change any of the following variables for your app:
	TIMEOUT=${TIMEOUT-60}
	APP_ROOT=/home/ubuntu/apps/app_name/current
	PID=$APP_ROOT/tmp/pids/unicorn.pid
	CMD="cd $APP_ROOT; bundle exec unicorn -D -c $APP_ROOT/config/unicorn.rb -E production"
	AS_USER=ubuntu
	set -u

	OLD_PIN="$PID.oldbin"

	sig () {
	  test -s "$PID" && kill -$1 `cat $PID`
	}

	oldsig () {
	  test -s $OLD_PIN && kill -$1 `cat $OLD_PIN`
	}

	run () {
	  if [ "$(id -un)" = "$AS_USER" ]; then
	    eval $1
	  else
	    su -c "$1" - $AS_USER
	  fi
	}

	case "$1" in
	start)
	  sig 0 && echo >&2 "Already running" && exit 0
	  run "$CMD"
	  ;;
	stop)
	  sig QUIT && exit 0
	  echo >&2 "Not running"
	  ;;
	force-stop)
	  sig TERM && exit 0
	  echo >&2 "Not running"
	  ;;
	restart|reload)
	  sig USR2 && echo reloaded OK && exit 0
	  echo >&2 "Couldn't reload, starting '$CMD' instead"
	  run "$CMD"
	  ;;
	upgrade)
	  if sig USR2 && sleep 2 && sig 0 && oldsig QUIT
	  then
	    n=$TIMEOUT
	    while test -s $OLD_PIN && test $n -ge 0
	    do
	      printf '.' && sleep 1 && n=$(( $n - 1 ))
	    done
	    echo

	    if test $n -lt 0 && test -s $OLD_PIN
	    then
	      echo >&2 "$OLD_PIN still exists after $TIMEOUT seconds"
	      exit 1
	    fi
	    exit 0
	  fi
	  echo >&2 "Couldn't upgrade, starting '$CMD' instead"
	  run "$CMD"
	  ;;
	reopen-logs)
	  sig USR1
	  ;;
	*)
	  echo >&2 "Usage: $0 <start|stop|restart|upgrade|force-stop|reopen-logs>"
	  exit 1
	  ;;
	esac


Atlast we will need to configure the nginx configuration /etc/nginx/sites-available/default


	upstream unicorn {
	  server unix:/tmp/unicorn.app_name.sock fail_timeout=0;
	}

	server {
	  listen 80 default deferred;
	  server_name x.x.x.x;
	  root /home/ubuntu/apps/app_name/current/public;

	  location ^~ /assets/ {
	    gzip_static on;
	    expires max;
	    add_header Cache-Control public;
	  }

	  try_files $uri/index.html $uri @unicorn;
	  location @unicorn {
	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	    proxy_set_header Host $http_host;
	    proxy_redirect off;
	    proxy_pass http://unicorn;
	  }

	  error_page 500 502 503 504 /500.html;
	  client_max_body_size 4G;
	  keepalive_timeout 10;
	}


Then restart the nginx server. We can start the unicorn with /etc/init.d/app_name start