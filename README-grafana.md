# Using ShipXplorer with Grafana and Prometheus

[`Grafana`](https://grafana.com/) is an analytics platform that can provide alternative graphs for `readsb`.

In this guide we will be using [`Prometheus`](https://prometheus.io/) as the data repository.

Using Grafana and Prometheus in this configuration does not require a plan, account, or credentials for their respective cloud offerings.

## Hardware requirements

Prometheus will store a lot of data, and Grafana will do a lot of data queries. As a result, it would be better if you run these containers on a different system than your feeder Raspberry Pi. This will leave your Pi focused on data collection and processing, and unbothered by the CPU and Disk IO load that Prometheus/Grafana will cause.

You *can* do it on a single system. We're assuming below that you are not. If you do it on a single system, then you can combine the `docker-compose.yml` components in a single file

## Step 1: Make Prometheus data available for the ShipXplorer

- Edit your ShipXplorer's `docker-compose.yml` file and ensure that the following is set for the `shipxplorer` service:

```yaml
    environment:
      - PROMETHEUS_ENABLE=true
    ports:
      - 9273:9273
```

Now recreate the ShipXplorer container (`docker-compose up -d shipxplorer`) and it will generate Prometheus data.

## Step 2: create a container stack for `prometheus` and `grafana`

You can skip this if you already have Prometheus and Grafana installed.

On the machine where you will run Prometheus and Grafana, create a docker-compose file in the `/opt/grafana` directory:

```bash
sudo mkdir -p -m777 /opt/grafana/grafana/appdata /opt/grafana/prometheus/config /opt/grafana/prometheus/data
cd /opt/grafana
cat > docker-compose.yml
```

Now paste in the following text *):

<details>
  <summary>&lt;&dash;&dash; Click the arrow to see the <code>docker-compose.yml</code> text</summary>

```yaml
version: '3.9'

volumes:
  grafana:
    driver: local
    driver_opts:
      type: none
      device: "/opt/grafana/grafana/appdata"
      o: bind
  prom-config:
    driver: local
    driver_opts:
      type: none
      device: "/opt/grafana/prometheus/config"
      o: bind
  prom-data:
    driver: local
    driver_opts:
      type: none
      device: "/opt/grafana/prometheus/data"
      o: bind

services:
  grafana:
    image: grafana/grafana-oss:latest
    restart: unless-stopped
    container_name: grafana
    hostname: grafana
    tty: true
    # uncomment the following section and set the variables if you are exposing Grafana to the internet behind a rev web proxy:
    environment:
    # snuids-radar-panel plugin is needed for polar plots:
      - GF_INSTALL_PLUGINS=snuids-radar-panel
    # uncomment and set the following variables if you are exposing Grafana to the internet behind a rev web proxy:
    #   - GF_SERVER_ROOT_URL=https://mywebsite.com/grafana/
    #   - GF_SERVER_SERVE_FROM_SUB_PATH=true
    ports:
      - 3000:3000
    volumes:
      - grafana:/var/lib/grafana

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    hostname: prometheus
    restart: unless-stopped
    tmpfs:
      - /tmp
    volumes:
      - prom-config:/etc/prometheus
      - prom-data:/prometheus
    ports:
      - 9090:9090
```

</details>

*) The volume definition structure is written this way purposely to ensure that the containers can place files in the persistent directories. Do not try to "directly" map volumes (`/opt/grafana/grafana/appdata:/var/lib/grafana`).

You should be able to see the following directories:

- `/opt/grafana/grafana/appdata`
- `/opt/grafana/prometheus/config`
- `/opt/grafana/prometheus/data`

Download and create Grafana and Prometheus for the first time with this command:

```bash
docker compose up -d
```

## Step 3: Configuring Prometheus

Prometheus needs to be told where to look for the data from the ShipXplorer. We will create a target prometheus configuration file that does this, please copy and paste the following. Make sure to replace `ip_of_ShipXplorer_machine:90` with the IP address or hostname of the machine and the web port where `shipxplorer` is running:

```bash
docker exec -it prometheus sh -c "echo -e \"  - job_name: 'ais'\n    static_configs:\n      - targets: ['ip_of_ShipXplorer_machine:90']\" >> /etc/prometheus/prometheus.yml"
docker restart prometheus
```

(If you screw this up, **do NOT** re-run the command. Instead, try `sudo nano /opt/grafana/prometheus/config/prometheus.yml` and fix it that way.)

## Accessing Prometheus and Grafana via your browser

You should be able to point your web browser at:

* `http://docker.host.ip.addr:9090/` to access the `prometheus` console.
* `http://docker.host.ip.addr:3000/` to access the `grafana` console, use admin/admin as initial credentials, you should be prompted to change the password on first login.

Remember to change `docker.host.ip.addr` to the IP address of the machine where Prometheus and Grafana are located.

## Configuring data source and dashboard in Grafana

After you have logged into the `grafana` console the following manual steps are required to connect to `prometheus` as the data source. **You must do this** even if you are already connected to Prometheus for other purposes (such as for getting `readsb`/`ultrafeeder`/`tar1090` data)

1. In Grafana, click `Configuration` (cogwheel) -> `Data Sources`
2. Click `Add a new data source` in the main panel
3. Click `Prometheus` from the list of options provided
4. Input or select the following options, if the option is not listed, do not input anything for that option:

Option | Input
------------- | -------------
Name | `ais`
URL | `http://prometheus:9090/`

Clicking `Save & Test` should return a green message indicating success. The dashboard can now be imported with the following steps

1. Hover over the `four squares` icon in the sidebar, click `+ Import`
2. Enter `18417` into the `Import via grafana.com` section and click `Load`
3. Select `readsb` from the bottom drop down list
4. Click `Import` on the subsequent dialogue box

At this point you should see a very nice dashboard, you can find it under `General` in the `Dashboards` section.
<img src="https://user-images.githubusercontent.com/15090643/228942953-ed8b64aa-3a38-4c6f-bd42-e929b72399b2.png">
<img src="https://user-images.githubusercontent.com/15090643/228943041-7e135856-543a-416a-9331-50853d2e0929.png">
<img src="https://user-images.githubusercontent.com/15090643/228943083-c017c5a0-f5aa-4d03-b241-8e58f2c8a5f6.png">
