
# Setting the new user
```
$ ssh root@DROPLET_IP
$ sudo adduser deployer
$ sudo usermod -a -G sudo deployer
$ su deployer
```


# Generating SSH key
[GitHub help page](https://help.github.com/articles/generating-ssh-keys/)  

```
$ ssh-keygen -t rsa -C "your_email@example.com"
$ eval "$(ssh-agent -s)"
$ ssh-add ~/.ssh/id_rsa
```


# Rbenv
```
$ sudo apt-get update
$ sudo apt-get install -y curl git-core build-essential zlib1g-dev libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libcurl4-openssl-dev libxml2-dev libxslt1-dev python-software-properties
```  

```
$ git clone https://github.com/sstephenson/rbenv.git ~/.rbenv
$ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
$ echo 'eval "$(rbenv init -)"' >> ~/.bashrc
```  

`$ exec $SHELL`

check if all is installed:  
`$ type rbenv`

you should see:
```
rbenv is a function
rbenv ()
{bla bla bla}
```


# Ruby
```
$ git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
$ rbenv install 2.1.2
$ rbenv global 2.1.2
```  

check ruby version:
`$ ruby -v`  
you should see:  
`ruby 2.1.2p95 (2014-05-08 revision 45877) [x86_64-linux]`


# Bundler and NodeJS
```
$ sudo apt-get install -y nodejs
$ gem install bundler
```


# PostgreSQL
```
$ sudo apt-get install -y postgresql postgresql-contrib libpq-dev
$ createuser --pwprompt
```  
then enter your password   
`$ sudo -u postgres psql`
`postgres=# \password postgres`  
enter your password  
```
postgres=# CREATE DATABASE APPNAME_production;
postgres=# \q
```


# Redis
```
$ sudo apt-get install -y redis-server
$ redis-server /etc/redis/redis.conf
```  

if all is okay `redis-cli` should work


# Nginx
```
$ sudo apt-get install -y nginx
$ sudo service nginx start
```

check if it's running:
`$ ps ax | grep nginx`
you can also open the browser and type in IP of the droplet you should see: **Welcome to nginx!**

### Nginx.conf
`$ sudo nano /etc/nginx/nginx.conf`

Here is example of `nginx.conf`:  
```
user www-data;
worker_processes 4;
pid /run/nginx.pid;

events {
  worker_connections 768;
}

http {
  ##
  # Basic Settings
  ##
  sendfile on;
  tcp_nopush on;
  tcp_nodelay on;
  keepalive_timeout 65;
  types_hash_max_size 2048;

  server_name_in_redirect off;

  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  ##
  # Logging Settings
  ##
  access_log /var/log/nginx/access.log;
  error_log /var/log/nginx/error.log;

  ##
  # Gzip Settings
  ##
  gzip on;
  gzip_disable "msie6";

  ##
  # Virtual Host Configs
  ##
  include /etc/nginx/conf.d/*.conf;
  include /etc/nginx/sites-enabled/*;
}
```

Then:  
`$ sudo nano /etc/nginx/sites-enabled/default`  
Example of the file:
```
upstream app {
    # Path to Unicorn SOCK file, as defined previously
    server unix:/home/deployer/YOUR_APP_NAME/shared/sockets/unicorn.sock fail_timeout=0;
}

server {
    listen 80 default_server;
    # Application root, as defined previously
    root /home/deployer/YOUR_APP_NAME/current/public;

    # uncomment and add your domain whenever you have one
    # server_name www.YOUR_APP_NAME.com YOUR_APP_NAME.com;

    try_files $uri/index.html $uri @app;

    access_log /var/log/nginx/YOUR_APP_NAME_access.log combined;
    error_log /var/log/nginx/YOUR_APP_NAME_error.log;

    location @app {
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_pass http://app;
        you're not running SSL
    }

    error_page 500 502 503 504 /500.html;
    client_max_body_size 4G;
    keepalive_timeout 10;
}
```


# Unicorn
This is inside your Rails application, create: `config/unicorn.rb`  
Example:  
```
# Set your full path to application.
app_dir = File.expand_path('../../', __FILE__)
shared_dir = File.expand_path('../../../shared/', __FILE__)

# Set unicorn options
worker_processes 2
preload_app true
timeout 30

# Fill path to your app
working_directory app_dir

# Set up socket location
listen "#{shared_dir}/sockets/unicorn.sock", :backlog => 64

# Loging
stderr_path "#{shared_dir}/log/unicorn.stderr.log"
stdout_path "#{shared_dir}/log/unicorn.stdout.log"

# Set master PID location
pid "#{shared_dir}/pids/unicorn.pid"

before_fork do |server, worker|
  defined?(ActiveRecord::Base) and ActiveRecord::Base.connection.disconnect!
  old_pid = "#{server.config[:pid]}.oldbin"
  if File.exists?(old_pid) && server.pid != old_pid
    begin
      sig = (worker.nr + 1) >= server.worker_processes ? :QUIT : :TTOU
      Process.kill(sig, File.read(old_pid).to_i)
    rescue Errno::ENOENT, Errno::ESRCH
      # someone else did our job for us
    end
  end
end

after_fork do |server, worker|
  defined?(ActiveRecord::Base) and ActiveRecord::Base.establish_connection
end

before_exec do |server|
  ENV['BUNDLE_GEMFILE'] = "#{app_dir}/Gemfile"
end
```


# [Mina](http://nadarei.co/mina/) (for deployment)  
add code below to your `Gemfile`:
```
gem 'mina'
gem 'mina-sidekiq', :require => false
gem 'mina-unicorn', :require => false


group :production do
  gem 'unicorn'
end
```

After that do:
`$ mina init`  
Created config/deploy.rb.

`config/deploy.rb` (example):
```
require 'mina/bundler'
require 'mina/rails'
require 'mina/git'
require 'mina/rbenv'
require 'mina_sidekiq/tasks'
require 'mina/unicorn'

# Basic settings:
#   domain       - The hostname to SSH to.
#   deploy_to    - Path to deploy into.
#   repository   - Git repo to clone from. (needed by mina/git)
#   branch       - Branch name to deploy. (needed by mina/git)

set :domain, 'YOUR DROPLETS IP'
set :deploy_to, '/home/deployer/YOUR_APP'
set :repository, 'YOUR_GIT_REPO_URL' # make sure to add previously generated ssh key to deploy keys on github and use ssh clone url from github
set :branch, 'master'
set :user, 'deployer'
set :forward_agent, true
set :port, '22'
set :unicorn_pid, "#{deploy_to}/shared/pids/unicorn.pid"

# Manually create these paths in shared/ (eg: shared/config/database.yml) in your server.
# They will be linked in the 'deploy:link_shared_paths' step.
set :shared_paths, ['config/database.yml', 'log', 'config/secrets.yml']


# This task is the environment that is loaded for most commands, such as
# `mina deploy` or `mina rake`.
task :environment do
  queue %{
echo "-----> Loading environment"
#{echo_cmd %[source ~/.bashrc]}
}
  invoke :'rbenv:load'
  # If you're using rbenv, use this to load the rbenv environment.
  # Be sure to commit your .rbenv-version to your repository.
end

# Put any custom mkdir's in here for when `mina setup` is ran.
# For Rails apps, we'll make some of the shared paths that are shared between
# all releases.
task :setup => :environment do
  queue! %[mkdir -p "#{deploy_to}/shared/log"]
  queue! %[chmod g+rx,u+rwx "#{deploy_to}/shared/log"]

  queue! %[mkdir -p "#{deploy_to}/shared/sockets"]

  queue! %[mkdir -p "#{deploy_to}/shared/config"]
  queue! %[chmod g+rx,u+rwx "#{deploy_to}/shared/config"]

  queue! %[touch "#{deploy_to}/shared/config/database.yml"]
  queue  %[echo "-----> Be sure to edit 'shared/config/database.yml'."]

  queue! %[touch "#{deploy_to}/shared/config/secrets.yml"]
  queue %[echo "-----> Be sure to edit 'shared/config/secrets.yml'."]

  # sidekiq needs a place to store its pid file and log file
  queue! %[mkdir -p "#{deploy_to}/shared/pids/"]
  queue! %[chmod g+rx,u+rwx "#{deploy_to}/shared/pids"]
end

desc "Deploys the current version to the server."
task :deploy => :environment do
  deploy do

    # stop accepting new workers
    invoke :'sidekiq:quiet'

    invoke :'git:clone'
    invoke :'deploy:link_shared_paths'
    invoke :'bundle:install'
    invoke :'rails:db_migrate'
    invoke :'rails:assets_precompile'

    to :launch do
      invoke :'sidekiq:restart'
      invoke :'unicorn:restart'
    end
  end
end
```

now you can do:  
```
$ mina setup
-----> Creating folders... done.
```


### Back to production - set up secrets.yml and database.yml
```
$ ssh deployer@DROPLET_IP
$ nano /home/deployer/YOUR_APP/shared/config/secrets.yml
```

Here is example of `secrets.yml`:
```
production:
  secret_key_base: 82d58d3dfb91238b495a311eb8539edf5064784f1d58994679db8363ec241c745bef0b446bfe44d66cbf91a2f4e497d8f6b1ef1656e3f405b0d263a9617ac75e
```
run `rake secret` to generate a key and paste above (on your local machine)  

then do:
`$ nano /home/deployer/YOUR_APP/shared/config/secrets.yml`  

example of `database.yml`:  
```
production:
  adapter: postgresql
  encoding: unicode
  database: APPNAME_production
  username: postgres
  password: password_set_in_the_very_top
  host: localhost
```


# Deployment
to add github to known_hosts I end up cloning my repo to production, accepting fingerprinting and removing the repo (end result 'handshake' between production and github repo). After that:  

`$ mina deploy`

happy hacking!





