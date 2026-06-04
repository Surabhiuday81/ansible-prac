# Ansible Deployment — Spring Petclinic

This project automates the deployment of the **Spring Petclinic** application using Ansible and Docker. The application is containerised and hosted on Docker Hub at `uday6395/spring-petclinic`. Ansible handles all steps — from installing Docker on the target host to pulling the image, running the container, and verifying the deployment.

---

## Project Structure

```
ansible-01/
├── deploy.yml                  # Main Ansible playbook
├── inventory.ini               # Inventory file defining target hosts
├── group_vars/
│   └── app_servers.yml         # Variables for the app_servers group
└── README.md                   # This file
```

---

## Prerequisites

| Requirement | Details |
|---|---|
| Ansible | Core 2.9+ (tested on 2.19.0) |
| community.docker collection | Installed via `ansible-galaxy` |
| Target OS | Ubuntu / Debian (uses `apt`) |
| Docker Hub image | `uday6395/spring-petclinic` |
| Sudo access | Required on the target host (`become: true`) |

Install the required Ansible collection once:

```bash
ansible-galaxy collection install community.docker
```

---

## Configuration

### `inventory.ini`

Defines the target host where the application will be deployed. This setup uses **localhost** so no remote server or SSH key is required.

```ini
[app_servers]
petclinic-host ansible_host=127.0.0.1 ansible_connection=local ansible_user=uday

[app_servers:vars]
ansible_python_interpreter=/usr/bin/python3
```

To deploy to a remote server instead, replace `127.0.0.1` with the server's IP address, remove `ansible_connection=local`, and add `ansible_ssh_private_key_file=~/.ssh/id_rsa`.

### `group_vars/app_servers.yml`

Holds all deployment variables. Override any of these at runtime using `-e`.

```yaml
docker_image: "uday6395/spring-petclinic"
docker_tag: "latest"
container_name: "spring-petclinic"
container_port: 8080
host_port: 8080
```

---

## Playbook Walkthrough — `deploy.yml`

### Pre-tasks

Before any installation begins, the playbook:

1. **Pings the target host** using `ansible.builtin.ping` to confirm connectivity.
2. **Prints deployment info** — the target hostname, Docker image name with tag, and port mapping — so you can verify the config before any changes are made.

---

### Block 1 — Install Docker

These tasks prepare the target host with Docker and all its dependencies.

1. **Update apt cache** — refreshes the package list (cached for 1 hour to avoid redundant updates on re-runs).
2. **Install Docker dependencies** — installs `apt-transport-https`, `ca-certificates`, `curl`, `gnupg`, `lsb-release`, and `python3-pip` via `apt`.
3. **Install Docker** — installs the `docker.io` package.
4. **Install python3-docker** — installs the `python3-docker` apt package, which is required by Ansible's `community.docker.*` modules to communicate with the Docker daemon. Using `apt` here (instead of `pip`) avoids the PEP 668 externally-managed-environment restriction on Ubuntu with Python 3.13+.
5. **Start and enable Docker** — ensures the Docker `systemd` service is running and set to start on boot.
6. **Add user to docker group** — adds the Ansible user to the `docker` group so Docker commands can run without `sudo` in future sessions.

---

### Block 2 — Pull Docker Image

1. **Pull the image** — uses `community.docker.docker_image` to pull `uday6395/spring-petclinic:{{ docker_tag }}` from Docker Hub. `force_source: true` ensures the latest version of the tag is always pulled, even if the image already exists locally.
2. **Print image digest** — confirms which image and tag was pulled.

The `docker_tag` variable defaults to `latest` but can be overridden at runtime to pin a specific version (see Usage section below).

---

### Block 3 — Deploy Container

1. **Remove existing container** — stops and removes any container already running under the name `spring-petclinic`. This ensures a clean re-deploy on every run. `ignore_errors: true` prevents failure if no container exists yet.
2. **Run the container** — launches the container using `community.docker.docker_container` with:
   - Port mapping `8080:8080` (host:container)
   - `restart_policy: always` so the container automatically restarts on system reboot or crash
   - `SPRING_PROFILES_ACTIVE=default` environment variable
   - A built-in Docker **healthcheck** that polls `/actuator/health` every 30 seconds, with a 60-second startup grace period
3. **Print container status** — confirms the container's current state (e.g. `running`).

---

### Block 4 — Health Check

1. **Wait for port 8080** — uses `ansible.builtin.wait_for` to pause until port 8080 is accepting connections, with a 10-second initial delay and a 120-second timeout.
2. **HTTP 200 check** — uses `ansible.builtin.uri` to make a GET request to `http://localhost:8080` and assert an HTTP 200 response. Retries up to 5 times with a 10-second delay between attempts.
3. **Confirm success** — prints the application URL and HTTP status code.

---

### Post-tasks

1. **Gather container info** — fetches live container metadata using `community.docker.docker_container_info`.
2. **Print deployment summary** — prints a formatted summary including the host, image, container name, container status, and the application URL.

---

### Handlers

- **Restart Docker service** — a handler that restarts the Docker `systemd` service if notified by any task (e.g. after a config change). Not triggered in the default flow but available for extension.

---

## Usage

### Standard deployment (latest image)

```bash
ansible-playbook -i inventory.ini deploy.yml --ask-become-pass
```

### Deploy a specific image version

```bash
ansible-playbook -i inventory.ini deploy.yml -e "docker_tag=1.0.0" --ask-become-pass
```

### Dry run (no changes made)

```bash
ansible-playbook -i inventory.ini deploy.yml --check --ask-become-pass
```

### Skip sudo password prompt (configure once)

```bash
echo "uday ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/uday
# After this, --ask-become-pass is no longer needed
ansible-playbook -i inventory.ini deploy.yml
```

---

## Verifying the Deployment

Once the playbook completes successfully, open your browser and navigate to:

```
http://localhost:8080
```

You should see the Spring Petclinic home page.

To check the running container manually:

```bash
docker ps
docker logs spring-petclinic
```

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `sudo-rs: interactive authentication required` | `become: true` needs sudo password | Run with `--ask-become-pass` |
| `externally-managed-environment` (pip) | Python 3.13+ blocks system-wide pip | Use `apt install python3-docker` instead |
| `community.docker not found` | Collection not installed | Run `ansible-galaxy collection install community.docker` |
| Port 8080 already in use | Another service occupying the port | Run `sudo lsof -i :8080` and stop the conflicting process |
| Container exits immediately | App startup failure | Check logs with `docker logs spring-petclinic` |

---

## Docker Image

| Property | Value |
|---|---|
| Registry | Docker Hub |
| Image | `uday6395/spring-petclinic` |
| Default tag | `latest` |
| Exposed port | `8080` |
| Application | Spring Petclinic (Spring Boot) |