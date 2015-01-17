---
layout: post
title: "Managing Rails Environment Variables"
date: 2015-01-16 08:48:03 -0700
comments: true
categories: rails
---
Environment variables are a set of dynamic named values that affect the way running processes behave on a computer.
In almost all operating systems each process has a private set of environment variables, and by default when a
process is created it inherits a duplicate environment from the parent process, except where explicit changes are made.

Typically environment variables are set in your `.bash_profile`, `.bashrc`, or `.zshrc`.

Examples include `$PATH, $PWD, $SHELL, $EDITOR`.  Typing `ENV` in your terminal will show you all the environment
variables currently set.  You can access environment variables using `echo $<env_var_name>` or if you don't know the
exact name of the environment variable by piping the output of env to grep (or ack, or ag), for example:

``` sql Piping ENV output to grep
$ ENV | grep ssh
SSH_AUTH_SOCK=/private/tmp/com.apple.launchd.okWDAcFz3u/Listeners
```

The Rails framework also relies on environment variables, typically to set TEST, DEVELOPMENT or PRODUCTION modes.
These variables are stored in a hash named `ENV`, and they can be explored from the Rails console:

``` ruby Fetching environment values from the ENV hash
irb(main):001:0 ENV['RAILS_ENV']
=> "development"
```

If you've ever found that a passing test suite suddenly is failing for no reason, it could be because the rails environment
variable is not running as "test", but as "development".  The takeaway here is: Environment Variables Configure Behavior

They also configure other applications, such as the secret token to set Rails sessions, and the keys used to access
cloud services like AWS, or email services like Send Grid.  Often we, unknowingly set these environment variables in
files that are committed to Github, which is open source.

This has become such a problem that services such as AWS monitor repos looking to see if AWS credentials have been
compromised, and will send a warning notice to the credential owners without minutes of the git commit that exposed them.
A cursory search of Stack Overflow will reveal sad tales of credentials that were scraped from Github and then used to run
up very expensive bills for services that the credential owner was liable for.

## But what about 'secrets.yml' in Rails?

The `secrets.yml` file in Rails 4.1 was added to address this issue, bu the problem is, it's not really secret, and it's
a bit convoluted to use.  It violates these two basic rules:

1. Don't store environment variables in any file that is committed to a repo.
2. Manage your `.gitignore`, and make sure your team excludes files that contain credentials.

## The Solution?

Like most things in the Ruby world, if it's a common problem there is probably a `gem` for it.

In this care there are two: `dotenv` & `figaro`.

They both approach the problem in similar ways, I'll give the basics for using `dotenv`

### Step 1: Gemfile

``` ruby Add dotenv gem to the Gemfile
source 'https://rubygems.org'

ruby '2.2.0'
gem 'rails', '4.2.0'

gem 'dotenv-rails', groups [:development, :test]
```

Then run `bundle install`

### Step 2: Update your `.gitignore`

``` ruby Add the .env file to .gitignore
log/*
.bundle/

.env

```

This is important, you don't want to create the `.env` file before it is ignored because it's
extra steps to remove it if it gets accidentally committed.

### Step 3: Create the `.env` file

The `.env` file is a hidden file that will hold your environment variables.

``` ruby Example .env file
SECRET_TOKEN = '2398sljae23lkj;2'
S3_BUCKET_NAME = 'portal-device'
AWS_ACCESS_KEY_ID = '052kjJKW620'
SENDGRID_API = 'Twer2590lkQ5'
```

### Step 4: Configure secrets.yml and your initializers

Your Rails app needs to know how to get to these ENV variables, so you'll pass them in
from either secrets.yml or the initializers for your service, e.g.

``` ruby secrets.yml
development:
secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>
omniauth_provider_key: <%= ENV["SENDGRID_API"] %>
omniauth_provider_secret: <%= ENV["AWS_ACCESS_KEY_ID"] %>

test:
secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>
send_gred_key: <%= ENV["SENDGRID_API"] %>
aws_access_key: <%= ENV["AWS_ACCESS_KEY_ID"] %>

production:
secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>
send_gred_key: <%= ENV["SENDGRID_API"] %>
aws_access_key: <%= ENV["AWS_ACCESS_KEY_ID"] %>
```

### Step 5: Set the variables in production

Make sure to set your environment variables in your production VPS, either from the
web interface or via the command line.  For heroku this would be look like this:

`heroku config:set SECRET_TOKEN=45FEW%dak27ks`

### Step 6: Distribute to your development team

Make the `.env` file available to your collaborators, but do it via a secure means.  That
means don't use Dropbox, Slack, HipChat, etc.  Also make sure that everyone has pulled the
git commit that updated `.gitignore` to ignore the the `.env` file and use the `dotenv` gem.

### Resources

[Rails Guides: Configuring Rails Applications](guides.rubyonrails.org/configuring.html)

[Rails App Project: Rails Environment Variables](railsapps.github.io/rails-environment-variables.html)

[dotenv gem](github.com/bkeepers/dotenv)

[figaro gem](github.com/laserlemon/figaro)

[Brandon Hilkert Blog: Using Rails 4.1 Secrets for Configuration](brandonhilkert.com/blog/using-rails-4-dot-1-secrets-for-configuration)


