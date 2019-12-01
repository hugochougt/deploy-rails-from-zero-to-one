# Config Capistrano

参考：[Deploying Rails app using Nginx, Puma and Capistrano 3](https://coderwall.com/p/ttrhow/deploying-rails-app-using-nginx-puma-and-capistrano-3)

进入 Rails app 所在的目录，先从 `master` 分支 `git checkout -b config-capistrano` ，再开始配置 capistrano 脚本。

## Install gems

在 `Gemfile` 文件增加 cap 部署相关的 gem，并且把 `puma` 版本升级到 4，因为 `capistrano3-puma` 要求 `puma` 版本至少是 4.0.0。

```ruby
# ...
gem 'puma', '~> 4.2.1'
# ...

group :development do
  # Remote multi-server automation tool
  # https://github.com/capistrano/capistrano
  gem "capistrano", "~> 3.11.1", require: false

  # RVM support for Capistrano v3
  # https://github.com/capistrano/rvm
  gem "capistrano-rvm", "~> 0.1.2", require: false

  # Rails specific Capistrano tasks
  # https://github.com/capistrano/rails
  gem "capistrano-rails", "~> 1.4.0", require: false

  # Bundler support for Capistrano 3.x
  # https://github.com/capistrano/bundler
  gem "capistrano-bundler", "~> 1.6.0", require: false

  # Puma integration for Capistrano 3
  # https://github.com/seuros/capistrano-puma
  gem "capistrano3-puma", "~> 4.0.0", require: false

  # Remote rails console for capistrano
  # https://github.com/ydkn/capistrano-rails-console
  gem "capistrano-rails-console", "~> 2.3.0", require: false

  # A collection of capistrano tasks for syncing assets and databases
  # https://github.com/sgruhier/capistrano-db-tasks
  gem "capistrano-db-tasks", "~> 0.6", require: false

  # Run any rake task on a remote server using Capistrano
  # https://github.com/sheharyarn/capistrano-rake
  gem "capistrano-rake", "~> 0.2.0", require: false
end
```

每个 gem 的用途，自行访问对应的 GitHub 仓库查阅。

```
$ bundle install
$ cap install
```

## 配置 Capfile

`cap install` 命令会执行以下操作：

```
mkdir -p config/deploy
create config/deploy.rb
create config/deploy/staging.rb
create config/deploy/production.rb
mkdir -p lib/capistrano/tasks
create Capfile
Capified
```

编辑 `Capfile`，将其内容替换为以下内容。

```ruby
# frozen_string_literal: true
# Load DSL and set up stages
require "capistrano/setup"

# Include default deployment tasks
require "capistrano/deploy"

# Load the SCM plugin appropriate to your project:
require "capistrano/scm/git"
install_plugin Capistrano::SCM::Git

# Include tasks from other gems included in your Gemfile
require "capistrano/rvm"
require "capistrano/bundler"
require "capistrano/rails"
require "capistrano/rails/console"
require "capistrano/rake"

require "capistrano/puma"
install_plugin Capistrano::Puma, load_hooks: false  # Default puma tasks
install_plugin Capistrano::Puma::Workers, load_hooks: false
set :puma_init_active_record, true

# Add database AND assets tasks to capistrano to a Rails project
# Read more: https://github.com/sgruhier/capistrano-db-tasks#capistranodbtasks
require "capistrano-db-tasks"
# if you want to remove the local dump file after loading
set :db_local_clean, true
# if you want to remove the dump file from the server after downloading
set :db_remote_clean, true

# if you are highly paranoid and want to prevent any push operation to the server
set :disallow_pushing, true

# Load custom tasks from `lib/capistrano/tasks` if you have any defined
Dir.glob("lib/capistrano/tasks/*.rake").each { |r| import r }
```

编辑 `config/deploy.rb` 文件，修改为一下内容。根据你的实际 repo 地址，修改 `repo_url` 的值。

```ruby
# config valid for current version and patch releases of Capistrano
lock "~> 3.11.2"

set :application, "rails-deployment-demo"
set :repo_url, "git@github.com:zhaqiang/rails-deployment-demo.git"
set :deploy_to, "/var/www/rails-deployment-demo"
set :init_system, :systemd

# Default value for :linked_files is []
append :linked_files, "config/database.yml", "config/master.key"

# Default value for linked_dirs is []
append :linked_dirs, "log", "tmp/pids", "tmp/cache", "tmp/sockets", "public/system"

namespace :puma do
  desc "Create Directories for Puma Pids and Socket"
  task :make_dirs do
    on roles(:app) do
      execute "mkdir #{shared_path}/tmp/sockets -p"
      execute "mkdir #{shared_path}/tmp/pids -p"
    end
  end

  before :start, :make_dirs
end

namespace :deploy do
  desc "Initialize configuration using example files provided in the distribution"
  task :upload_config do
    on release_roles :all do |host|
      Dir["config/master.key", "config/*.yml.example"].each do |file|
        save_to = "#{shared_path}/config/#{File.basename(file, '.example')}"
        unless test "[ -f #{save_to} ]"
          upload!(File.expand_path(file), save_to)
        end
      end
    end
  end
  before "deploy:check:linked_files", "deploy:upload_config"

  desc "Initial Deploy"
  task :initial do
    on roles(:app) do
      before "deploy:restart", "puma:start"
      invoke "deploy"
    end
  end

  desc "Restart application"
  task :restart do
    on roles(:app), in: :sequence, wait: 5 do
      invoke "puma:restart"
    end
  end

  after :finishing, :cleanup
  after :finishing, :restart
end
```

## 配置 deploy files

修改 `config/deploy/production.rb` 文件，根据实际情况替换 `app_url` 和 IP 地址为你的服务器的值。

```ruby
# frozen_string_literal: true

set :branch, "master"
set :rails_env, "production"
set :app_url, "https://deploy-demo.zq-dev.com"
set :rvm_ruby_version, "2.6.5"
set :rvm_custom_path, "/usr/share/rvm"

set :puma_bind, "unix://#{shared_path}/tmp/sockets/puma.sock"
set :puma_state, "#{shared_path}/tmp/pids/puma.state"
set :puma_pid, "#{shared_path}/tmp/pids/puma.pid"
set :puma_access_log, "#{shared_path}/log/puma.error.log"
set :puma_error_log, "#{shared_path}/log/puma.access.log"

server "your-server-ip", user: "deploy", roles: %w[app db web]
```

修改完毕后，`git add .` 然后 `git commit -m "Config capistrano"` ，提交本次修改内容。再 `git push origin config-capistran` ，把改动推送到 GitHub 。

## 确保服务器的数据库已经创建

参考：[Creating user, database and adding access on PostgreSQL](https://medium.com/coding-blocks/creating-user-database-and-adding-access-on-postgresql-8bfcd2f4a91e)

如果你是跟着我的教程执行，那你的数据库肯定是没创建的，在部署前需要先手动创建。

ssh 登录服务器，执行以下命令。

```
$ sudo -u postgres psql
postgres=# create database rails_deployment_demo_production;
postgres=# create user deploy with encrypted password 'your-password';
postgres=# grant all privileges on database rails_deployment_demo_production to deploy;
postgres=# \q
```

检查本地的 `config/database.yml`，确保 `production` 块的数据库连接配置在跟服务器上的匹配。

```yaml
production:
  <<: *default
  database: rails_deployment_demo_production
  username: deploy
  password: <%= ENV['RAILS_DEPLOYMENT_DEMO_DATABASE_PASSWORD'] %>
```

登录服务器，编辑 `~/.profile` ，增加环境变量 `RAILS_DEPLOYMENT_DEMO_DATABASE_PASSWORD`。

```bash
export RAILS_DEPLOYMENT_DEMO_DATABASE_PASSWORD='your-password'
```

进入 `/var/www/` 目录，创建 `rails-deployment-demo` 目录并将其权限修改为 deploy 用户。

```bash
cd /var/www
sudo mkdir rails-deployment-demo
sudo chown deploy:deploy rails-deployment-demo/
```

## 初始化部署

返回到 rails app 根目录，执行 `cap production deploy:initial`。如果提示没有读取仓库的权限，则到服务器根据 [Generating a new SSH key and adding it to the ssh-agent](https://help.github.com/en/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) 创建一对 ssh 密钥，并且把 `id_rsa.pub` 文件的内容添加到 GitHub 的 ssh keys 设置里。

如无意外的话，控制台应该输出类似以下成功初始化部署的信息。

```
✔ 01 deploy@your-server-ip 2.144s
```
