Osmosis Dashboard
========

![OsmosisDashboard1](https://raw.githubusercontent.com/czarcas7ic/Osmosis-Grafana-Prometheus-Docker/master/screens/Osmosis_Dashboard_1.png)

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

Clone this repository on your Docker host, cd into test directory and run compose up

```
git clone https://github.com/czarcas7ic/Osmosis-Grafana-Prometheus-Docker.git
cd Osmosis-Grafana-Prometheus-Docker
docker-compose up -d
```

This command will create the following containers

* Grafana (visualize metrics) `http://<host-ip>:3000`
* Prometheus (metrics database) `http://<host-ip>:9092`
* Prometheus-Pushgateway (push acceptor for ephemeral and batch jobs) `http://<host-ip>:9091`
* NodeExporter (host metrics collector)
* cAdvisor (containers metrics collector)

Once you have run docker-compose (**and Osmosis has caught up to the head of the chain**), simply go to `http://<host-ip>:3000`, login with `admin` and `admin` as the username and password, set your new password, and go to the dashboards tab (the icon that looks like four squares). Select browse and then select the `Osmosis Dashboard` from the home screen. The dashboard can also be reached directly at `http://<host-ip>:3000/d/UJyurCTWz/osmosis-dashboard`

## Uninstall / Deactivate

To shut down all of the above docker containers but retain the data

```
docker-compose down
```

To shut down and delete all metrics collected

```
docker-compose down --volumes
```

## Setup Grafana

Navigate to `http://<host-ip>:3000` and login with user ***admin*** password ***admin***. You can change the credentials in the compose file or by supplying the `ADMIN_USER` and `ADMIN_PASSWORD` environment variables via .env file on compose up. The config file can be added directly in grafana part like this
```
grafana:
  image: grafana/grafana:5.2.4
  env_file:
    - config

```
and the config file format should have this content
```
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=changeme
GF_USERS_ALLOW_SIGN_UP=false
```
If you want to change the password, you have to remove this entry, otherwise the change will not take effect
```
- grafana_data:/var/lib/grafana
```

Grafana is preconfigured with dashboards and Prometheus as the default data source:

* Name: Prometheus
* Type: Prometheus
* Url: http://prometheus:9090
* Access: proxy

***Osmosis Dashboard***

The Osmosis Dashboard shows key metrics for monitoring the chain state as well as machine resource usage:

![OsmosisDashboard1](https://raw.githubusercontent.com/czarcas7ic/Osmosis-Grafana-Prometheus-Docker/master/screens/Osmosis_Dashboard_1.png)

![OsmosisDashboard2](https://raw.githubusercontent.com/czarcas7ic/Osmosis-Grafana-Prometheus-Docker/master/screens/Osmosis_Dashboard_2.png)

![OsmosisDashboard3](https://raw.githubusercontent.com/czarcas7ic/Osmosis-Grafana-Prometheus-Docker/master/screens/Osmosis_Dashboard_3.png)


## WIP

## Define alerts

Three alert groups have been setup within the [alert.rules](https://github.com/Einsteinish/Docker-Compose-Prometheus-and-Grafana/blob/master/prometheus/alert.rules) configuration file:

* Monitoring services alerts [targets](https://github.com/Einsteinish/Docker-Compose-Prometheus-and-Grafana/blob/master/prometheus/alert.rules#L2-L11)
* Docker Host alerts [host](https://github.com/Einsteinish/Docker-Compose-Prometheus-and-Grafana/blob/master/prometheus/alert.rules#L13-L40)
* Docker Containers alerts [containers](https://github.com/Einsteinish/Docker-Compose-Prometheus-and-Grafana/blob/master/prometheus/alert.rules#L42-L69)

You can modify the alert rules and reload them by making a HTTP POST call to Prometheus:

```
curl -X POST http://admin:admin@<host-ip>:9090/-/reload
```

***Monitoring services alerts***

Trigger an alert if any of the monitoring targets (node-exporter and cAdvisor) are down for more than 30 seconds:

```yaml
- alert: monitor_service_down
    expr: up == 0
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "Monitor service non-operational"
      description: "Service {{ $labels.instance }} is down."
```

***Docker Host alerts***

Trigger an alert if the Docker host CPU is under high load for more than 30 seconds:

```yaml
- alert: high_cpu_load
    expr: node_load1 > 1.5
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Server under high load"
      description: "Docker host is under high load, the avg load 1m is at {{ $value}}. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}."
```

Modify the load threshold based on your CPU cores.

Trigger an alert if the Docker host memory is almost full:

```yaml
- alert: high_memory_load
    expr: (sum(node_memory_MemTotal_bytes) - sum(node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes) ) / sum(node_memory_MemTotal_bytes) * 100 > 85
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Server memory is almost full"
      description: "Docker host memory usage is {{ humanize $value}}%. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}."
```

Trigger an alert if the Docker host storage is almost full:

```yaml
- alert: high_storage_load
    expr: (node_filesystem_size_bytes{fstype="aufs"} - node_filesystem_free_bytes{fstype="aufs"}) / node_filesystem_size_bytes{fstype="aufs"}  * 100 > 85
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Server storage is almost full"
      description: "Docker host storage usage is {{ humanize $value}}%. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}."
```

***Docker Containers alerts***

Trigger an alert if a container is down for more than 30 seconds:

```yaml
- alert: jenkins_down
    expr: absent(container_memory_usage_bytes{name="jenkins"})
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "Jenkins down"
      description: "Jenkins container is down for more than 30 seconds."
```

Trigger an alert if a container is using more than 10% of total CPU cores for more than 30 seconds:

```yaml
- alert: jenkins_high_cpu
    expr: sum(rate(container_cpu_usage_seconds_total{name="jenkins"}[1m])) / count(node_cpu_seconds_total{mode="system"}) * 100 > 10
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Jenkins high CPU usage"
      description: "Jenkins CPU usage is {{ humanize $value}}%."
```

Trigger an alert if a container is using more than 1.2GB of RAM for more than 30 seconds:

```yaml
- alert: jenkins_high_memory
    expr: sum(container_memory_usage_bytes{name="jenkins"}) > 1200000000
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Jenkins high memory usage"
      description: "Jenkins memory consumption is at {{ humanize $value}}."
```

## Setup alerting

The AlertManager service is responsible for handling alerts sent by Prometheus server.
AlertManager can send notifications via email, Pushover, Slack, HipChat or any other system that exposes a webhook interface.
A complete list of integrations can be found [here](https://prometheus.io/docs/alerting/configuration).

You can view and silence notifications by accessing `http://<host-ip>:9093`.

The notification receivers can be configured in [alertmanager/config.yml](https://github.com/Einsteinish/Docker-Compose-Prometheus-and-Grafana/blob/master/alertmanager/config.yml) file.

To receive alerts via Slack you need to make a custom integration by choose ***incoming web hooks*** in your Slack team app page.
You can find more details on setting up Slack integration [here](http://www.robustperception.io/using-slack-with-the-alertmanager/).

Copy the Slack Webhook URL into the ***api_url*** field and specify a Slack ***channel***.

```yaml
route:
    receiver: 'slack'

receivers:
    - name: 'slack'
      slack_configs:
          - send_resolved: true
            text: "{{ .CommonAnnotations.description }}"
            username: 'Prometheus'
            channel: '#<channel>'
            api_url: 'https://hooks.slack.com/services/<webhook-id>'
```

![Slack Notifications](https://raw.githubusercontent.com/Einsteinish/Docker-Compose-Prometheus-and-Grafana/master/screens/Slack_Notifications.png)

## Sending metrics to the Pushgateway

The [pushgateway](https://github.com/prometheus/pushgateway) is used to collect data from batch jobs or from services.

To push data, simply execute:

    echo "some_metric 3.14" | curl --data-binary @- http://user:password@localhost:9091/metrics/job/some_job

Please replace the `user:password` part with your user and password set in the initial configuration (default: `admin:admin`).

## Updating Grafana to v5.2.2

[In Grafana versions >= 5.1 the id of the grafana user has been changed](http://docs.grafana.org/installation/docker/#migration-from-a-previous-version-of-the-docker-container-to-5-1-or-later). Unfortunately this means that files created prior to 5.1 won’t have the correct permissions for later versions.

| Version |   User  | User ID |
|:-------:|:-------:|:-------:|
|  < 5.1  | grafana |   104   |
|  \>= 5.1 | grafana |   472   |

There are two possible solutions to this problem.
- Change ownership from 104 to 472
- Start the upgraded container as user 104

##### Specifying a user in docker-compose.yml

To change ownership of the files run your grafana container as root and modify the permissions.

First perform a `docker-compose down` then modify your docker-compose.yml to include the `user: root` option:

```
  grafana:
    image: grafana/grafana:5.2.2
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/datasources:/etc/grafana/datasources
      - ./grafana/dashboards:/etc/grafana/dashboards
      - ./grafana/setup.sh:/setup.sh
    entrypoint: /setup.sh
    user: root
    environment:
      - GF_SECURITY_ADMIN_USER=${ADMIN_USER:-admin}
      - GF_SECURITY_ADMIN_PASSWORD=${ADMIN_PASSWORD:-admin}
      - GF_USERS_ALLOW_SIGN_UP=false
    restart: unless-stopped
    expose:
      - 3000
    networks:
      - monitor-net
    labels:
      org.label-schema.group: "monitoring"
```

Perform a `docker-compose up -d` and then issue the following commands:

```
docker exec -it --user root grafana bash

# in the container you just started:
chown -R root:root /etc/grafana && \
chmod -R a+r /etc/grafana && \
chown -R grafana:grafana /var/lib/grafana && \
chown -R grafana:grafana /usr/share/grafana
```

To run the grafana container as `user: 104` change your `docker-compose.yml` like such:

```
  grafana:
    image: grafana/grafana:5.2.2
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/datasources:/etc/grafana/datasources
      - ./grafana/dashboards:/etc/grafana/dashboards
      - ./grafana/setup.sh:/setup.sh
    entrypoint: /setup.sh
    user: "104"
    environment:
      - GF_SECURITY_ADMIN_USER=${ADMIN_USER:-admin}
      - GF_SECURITY_ADMIN_PASSWORD=${ADMIN_PASSWORD:-admin}
      - GF_USERS_ALLOW_SIGN_UP=false
    restart: unless-stopped
    expose:
      - 3000
    networks:
      - monitor-net
    labels:
      org.label-schema.group: "monitoring"
```
