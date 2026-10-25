# Delegation-Friendly ZKMB

A delegation-friendly zero-knowledge middlebox integrating TLS traffic handling,
circuit generation, and delegated proof generation using DFS. The Docker
environment runs a client, TLS server, coordinator, and three proving workers.

This project extends the artifact for **DFS: Delegation-friendly zkSNARK and
Private Delegation of Provers**, by Yuncong Hu, Pratyush Mishra, Xiao Wang,
Jie Xie, Kang Yang, Yu Yu, and Yuwen Zhang (USENIX Security 2025), adding TLS
instrumentation, circuit parsing, Merkle-tree proof support, and a container network.

- [Original paper](https://www.usenix.org/conference/usenixsecurity25/presentation/hu-yuncong)
- [Original artifact, version cited by the artifact appendix](https://doi.org/10.5281/zenodo.14677896)
- [Artifact appendix](https://secartifacts.github.io/usenixsec2025/appendix-files/sec25cycle1ae-final77.pdf)

## Host requirements

| Requirement | Specification |
| --- | --- |
| Platform | Ubuntu 24.04, amd64 |
| Tools | Bash, Docker Engine, Docker Compose plugin, and Docker Buildx |
| RAM | At least 32 GB |
| Free disk space | At least 20 GB for images, build cache, and generated data |
| Network | Internet access during installation; an available Docker subnet |

Use a local, rootful Docker daemon. The default subnet is `172.20.0.0/16`.
To change it, update the subnet and service addresses in `docker-compose.yml`
and the routes in `docker/client-entrypoint.sh` and `docker/server-entrypoint.sh`,
then rebuild the client and server images.

Rust, Java 17, Python, MPI, and their build/runtime dependencies are installed
automatically in the container images.

## Setup

If Docker is not installed, configure its apt repository using the
[official Ubuntu instructions](https://docs.docker.com/engine/install/ubuntu/#install-using-the-apt-repository),
then install the packages:

```sh
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

From the repository root, build the images and generate the circuit and proof
setup data:

```sh
sudo docker compose build server client coordinator
mkdir -p artifacts/circuits examples/rss_snark/inst_folder
sudo docker compose --profile tools run --rm --no-deps setup
```

Omit `sudo` if your user already has Docker access. Setup writes the circuit and
input layout to `artifacts/circuits/` and proof setup data to
`examples/rss_snark/inst_folder/inst_13_0_1/`.

Run setup once, and repeat it when the circuit or proof format changes. Stop any
running simulation before regenerating setup data. Build and setup must finish
before launching a run.

## Run

```sh
sudo bash docker_compose_launcher.sh run
```

The client sends a TLS request through the coordinator, generates session inputs
using `JRPCInputGenerator`, and distributes shares to the workers for delegated
proof generation. Inputs are written inside the client container under
`/app/client-runtime/files/`; the setup circuit and input layout are mounted read-only.

The Nginx TLS server acknowledges `POST /mcp` with
`{"status":"ok","simulation":true}` without executing the requested command.

Running the launcher without arguments is equivalent to `run`.

| Launcher command | Purpose |
| --- | --- |
| `sudo bash docker_compose_launcher.sh check` | Check Docker access and Compose configuration |
| `sudo bash docker_compose_launcher.sh logs` | Show the last 100 log lines per service |
| `sudo bash docker_compose_launcher.sh down` | Stop and remove containers |
| `bash docker_compose_launcher.sh --help` | Show command usage |

The launcher saves service logs and removes containers after a successful run.
On failure, containers remain available for inspection:

```sh
sudo bash docker_compose_launcher.sh logs
sudo bash docker_compose_launcher.sh down
```

`down` preserves setup data, measurements, and named volumes. Save any debugging
files stored inside the containers before removing them.

Optionally, repeat runs or adjust time limits:

```sh
sudo env REPS=3 STARTUP_TIMEOUT=1800 RUN_TIMEOUT=7200 bash docker_compose_launcher.sh run
```

Defaults are one run, 30 minutes per startup readiness check, and two hours per
run. The example runs three times with the default timeouts. These timeouts do
not apply to build or setup.

## Results

| Output | Contents |
| --- | --- |
| `measurments_client/output_client1.json` | Client output and timing |
| `measurments_client/memory_client1.json` | Client memory and CPU samples |
| `measurments_client/cputime_client1.txt` | Sampled client CPU time |
| `measurments/run_1.log` | Combined service logs |

The number in each filename identifies the run. Run IDs restart at 1 on every
launcher invocation, so copy results elsewhere before launching again to avoid
overwriting them.

## Optional measurements

To collect additional worker and coordinator CPU/memory metrics, prepare a host
Python environment and enable measurement:

```sh
sudo apt update
sudo apt install -y python3 python3-venv
python3 -m venv .venv
.venv/bin/python -m pip install docker
sudo bash docker_compose_launcher.sh run --measure
```

`--measure` runs `measurment_docker_streaming.py` using the prepared `.venv/`
and stops it after each run. It writes `metrics_<container>_<run>.csv` in the
repository root for `work1_dfs`, `work2_dfs`, `work3_dfs`, and `coord_dfs`.
Client measurements are collected on every run, including without `--measure`.
