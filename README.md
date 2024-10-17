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

Ansible is used because:

1. it's idempotent, making it easier to modify the server setup and safe to run multiple times.
2. it's vendor-agnostic, so we can use it to provision any kind of cloud servers.

## Usage

1. Install ansible on your local machine

```bash
brew install ansible
```

2. Configure the `inventory.ini` files

Example:
```bash
cp inventory.ini.example l1.ini
cp inventory.ini.example l2.ini
```

replace the placeholder with the actual server information. Note that the server needs to be reachable from your local machine via ssh. On GCP, we can use the [project metadata](https://cloud.google.com/compute/docs/connect/add-ssh-keys#add_ssh_keys_to_project_metadata) to add our public SSH keys to to access all VMs in a project.

3. Configure SSH

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

Make sure all team members' public ssh keys are added to the playbook. Check the "Set up authorized keys for shared user" section in the playbook.

This may require registering your public SSH keys on cloud console (e.g. under `VM -> Metadata -> SSH Keys` for GCP).

4. Start L1

```bash
ansible-playbook -i l1.ini debian_op_babylon_devnet_l1.yml
```

once it's up you can test with:
```
cast block latest --rpc-url http://<l1-server-ip>:18545
```

Then ssh into the L1 server and find:
- the L1 chain ID in `~/op-chain-deployment/configs/l1/network_params.yaml`
- the pre-funded account private key in `~/op-chain-deployment/configs/l1/l1-prefund-wallet.json`

which you will need for the L2 playbook.

5. Start L2

```bash
ansible-playbook -i l2.ini debian_op_babylon_devnet_l2.yml
```

once it's up, ssh into the server with `ssh <l2-server-hostname>` and:

- `cd /home/snapchain/op-chain-deployment`
- run `sudo chown -R snapchain:snapchain /home/snapchain/op-chain-deployment`
- modify `.env` to set L1 URLs, chain ID and the pre-funded account priv key
```
L1_RPC_URL=http://<l1-server-ip>:18545
L1_BEACON_URL=http://<l1-server-ip>:15052
L1_CHAIN_ID=<l1-chain-id>
L1_FUNDED_PRIVATE_KEY=<l1-pre-funded-account-private-key>
```
- modify `.env.explorer` to set L2 RPC URL and common host
```
L2_RPC_URL=http://<l2-server-ip>:9545
COMMON_HOST=<l2-server-ip>
```

then run:
```
make l2-launch
```

after it's up, you can test with:
```
make verify-op-devnet # on the L2 server
cast block latest --rpc-url http://<l2-server-ip>:9545 # from anywhere
```

you can also access the bridge UI at http://<l2-server-ip>:3002/

## Notes

The playbook now only supports debian servers (default on GCP)
