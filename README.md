Osmosis Dashboard
========

![OsmosisDashboard1](https://raw.githubusercontent.com/osmosis-labs/osmosis-monitor/master/screens/Osmosis_Dashboard_1.png)

A monitoring solution for Osmosis full nodes with [Prometheus](https://prometheus.io/), [Grafana](http://grafana.org/), [cAdvisor](https://github.com/google/cadvisor),
[NodeExporter](https://github.com/prometheus/node_exporter) and alerting with [AlertManager](https://github.com/prometheus/alertmanager).


## Guide

### Prerequisites

* Docker Engine >= 1.13
```
sudo apt-get remove docker docker-engine docker.io
sudo apt-get update
sudo apt install docker.io -y
```
* Docker Compose >= 1.11
```
sudo apt install docker-compose -y
```
* Osmosis full node running
  - See https://get.osmosis.zone or https://docs.osmosis.zone/developing/cli/install.html to install the Osmosis binary
  - Set `prometheus = true` under instrumentation in the config.toml
  - Set `enable = true` and `prometheus-retention-time = 1` under telemetry in the app.toml
* Ensure the following ports are not in use
  - 3000
  - 9100
  - 9092
  - 8001

### Install

Set your public IP as an environment variable

```
export HOST_IP=$(dig +short txt ch whoami.cloudflare @1.0.0.1)
```

Clone this repository on your Docker host, cd into test directory and run docker-compose up

```
git clone https://github.com/osmosis-labs/osmosis-monitor.git
cd Osmosis-Grafana-Prometheus-Docker
docker-compose up -d
```

This command will create the following containers

* Grafana (visualize metrics) `http://<host-ip>:3000`
* Prometheus (metrics database) `http://<host-ip>:9092`
* Prometheus-Pushgateway (push acceptor for ephemeral and batch jobs) `http://<host-ip>:9091`
* NodeExporter (host metrics collector)
* cAdvisor (containers metrics collector)

Once you have run docker-compose (**and Osmosis has caught up to the head of the chain**), simply go to `http://<host-ip>:3000`, login with `admin` and `admin` as the username and password, set your new password, and go to the dashboards tab (the icon that looks like four squares). Select browse and then select the `Osmosis Dashboard`. 

The dashboard can also be reached directly at `http://<host-ip>:3000/d/UJyurCTWz/osmosis-dashboard`

## Uninstall / Deactivate

To shut down all of the above docker containers but retain the data

```
docker-compose down
```

To shut down and delete all metrics collected

```
docker-compose down --volumes
```

## Osmosis Dashboard

The Osmosis Dashboard shows key metrics for monitoring the chain state as well as machine resource usage:

![OsmosisDashboard1](https://raw.githubusercontent.com/osmosis-labs/osmosis-monitor/master/screens/Osmosis_Dashboard_1.png)

![OsmosisDashboard2](https://raw.githubusercontent.com/osmosis-labs/osmosis-monitor/master/screens/Osmosis_Dashboard_2.png)

![OsmosisDashboard3](https://raw.githubusercontent.com/osmosis-labs/osmosis-monitor/master/screens/Osmosis_Dashboard_3.png)
