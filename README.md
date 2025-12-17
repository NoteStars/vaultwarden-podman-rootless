# vaultwarden-podman-rootless
For more information, I recommend visiting Dani's Garcia on how to officially run Vaultwarden using Podman: [Using Podman](https://github.com/dani-garcia/vaultwarden/wiki/Using-Podman)

## 1. Create a Non-Root User

`sudo useradd -m vaultwarden`

**NOTE:** This user will have no password, no SSH keys, and no sudo access. 

## 2. Configuring the Host

* **systemd**:
  * `sudo loginctl enable-linger vaultwarden`
  * If linger is not enabled, systemd will stop a user's processes when not logged in. 

* **DNS**:
  * We need a DNS package for rootless Podman containers to communicate to each other or else it won't work.
  * `sudo apt install aardvark-dns`
  
### Switching Users
---
We can access the user `vaultwarden` by using the systemd command `machinectl`.
This has serveal advantages over using `su` to switch users including communicating to the dbus which is not found in other commands. 

You can use the `machinectl` utility by installing the `systemd-container` on Ubuntu. 

**Command:** `sudo machinectl shell vaultwarden@`

## 3. Creating a Pod

All quadlet files are stored in `~/.config/containers/systemd` as rootless.

<details>
<summary>Beginners note for sudo in unpriviledged users:</summary>
Using `sudo mkdir -p ~/.config/containers/systemd`, we would get the following `vaultwarden is not in the sudoers file.`How would we even create new files or directies then? Please note since our users does not have sudo access, we could still write files in our own home directory. You would just simply omit `sudo`.
</details>

     3.1 Create the directory 
        - `mkdir -p ~/.config/containers/systemd`
     3.2 Create the quadlet file .pod
        - `nano ~/.config/containers/systemd/vaultwarden.pod` 
    
    
    [Pod]
    PodName=vaultwarden
    Network=vaultwarden.network
    PublishPort=8080:8080
    
- 3.3 Define the Pod Network
    - Create the file `~/.config/containers/systemd/vaultwarden.network`
    ```
    [Network]
    NetworkName=vaultwarden
    Gateway=192.168.220.1
    Subnet=192.168.220.0/24

    ```
        
    `Gateway=192.168.220.1` - Router IP inside the new Podman network. Containers will use this as their default gateway.
    `Subnet=192.168.220.0` - IP range assigned to the network. Containers get IPs like `192.168.220.X`.
    
    This will create a private virtual network isolated from the host LAN.
    Vaultwarden app container and DB container can talk to each other using container IPs, and not `localhost`.
    
- 3.4 Making volumes persistent
    - Create the file `~/.config/containers/systemd/vaultwarden-app.volume`
        ```
        [Volume]
        VolumeName=vaultwarden-app
        ```
    - Create the file `~/.config/containers/systemd/vaultwarden-db.volume`
        ```
        [Volume]
        VolumeName=vaultwarden-db
        ```
    
- 3.5 Setting the definition of containers
    - Create the file `~/.config/containers/systemd/vaultwarden-app.container`
      ```
      [Container]
      ContainerName=vaultwarden-app
      EnvironmentFile=/home/vaultwarden/vaultwarden/config
      
      HealthCmd=/healthcheck.sh
      HealthInterval=120s
      HealthRetries=10
      HealthTimeout=45s
      
      Image=quay.io/vaultwarden/server
      
      Pod=vaultwarden.pod
      
      Secret=database_url,type=env,target=DATABASE_URL
      Secret=admin_token,type=env,target=ADMIN_TOKEN
      
      Volume=vaultwarden-app.volume:/data
      
      [Unit]
      Requires=vaultwarden-db.service
      After=vaultwarden-db.service

      [Install]
      WantedBy=default.target
      ```
     
      These are settings basically for the vaultwarden-app but formatted and used by systemd.
      
      Make a new folder ~/vaultwarden/config
      
    - Create the file `~/.config/containers/systemd/vaultwarden-db.container`
      ```
      [Container]
      ContainerName=vaultwarden-db
      EnvironmentFile=/home/vaultwarden/vaultwarden/vaultwarden-db.env
      
      HealthCmd=/usr/bin/pg_isready -q -d vaultwarden -U vaultwarden
      HealthInterval=120s
      HealthRetries=10
      HealthTimeout=45s
      
      Image=quay.io/library/postgres
      Pod=vaultwarden.pod
      
      Secret=postgres_password,type=env,target=POSTGRES_PASSWORD
      Volume=vaultwarden-db.volume:/var/lib/postgresql/data

      [Install]
      WantedBy=default.target
      ```
      
# 4. Configuration 

Setup the folders:

```
mkdir -p ~/vaultwarden
mkdir -p ~/vaultwarden/data
```

`touch ~/.vaultwarden/vaultwarden-db.env`


To check if its generated use `systemctl --user list-unit-files | grep vaultwarden`

## Troubleshooting 
* `Failed to start vaultwarden-pod.service: Unit vaultwarden-pod.service not found.` 
    * If the service is not being generated, you can run `/usr/lib/systemd/system-generators/podman-system-generator --user --dryrun`

To reload systemd configuration,
`systemctl --user daemon-reload`
`systemctl --user restart vaultwarden-app.service`
## Sources
    - [YouTube: Michael Fox - Podman + Quadlet + Ansible: Rootless Service Management](https://www.youtube.com/watch?v=F0hhtDnTVwo)
    - [Documentation: SUSE - Rootless Podman](https://documentation.suse.com/smart/container/pdf/rootless-podman_en.pdf)
