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

replace the placeholder with the actual server information

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

## Notes

The playbook now only supports debian servers (default on GCP)