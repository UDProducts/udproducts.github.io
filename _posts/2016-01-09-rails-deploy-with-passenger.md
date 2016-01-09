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

I have been using nginx and unicorn combination in the vps server for my rails production application, and recently when working in a new project i found out that passenger was running on it and i wanted to give it a try.

So lets start from first. Create a  new rails application

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


bundle exec cap install


Then configure deploy.rb

add this lines

set :passenger_restart_with_touch, true

and uncomment set scm, linked_files and linked_dirs lines.


then configure deploy/production.rb

### Setup production server



Now Configure Fresh AWS server

sudo apt-get install libgmp3-dev

required for installing json gem 

To fix runtime js error

sudo apt-get install nodejs





 
#### Now deploy the app

Using capistrano deploy your app

cap production deploy

