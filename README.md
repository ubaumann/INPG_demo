# INPG Stack Demo - Infrahub, NUTS, Prometheus and Grafana

## Getting Started
You can run this demo on your PC using docker, or using Github Codespaces.

### Prerequisites on local Instances

If you are not using Devcontainer or Github Codespaces, you will need to do the following steps. Otherwise jump to the step Demo.

Export those variables:

```shell
export INFRAHUB_ADDRESS="http://localhost:8000"
export INFRAHUB_API_TOKEN="06438eb2-8019-4776-878c-0941b1f1d1ec"
export CEOS_DOCKER_IMAGE="registry.opsmill.io/external/ceos-image:4.29.0.2F"
export LINUX_HOST_DOCKER_IMAGE="registry.opsmill.io/external/alpine-host:v3.1.1"
```

### Install the Infrahub SDK

```shell
poetry install --no-interaction --no-ansi --no-root
```

### Start Infrahub

```shell
poetry run invoke start
```

### Load schema and data into Infrahub

This will create :

- Basics data (Account, organization, ASN, Device Type, and Tags)
- Locations data (Locations, VLANs, and Prefixes)
- Topology data (Topology, Topology Elements)
- Security data (Policies, rules, objects)

```shell
poetry run invoke load-schema load-data
```

## Demo

### 1. Add the repository into Infrahub (Replace GITHUB_USER and GITHUB_TOKEN)

> [!NOTE]
> Reference the [Infrahub documentation](https://docs.infrahub.app/guides/repository) for the multiple ways this can be done.
> Public repositories do not need credentials for read-only.

```graphql
mutation AddCredential {
  CorePasswordCredentialCreate(
    data: {
      name: {value: "my-git-credential"},
      username: {value: "<GITHUB_USERNAME>"},
      password: {value: "<GITHUB_TOKEN>"}
    }
  ) {
    ok
    object {
      hfid
    }
  }
}


mutation AddRepository{
  CoreRepositoryCreate(
    data: {
      name: { value: "INPG_demo" }
      location: { value: "https://github.com/marcom4rtinez/INPG_demo.git" }
      # The HFID return from the previous mutation. Will be the name of the credentials
      credential: { hfid: "my-git-credential" }
    }
  ) {
    ok
    object {
      id
    }
  }
}
```

### 2. Generate a Topology (Device, Interfaces, Cabling, BGP Sessions, ...)


> [!NOTE]
> The example below creates the topology fra05-pod1

```shell
poetry run infrahubctl run bootstrap/generate_topology.py topology=fra05-pod1
```


### 3. Render Artifacts

Use the Infrahub UI to inspect the artifact on Topology and Device objects.


### 4. Deploy your environment to containerlabs

The containerlab generator automatically generates a containerlab topology artifact for every topology. Every device has its startup config as an artifact.

```shell
# Download all artifacts automatically to ./generated-configs/
# poetry run python3 scripts/get_configs.py --help  for all options
poetry run python3 scripts/get_configs.py -n -c -d

# Start the containerlab
sudo -E containerlab deploy -t ./generated-configs/clab/fra05-pod1.yml --reconfigure

# Set the IP address on the interface connecting infrahub server with the nuts container
docker exec --privileged inpg_demo-infrahub-server-1 ip address add 172.20.0.2/16 dev eth1
```

### 5. Start Prometheus and Grafana

NUTS can be run independently, however if visualization is desired it can be run with the `--metrics` flag and it will be sent to grafana.

```shell
cd nuts
docker compose up -d
cd -
```

### 6. Use NUTS to test the environment

The NUTS test client in the containerlab is set up to be ready for test execution. Tests where pulled in step 4. with `get_config.py`.

> [!NOTE]
> By default, the metrics are sent to the prometheus pushgateway from the dokcer-compose.yml
> ```shell
> # Set the variables in the nuts container if you want to change the target
> export PROMETHEUS_PUSHGATEWAY_URL="http://prom-pushgateway:9091"
> export PROMETHEUS_PUSHGATEWAY_JOB="nuts"
> ```

```shell
# Enter the container
docker exec -it clab-demo-dc-fabric-nuts bash

# Execute all generated tests for fra05-pod1
pytest tests/test_fra05-pod1-* --metrics
```

Go to grafana and see the dashboard on http://localhost:3000 (admin:infrahub). After login go to the Infrahub NUTS Dashboard

```shell
# Do it continously to generate visibility in grafana
count=1; while [ $count -le 100 ]; do pytest tests/test_fra05-pod1-* --metrics; ((count++)); done
```
