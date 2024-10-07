# devops

This repo contains the ansible playbooks for setting up all our remote servers. It:

- sets up the server with a shared user `snapchain`
- installs oh-my-zsh
- installs docker and git

Ansible is used because:

1. it's idempotent, making it easier to modify the server setup and safe to run multiple times.
2. it's vendor-agnostic, so we can use it to provision any kind of cloud servers.

## Usage

1. Install ansible on your local machine

```bash
brew install ansible
```

2. Configure the `inventory.ini` file

```bash
cp inventory.ini.example inventory.ini
```

replace the placeholder with the actual server information. Note that the server needs to be reachable from your local machine via ssh. On GCP, we can use the [project metadata](https://cloud.google.com/compute/docs/connect/add-ssh-keys#add_ssh_keys_to_project_metadata) toadd our public SSH keys to to access all VMs in a project.

3. Make sure all team members' public ssh keys are added to the playbook

check the "Set up authorized keys for shared user" section in the playbook

4. Run the playbook

```bash
ansible-playbook -i inventory.ini debian_server_setup_playbook.yml
```

5. Test the setup

```bash
ssh -i <path-to-private-ssh-key> snapchain@<server-ip>
```

You can also config your `~/.ssh/config` to directly ssh into the server by adding a `Host <hostname>` block.

```
Host my-server
    HostName <server-ip>
    User snapchain
    IdentityFile /path/to/private/ssh-key
    UserKnownHostsFile=/path/to/known_hosts
    IdentitiesOnly=yes
    CheckHostIP=no
    ForwardAgent=yes
```

and then ssh into the server with `ssh my-server`

## Notes

The playbook now only supports debian servers (default on GCP)