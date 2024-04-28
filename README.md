# Nix Lima 

A Lima VM which serves as remote docker host and a nix linux builder. This is especially useful if you're looking to build OCI images using Nix rather than Dockerfiles and run containers on the VM using docker CLI rather than using Docker desktop VM for running containers and a linux builder VM for builds. 


## Setup

Assumptions-
* You're on a Mac
* You have Nix installed with flakes enabled. If you don't, install [it](https://github.com/DeterminateSystems/nix-installer). 

### Install nix-darwin
We need nix-darwin to configure our system to pick up the linux VM as a builder easily. 
You can follow [this guide](https://github.com/LnL7/nix-darwin?tab=readme-ov-file#flakes) to setup nix-darwin. 

### Install Lima

You can install it using an ad-hoc command like this. 
```
nix profile install nixpkgs#lima
```
Or add it in your nix-darwin flake.nix(make sure to `darwin-rebuild switch --flake .`):
```
environment.systemPackages =
        [ pkgs.lima
        ];
```

### Start the VM

This can take a minute or two. 
```
limactl delete -f default && limactl start --name=default x86.yaml
```

### Connect Docker CLI to VM

In order for the socket to be accessible, we need to give it permissions. 
```
lima sudo chmod 666 /var/run/docker.sock
```

Connect to it:
```
export DOCKER_HOST=$(limactl list default --format 'unix://{{.Dir}}/sock/docker.sock')
```

You can now check by:
```
docker ps
```

```
docker run hello-world
```


### Use VM as Nix Linux builder

In order to use a VM running on non-default port like lima VM, this was the only way I could find as of now(28th April 2024). 

#### Setup SSH 

```
sudo mkdir -p ~root/.ssh
```

```
sudo vi ~root/.ssh/config
```

and paste the contents(make sure to change the username accordingly):
```
Host lima-default
StrictHostKeyChecking no
User username
HostName 127.0.0.1
Port 60022
IdentityFile /Users/username/.lima/_config/user
IdentityFile /Users/username/.ssh/id_rsa
```

#### Modify darwin flake

In your darwin system flake, add this section(make sure to change the username accordingly):
```
    nix.distributedBuilds = true;
    nix.buildMachines = [{
     hostName = "lima-default";
     sshUser = "username";
     protocol = "ssh-ng";
     sshKey = "/Users/username/.lima/_config/user";
     systems = [ "x86_64-linux" ];
     maxJobs = 2;
     speedFactor = 2;
     supportedFeatures = [ "kvm" ];
     mandatoryFeatures = [ ];
```

```
darwin-rebuild switch --flake .
```

#### Add user to trusted-users
Final step involves adding your user to trusted-users on the VM. 

* SSH into the VM:
```
lima
```
* `cd /etc/nixos/`
* `sudo vi configuration.nix`

Add following contents:
```
nix.settings.trusted-users = ["root" "username"];
```

Rebuild:
```
nixos-rebuild switch
```


#### Test!
From your local machine, you can now run this:
```
nix-build -I nixpkgs=/Users/harshthakur/Desktop/buildsafe/nix  -E 'with import <nixpkgs> { system = "x86_64-linux"; }; hello.overrideAttrs (drv: { REBUILD = builtins.currentTime; })'
```


## Future
* Install [Nix-snapshotter](https://github.com/pdtpartners/nix-snapshotter?tab=readme-ov-file)
* Investigate cross compilation stability for nix builds for same OS, different arch. If works, maybe we can use vz instead of qemu.
* Invesigate networking to enable port-forwarding from host for accessing containers
* Package it similar to [Finch](https://github.com/runfinch/finch)


## Credits
[patryk4815](https://github.com/patryk4815/ctftools/tree/master/lima-vm) for initial Nix lima template.


All contributions are welcome! Feel free to create an issue or pull request or fork.

