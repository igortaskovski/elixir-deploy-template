# deploy-template

This is a turn-key example app which shows how to deploy Elixir app based on
this [best practices for deploying elixir
apps](https://www.cogini.com/blog/best-practices-for-deploying-elixir-apps/)
blog post.

It's regularly tested deploying to [Digital Ocean](https://m.do.co/c/65a8c175b9bf),
with CentOS 7, Ubuntu 16.04, Ubuntu 18.04 and Debian 9.4. It assumes a distro
that supports systemd.

The instructions here go through things step by step. You should be able to
copy and paste and it will work, with some minor changes to configure it for
your server.

After it's set up, you can deploy a release by logging
into your server and running:

```shell
cd ~/build/elixir-deploy-template
git pull

# Install specified versions of Erlang/Elixir/Node.js
asdf install

# Build release
mix deps.get --only prod
MIX_ENV=prod mix do compile, phx.digest, release

# Deploy locally
MIX_ENV=prod mix deploy.local
sudo /bin/systemctl restart deploy-template

# Deploy to remote servers
ansible-playbook -u deploy -v -l web-servers playbooks/deploy-template.yml --tags deploy --extra-vars ansible_become=false -D
```

# Installation

Check out the code from git to your local dev machine (make a fork, perhaps):

```shell
git clone https://github.com/cogini/elixir-deploy-template
```

## Set up ASDF

Install ASDF as described in [the ASDF docs](https://github.com/asdf-vm/asdf).

Install plugins for our tools:

```shell
asdf plugin-add erlang
asdf plugin-add elixir
asdf plugin-add nodejs
```

Install Erlang, Elixir and Node.js:

```shell
asdf install
```
Run this multiple times until everything is installed (should be twice).

Install libraries into the ASDF Elixir dirs:

```shell
mix local.hex --force
mix local.rebar --force
mix archive.install https://github.com/phoenixframework/archives/raw/master/phx_new.ez --force
```

## Initialize the app

```shell
mix deps.get
mix deps.compile
```

At this point you should be able to run the app locally with:

```shell
mix compile
iex -S mix phx.server
open http://localhost:4000/
```

## Deploy the app

Install Ansible on your dev machine:

```shell
pip install ansible
```

See [the Ansible docs](http://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
for other options.

### Set up a target machine

An easy option is [Digital Ocean](https://m.do.co/c/65a8c175b9bf).
Their smallest $5/month Droplet will run Phoenix fine.

Add the host to the `~/.ssh/config` on your dev machine:

    Host elixir-deploy-template
        HostName 123.45.67.89

Add the host to the groups in the Ansible inventory `ansible/inventory/hosts` file:

    [web-servers]
    elixir-deploy-template

    [build-servers]
    elixir-deploy-template

The host name is not important, you can use an existing server. Just use the
ssh name in the `inventory/hosts` config and in the `ansible-playbook` commands
below.

### Configure the target server using Ansible

From the `ansible` dir:

Newer versions of Ubuntu (16.04+) ship with Python 3, but the default for
Ansible is Python 2. If you are running Ubuntu, install Python 2:

```shell
ansible-playbook -u root -v -l elixir-deploy-template playbooks/setup-python.yml -D
```

In this command, `elixir-deploy-template` is the hostname.

Edit the `playbooks/manage-users.yml` script to specify user accounts:

```yaml
  vars:
    users_app_user: foo
    users_app_group: foo
    users_deploy_user: deploy
    users_deploy_group: deploy
    users_users:
      - user: jake
        name: "Jake Morrison"
        github: reachfh
    users_admin_users:
      - jake
    users_app_users:
      - jake
    users_deploy_users:
      - jake
    users_deploy_groups:
      - foo
```

* `foo` is the OS account which is used used to run the app.
* `deploy` is the OS account which is used to deploy the app.
* `jake` is a live user (me!), change it to match your account.
* `users_users` defines the list of users.
* `users_admin_users` defines OS admin users who can sudo.
* `users_app_users` defines users who can login to the app (foo) account via ssh.
* `users_deploy_users` defines users who can login to the app (foo) account via ssh.
* `users_deploy_groups` defines secondary groups that the deploy user should
  have, e.g. to share access.

See [the documentation for the role](https://galaxy.ansible.com/cogini/users/)
for more details about options, e.g. defining user keys instead of relying on GitHub.

```shell
ansible-playbook -u root -v -l elixir-deploy-template playbooks/manage-users.yml -D
```
See comments in `playbooks/manage-users.yml` for other ways to do the initial bootstrap.

Do initial server setup (currently minimal):

```shell
ansible-playbook -u $USER -v -l web-servers playbooks/setup-web.yml -D
```

In this command, `web-servers` is the group of servers. Ansible allows you to work
on groups of servers simultaneously. Configuration tasks are written to be idempotent,
so we can run the playbook against all our servers and it will make whatever changes
are needed to get them up to date.

Set up the app (create app dirs, etc.):

```shell
ansible-playbook -u $USER -v -l web-servers playbooks/deploy-template.yml --skip-tags deploy -D
```

## Set up the build server

The build server can be the same as the web server.

Set up the server, mainly ASDF:

```shell
ansible-playbook -u $USER -v -l build-servers playbooks/setup-build.yml -D
```

## Build the app

Log into the `deploy` user on the build machine:

```shell
ssh -A deploy@elixir-deploy-template
```

The `-A` flag on the ssh command gives the session on the server access to your
local ssh keys. If your local user can access a GitHub repo, then the server
can do it, without having to put keys on the server. Similarly, you can deploy
code to a prod server using Ansible without the web server trusting the build server.

If you are using a CI server to build and deploy code, then you might
create a deploy key in GitHub so it can access to your source and add the ssh key
to the `deploy` user account on the prod servers so the CI server can push releases.

Check out the source:

```shell
mkdir build
cd build
git clone https://github.com/cogini/elixir-deploy-template
cd elixir-deploy-template
```

Install Erlang, Elixir and Node.js as specified in `.tool-versions`:

```shell
asdf install
```
Run this multiple times until everything is installed (should be twice).

The initial build of Erlang from source can take a while, so you may
want to run it under `tmux` or `screen`.

Install libraries into the ASDF dir for the specified Elixir version:

```shell
mix local.hex --force
mix local.rebar --force
```

Generate a cookie and put it in `config/cookie.txt`:

```elixir
iex> :crypto.strong_rand_bytes(32) |> Base.encode16
```

Build the production release:

```shell
mix deps.get --only prod
MIX_ENV=prod mix do compile, phx.digest, release
```

Now you should be able to run the app from the release:

```shell
PORT=4001 _build/prod/rel/deploy_template/bin/deploy_template foreground
```

```shell
curl -v http://localhost:4001/
```

## Deploy the release

If you are running on the same machine, then you can use the custom
mix tasks in `lib/mix/tasks/deploy.ex` to deploy locally.

In `mix.exs`, set `deploy_dir` to match the directory structure in the
Ansible tasks, e.g.:

```elixir
deploy_dir: "/opt/myorg/deploy-template/",
```

Deploy the release:

```shell
MIX_ENV=prod mix deploy.local
sudo /bin/systemctl restart deploy-template
```

This assumes that the build is being done under the `deploy` user, who owns the
files under `/opt/myorg/deploy-template` and has a special `/etc/sudoers.d`
config which allows it to run the `/bin/systemctl restart deploy-template`
command.

You should be able to connect to the app supervised by systemd:
```shell
curl -v http://localhost:4001/
```

Have a look at the logs:
```shell
# systemctl status deploy-template
# journalctl -r -u deploy-template
```

You should also be able to access the machine over the network on port 80
through the magic of iptables port forwarding.

## Deploy to a remote machine using Ansible

From your dev machine, install Ansible on the build machine:

```shell
ansible-playbook -u $USER -v -l build-servers playbooks/setup-ansible.yml -D
```

Log into the build machine:

```shell
ssh -A deploy@elixir-deploy-template
cd ~/build/elixir-deploy-template/ansible
```

Add the `web-servers` hosts to the `~/.ssh/config` on the deploy machine:

    Host elixir-deploy-template
        HostName 123.45.67.89

We normally maintain the list of servers in a `ssh.config` file in the repo.
See `ansible/ansible.cfg` for options.

From deploy machine, deploy the app:

```shell
ansible-playbook -u deploy -v -l web-servers playbooks/deploy-template.yml --tags deploy --extra-vars ansible_become=false -D
```

# Changes

Following are the steps used to set up this repo.

It all began with a new Phoenix project:

```shell
mix phx.new --no-ecto deploy_template
```

## Set up distillery

Generate initial `rel` files:

```shell
mix release.init
```

Modify `rel/config.exs` to set the cookie from a file and update `vm.args.eex`
to tune the VM settings.

## Set up ASDF

Add `.tool-versions` file to specify versions of Elixir and Erlang.

## Configure for running in a release

Edit `config/prod.exs`

Uncomment this:

```elixir
config :phoenix, :serve_endpoints, true
```

Comment this, as we are not using `prod.secret.exs`:

```elixir
# import_config "prod.secret.exs"
```

## Add Ansible

Add tasks to set up the servers and deploy code, in the `ansible`
directory.

To make it easier to run, this repository contains local copies
of roles from Ansible Galaxy in `roles.galaxy`. To update them, run:

```shell
ansible-galaxy install --roles-path roles.galaxy -r install_roles.yml
```

## Add mix tasks for local deploy

Add `lib/mix/tasks/deploy.ex`

## Add shutdown_flag library

Add [shutdown_flag](https://github.com/cogini/shutdown_flag) to `mix.exs`:

    {:shutdown_flag, github: "cogini/shutdown_flag"},

Add to `config/prod.exs`:

```elixir
config :shutdown_flag,
  flag_file: "/var/tmp/deploy/deploy-template/shutdown.flag",
  check_delay: 10_000
```
