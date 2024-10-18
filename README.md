# devops

This repo contains the ansible playbooks for setting up all our remote servers.

`debian_server_setup_playbook.yml`
- installs build-essential, ca-certificates, curl, git, zsh, acl
- sets up the server with a shared user `snapchain`
- sets up authorized keys for the shared user
- installs oh-my-zsh
- installs and configures powerlevel10k theme
- installs and sets up docker

`debian_op_babylon_devnet_l1.yml`
- runs the base server setup playbook `debian_server_setup_playbook.yml`
- installs foundry, jq, yq
- installs Kurtosis
- clones the Snapchain/op-chain-deployment repo
- starts a private L1 network with Kurtosis

`debian_op_babylon_devnet_l2.yml`
- runs the base server setup playbook `debian_server_setup_playbook.yml`
- installs Go, Node.js, pnpm, just, foundry, jq, yq, nc
- clones the Snapchain/op-chain-deployment repo

`debian_op_devnet_babylon.yml`
- runs the base server setup playbook `debian_server_setup_playbook.yml`
- installs jq
- clones the babylonlabs-io/babylon-integration-deployment repo
- sets up ENV and a few config files

Ansible is used because:

1. it's idempotent, making it easier to modify the server setup and safe to run multiple times.
2. it's vendor-agnostic, so we can use it to provision any kind of cloud servers.

## Prerequisites

1. Install ansible on your local machine

```bash
brew install ansible
```

2. Register your public SSH key to GCP's project metadata

The server needs to be reachable from your local machine via ssh. On GCP, we can use the [project metadata](https://cloud.google.com/compute/docs/connect/add-ssh-keys#add_ssh_keys_to_project_metadata) (e.g. under `VM -> Metadata -> SSH Keys` for GCP) to add our public SSH keys to to access all VMs in a project.

3. Register your public SSH key to the base playbook

Make sure your' public ssh keys are also added to the debian_server_setup_playbook.yml playbook. Check the "Set up authorized keys for shared user" section in the playbook.

4. Reserve VM on GCP and configure SSH

You `~/.ssh/config` should include L1 and L2 like this:

```
Host <hostname>
    HostName <server-ip>
    User snapchain
    IdentityFile /path/to/private/ssh-key
    UserKnownHostsFile=/path/to/known_hosts
    IdentitiesOnly=yes
    CheckHostIP=no
    ForwardAgent=yes
```

## To start L1 and L2

1. Configure the inventory files

Example:
```bash
cp l1.ini.example l1.ini
cp l2.ini.example l2.ini
```

Under `[gcp_vm]`, replace the IP address with the L1 and L2 server IP addresses you reserved on GCP or other cloud provider. Replace `ansible_user` with your server username. Replace `ansible_ssh_private_key_file` with the **local** path to your ssh key.

For `l2.ini`, under `[gcp_vm:vars]`, fill in the server IPs. The desired L1 chain ID and pre-funded account private key are retrieved from the L1 server (see steps below).

2. Start L1

```bash
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i l1.ini debian_op_babylon_devnet_l1.yml
```

Once it's up you can test with:
```
cast block latest --rpc-url http://<l1-server-ip>:18545
```

Then ssh into the L1 server and find:
- the L1 chain ID in `~/op-chain-deployment/configs/l1/network_params.yaml`
- the pre-funded account private key in `~/op-chain-deployment/configs/l1/l1-prefund-wallet.json`

Now modify `l2.ini`'s `gcp_vm:vars` section with the L1 chain ID and the pre-funded account private key.

3. Start L2

```bash
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i l2.ini debian_op_babylon_devnet_l2.yml
```

once it's done, ssh into the server with `ssh <l2-server-hostname>` and:

- run `cd /home/snapchain/op-chain-deployment && sudo chown -R snapchain:snapchain /home/snapchain/op-chain-deployment && make l2-launch`

after it's up, you can test with:
```
make verify-op-devnet # on the L2 server
cast block latest --rpc-url http://<l2-server-ip>:9545 # from anywhere
```

you can also access the bridge UI at `http://<l2-server-ip>:3002/`

## To start Babylon devnet and finality gadget

1. Configure the inventory file

Example:
```bash
cp babylon.ini.example babylon.ini
```

replace the IP address with the one you reserved on GCP. add other required variables.

2. Start Babylon devnet and finality gadget
```bash
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i babylon.ini debian_op_devnet_babylon.yml
```

once it's done, ssh into the server with `ssh <server-hostname>` and run:

```bash
sudo chown snapchain:snapchain -R ~/babylon && cd ~/babylon/deployments/finality-gadget-integration-op-l2  && make start-babylon
```

## Notes

The playbook now only supports debian servers (default on GCP)

## Troubleshooting

1. If you ssh into the server before the playbook finishes running, you might see errors like:

```
cast: command not found
```

This is because the `PATH` environment variable is not updated when the playbook finishes running.

To fix, run `omz reload` on the server once ansible finishes running.
