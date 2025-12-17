![Podman](https://raw.githubusercontent.com/containers/common/main/logos/podman-logo-full-vert.png)

# vaultwarden-podman-rootless
For more information, I recommend visiting Dani's Garcia on how to officially run Vaultwarden using Podman: [Using Podman](https://github.com/dani-garcia/vaultwarden/wiki/Using-Podman)

I created this guide just for beginners to go through, along with the issues that I came across.

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
<summary>Beginners note for sudo in unprivileged users:</summary>

* Using `sudo mkdir -p ~/.config/containers/systemd`, we would get the following:

     * `vaultwarden is not in the sudoers file.`

* How would we even create new files or directories then?
    * Please note since our users do not have sudo access, we can still write files in our own home directory. You would just simply omit `sudo`.

</details>

---
### 3.1 Create the directory 
- `mkdir -p ~/.config/containers/systemd`

---
### 3.2 Create the `.pod` quadlet file
- `nano ~/.config/containers/systemd/vaultwarden.pod`  
 
- ```systemd
  [Pod]
  PodName=vaultwarden
  Network=vaultwarden.network
  PublishPort=8080:8080
  ```

---    
### 3.3 Define the Pod Network
- For now, I couldn't get the Podman `.network` file to work. You could refer to the [Podman](https://github.com/dani-garcia/vaultwarden/wiki/Using-Podman) section if you need to define a `.network` file.
- We will leave out `.network` for this deployment and let [Pasta](https://docs.oracle.com/en/learn/ol-podman-pasta-networking/#introduction) do the work of networking our pod. 
---   
### 3.4 Making volumes persistent
- Create the file `~/.config/containers/systemd/vaultwarden-app.volume`
  ```systemd
  [Volume]
  VolumeName=vaultwarden-app
  ```
- Create the file `~/.config/containers/systemd/vaultwarden-db.volume`
  ```systemd
  [Volume]
  VolumeName=vaultwarden-db
  ```
---    
### 3.5 Setting the `.container` files
- Create the file `~/.config/containers/systemd/vaultwarden-app.container`
  ```systemd
  [Container]
  ContainerName=vaultwarden-app
  Environment=ROCKET_PORT=8080
  Environment=SIGNUPS_ALLOWED=true # Set this to false after setting up your first account on Vaultwarden and invite your users through the dashboard
  Environment=LOG_FILE=/data/vaultwarden.log
  HealthCmd=/healthcheck.sh
  HealthInterval=120s
  HealthRetries=10
  HealthTimeout=45s
  Image=docker.io/vaultwarden/server:1.34.3 # Use the latest working version
  Pod=vaultwarden.pod
  Secret=database_url,type=env,target=DATABASE_URL
  Secret=admin_token,type=env,target=ADMIN_TOKEN
  Volume=vaultwarden-app.volume:/data
  DNS=1.1.1.1
  [Unit]
  Requires=vaultwarden-db.service
  After=vaultwarden-db.service

  [Install]
  WantedBy=default.target
  ```
    - These are parameters for the vaultwarden-app but formatted and used by systemd.

- Create the file `~/.config/containers/systemd/vaultwarden-db.container`
  - ```systemd
    [Container]
    ContainerName=vaultwarden-db
    EnvironmentFile=/home/vaultwarden/vaultwarden/vaultwarden-db.env

    HealthCmd=/usr/bin/pg_isready -q -d vaultwarden -U vaultwarden
    HealthInterval=120s
    HealthRetries=10
    HealthTimeout=45s

    Image=docker.io/library/postgres:17
    Pod=vaultwarden.pod

    Secret=postgres_password,type=env,target=POSTGRES_PASSWORD
    Volume=vaultwarden-db.volume:/var/lib/postgresql/data

    [Install]
    WantedBy=default.target
    ```
      
## 4. Configuration 

- Setup the folders:

  - ```bash
    mkdir -p ~/vaultwarden
    mkdir -p ~/vaultwarden/data
    ```

- Create the environment file `~/vaultwarden/vaultwarden-db.env`

  - ```bash
    POSTGRES_USER=vaultwarden
    POSTGRES_DB=vaultwarden
    ```

## 5. Setting up Secrets 
Setup the following secrets for: `postgres_password`, `database_url` and `admin_token`. This assumes you have `POSTGRES_USER` & `POSTGRES_DB` as `vaultwarden`.

```bash
openssl rand -base64 32|podman secret create postgres_password -
echo "postgres://vaultwarden:$(podman secret inspect --showsecret --format '{{.SecretData}}' postgres_password)@vaultwarden-db/vaultwarden" | tr -d '\n' | podman secret create database_url -
echo -n "MySecretPassword" | argon2 "$(openssl rand -base64 32)" -e -id -k 65540 -t 3 -p 4| tr -d '\n' | podman secret create admin_token -
```
To make sure all the secrets are generated, run `podman secret ls`. `postgres_password`, `admin_token`, & `database_url` should show up. 

## 6. Deploying The Pod
```bash
systemctl --user daemon-reload
systemctl --user start vaultwarden-pod.service
```
To check if its generated use `systemctl --user list-unit-files | grep vaultwarden`

---
## Troubleshooting 
* `Failed to start vaultwarden-pod.service: Unit vaultwarden-pod.service not found.` 
    * If the service is not being generated, you can run `/usr/lib/systemd/system-generators/podman-system-generator --user --dryrun`

    * To reload systemd configuration,
     `systemctl --user daemon-reload`
     `systemctl --user restart vaultwarden-pod.service`
---
### Sources
- [YouTube: Michael Fox - Podman + Quadlet + Ansible: Rootless Service Management](https://www.youtube.com/watch?v=F0hhtDnTVwo)
- [Documentation: SUSE - Rootless Podman](https://documentation.suse.com/smart/container/pdf/rootless-podman_en.pdf)
---
### Additional Configurations
[Backing Up Vaultwarden](https://github.com/NoteStars/vaultwarden-podman-rootless/wiki/Backing-up-Vaultwarden)

