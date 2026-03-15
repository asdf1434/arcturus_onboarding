# Environment Setup

## Docker installation

We develop inside a Docker container — think of it as a lightweight virtual machine with all our tools pre-installed, so everyone has the same environment.

1. Install [Git](https://git-scm.com/downloads) and [Docker](https://www.docker.com/products/docker-desktop/) for your OS
2. Clone the Docker environment:
   ```bash
   git clone --recurse-submodules git@github.com:ArcturusNavigation/onboarding_lab_docker.git
   ```
3. `cd` into the cloned directory and start the container:
   ```bash
   docker compose up
   ```
4. In a separate terminal, open a shell inside the container:
   ```bash
   docker compose exec arcturus bash
   ```

> Apple Silicon users: edit `docker-compose.yml` and change the `image` field to `popkarsd/arcturus_onboarding:aarch64`

Some things to know:
- Files in `/home/arcturus` persist between restarts. Changes elsewhere are lost.
- Always run `docker compose down` before restarting.
- Login credentials: username `arcturus`, password `arcturus`.

## Building the workspace

Inside the container:

```bash
cd ~/arcturus/dev_ws
colcon build --symlink-install
source install/setup.bash
```

You need to run `source install/setup.bash` every time you open a new terminal.

## Accessing the GUI

Open a browser and go to: `http://localhost:6080/vnc.html?resize=remote`

This gives you a desktop inside the container where you can see the visualization.
