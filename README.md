# Guideline to deploy Rails app on cloud server

## Aws
### Pre-requisite on your local machine 
1. ruby 2.2.3 or 2.2.1 and rails 4. (using rvm) installed
2. Git installed
3. Nodejs installed
4. bundler installed
5. AWS EC2 instance created
6. Puma configured (Follow the steps from the file Deploy on Heroku Rails+ Postgresql with Puma.)

### Configuring Puma & Capistrano
Include thesse in GEM file
```
gem 'figaro' # for env variables 
gem 'puma'
group :development do
  gem 'capistrano'
  gem 'capistrano3-puma'
  gem 'capistrano-rails', require: false
  gem 'capistrano-bundler', require: false
  gem 'capistrano-rvm'
end
```
Run 
```
cap install STAGES=production
```
### Edit deploy.rb in config/deploy.rb
```
lock '3.5.0'
set :application, 'contactbook'
set :repo_url, 'git@github.com:devdatta/contactbook.git' # Edit this to match your repository
set :ssh_options, { forward_agent: true }
set :branch, :master
set :deploy_to, '/home/deploy/contactbook'
set :pty, true
set :linked_files, %w{config/database.yml config/application.yml}
set :linked_dirs, %w{bin log tmp/pids tmp/cache tmp/sockets vendor/bundle public/system public/uploads public/assets}
set :keep_releases, 5
set :rvm_type, :user
set :rvm_ruby_version, '2.2.3' # Edit this if you are using MRI Ruby
set :puma_rackup, -> { File.join(current_path, 'config.ru') }
set :puma_state, "#{shared_path}/tmp/pids/puma.state"
set :puma_pid, "#{shared_path}/tmp/pids/puma.pid"
set :puma_bind, "unix://#{shared_path}/tmp/sockets/puma.sock"    #accept array for multi-bind
set :puma_conf, "#{shared_path}/puma.rb"
set :puma_access_log, "#{shared_path}/log/puma_error.log"
set :puma_error_log, "#{shared_path}/log/puma_access.log"
set :puma_role, :app
set :puma_env, fetch(:rack_env, fetch(:rails_env, 'production'))
set :puma_threads, [0, 8]
set :puma_workers, 0
set :puma_worker_timeout, nil
set :puma_init_active_record, true
set :puma_preload_app, false

#In config/deploy.rb under namespace :deploy do line add this for assets precompile in capistrano 

desc 'run rake assets precompile task'
  task :assets_precompile do
    on roles(:app) do
      within "#{release_path}" do
        with rails_env: "#{fetch(:rails_env)}"  do
          execute :rake, 'assets:precompile'
        end
      end
    end
  end
end
 ```

### Login to your server, create a user for the app
```
ssh -i your_ec2_key.pem adminuser@yourserver.com
```
Replace adminuser with the name of an account with administrator privileges or sudo privileges. This is usually admin, ec2-user, root or ubuntu.

Now that you have logged in, you should create an operating system user account for your app. For security reasons, it is a good idea to run each app under its own user account, in order to limit the damage that security vulnerabilities in the app can do.

You should give the user account the same name as your app. But for demonstration purposes, this tutorial names the user account myappuser.
```
$ sudo adduser myappuser
```
We also ensure that that user has your SSH key installed:
```
sudo mkdir -p ~myappuser/.ssh
touch $HOME/.ssh/authorized_keys
sudo sh -c "cat $HOME/.ssh/authorized_keys >> ~myappuser/.ssh/authorized_keys"
sudo chown -R myappuser: ~myappuser/.ssh
sudo chmod 700 ~myappuser/.ssh
sudo sh -c "chmod 600 ~myappuser/.ssh/*"
```
### Install Git on the server

### Let EC-2 instance able to access github
1. login to the myappuser that was just created
```
$su - myappuser
```
2. gen ssh key by typing 
```
$ssh-keygen
```
3. copy key & paste to your github
```
$ cat .ssh/id_rsa.pub
```

### Set authorized_keys
(Capistrano will connect to the EC2 instance via ssh for deployment)
```
$ cat ~/.ssh/id_rsa.pub(@local)
```
copy key & paste to Ec2 instance
```
$ nano .ssh/authorized_keys
```
### Create some file that Capistrano deploy need
```
$ mkdir <app-name>
$ mkdir -p <app-name>/shared/config
$ nano <app-name>/shared/config/database.yml
```

### Inside <app-name>/shared/config/database.yml 
```
production:
  adapter: postgresql
  encoding: unicode
  database: contactbook_production       #edit it to fit your app
  username: deploy                       #edit it to fit your app name on remote postgresql
  password: 123456                       #edit it to fit your app password on remote postgresql
  host: localhost
  port: 5432
```
### Create application.yml
```
$ nano contactbook/shared/config/application.yml
```
application.yml
SECRET_KEY_BASE: created by your local(use rake secret)
Add all the secret keys here such as api_keys etc. if you are using figaro gem.

## Heroku
### Add these to GEM file
```
group :production do
  gem 'pg', '~> 0.15'
  gem 'rails_12factor', '~> 0.0.3'
  gem 'puma', '~> 2.16'
end
```
### Start Heroku
```
$ heroku create <app-name>
```
### Verify remote 
$ git remote -v

### Deploy 
```
$ git push heroku master
```
### Migrate your database on server
```
$ heroku run rake db:migrate
```
### Add Procfile to use puma webserver 
```
$ echo > Procfile
```
### Procfile content 
```
$ bundle exec puma -t 5:5 -p ${PORT:-3000} -e ${RACK_ENV:-development}
```
### In the command shell run  
```
$ echo "RACK_ENV=development" >>.env
$ echo "PORT=3000" >> .env
```
### In the command shell run 
```
$ echo ".env" >> .gitignore
$ git add .gitignore
$ git commit -m "add .env to .gitignore"
```
### Add Figaro Gem env configuration
```
$ figaro heroku:set -e production
```
### Compile assets in production
```
$ RAILS_ENV=production rake assets:precompile
```

### Run the following 
```
$ git add -A
$ git commit -m "use puma via procfile"
$ git push heroku master
```


