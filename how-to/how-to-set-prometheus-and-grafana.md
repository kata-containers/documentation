# How to setup Prometheus and Grafana as observability platform

Prometheus and Grafana are open sources analytics & monitoring solutions. This how-to will introduce a simple setup for a single node configuration with Kata Containers, Prometheus and Grafana.

> **Warning**: This how-to is only for evaluation purpose, you **SHOULD NOT** running it in production.

* [Install Prometheus](#install-prometheus)
* [Install Grafana](#install-grafana)
* [Setup Grafana](#setup-grafana)
  * [Create `datasource`](#create-datasource)
  * [Import dashboard](#import-dashboard)

Both Prometheus and Grafana can be installed using pre-compiled distributions.

## Install Prometheus

Download and extract Prometheus.

```
$ export VERSION=2.18.1
$ curl -sL https://github.com/prometheus/prometheus/releases/download/v${VERSION}/prometheus-${VERSION}.linux-amd64.tar.gz -o prometheus-${VERSION}.linux-amd64.tar.gz
$ tar xvfz prometheus-${VERSION}.linux-amd64.tar.gz
$ cd prometheus-${VERSION}.linux-amd64
```

To enable Prometheus collection of Kata Containers metrics, add `magent` as a Prometheus target.

Open `prometheus.yml` and add `localhost:8090` to `scrape_configs` -> `static_configs`:

```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:8090']
```

And then start Prometheus server:

```
$ ./prometheus --config.file=prometheus.yml
```

Prometheus listens on address `*:9000` by default.

## Install Grafana

In another terminal session, download and extract Grafana.

```
$ export VERSION=7.0.1
$ curl -sL https://dl.grafana.com/oss/release/grafana-${VERSION}.linux-amd64.tar.gz -o grafana-${VERSION}.linux-amd64.tar.gz
$ tar -zxvf grafana-${VERSION}.linux-amd64.tar.gz
$ cd grafana-${VERSION}
$ bin/grafana-server
```

## Setup Grafana

After the Grafana server is started, you can open http://localhost:3000 to access Grafana. For Grafana 7.0.1, the default user/password is `admin/admin`. You can modify the default account and adjust other security settings by editing the [Grafana configuration](https://grafana.com/docs/grafana/latest/installation/configuration/#security).

To use Grafana show data from Prometheus, you must create a Prometheus `datasource` and dashboard.


### Create `datasource`

Open http://localhost:3000/datasources/new in your browser, select Prometheus from time series databases list.

Normally you only need to set `URL` to http://localhost:9090 to let it work, and leave the name as `Prometheus` as default.

### Import dashboard

A [sample dashboard](data/dashboard.json) for Kata Containers metrics is provided which can be imported to Grafana for evaluation.

You can import this dashboard using [browser](http://localhost:3000/dashboard/import), or using `curl` command in console.


```
$ curl -XPOST -i localhost:3000/api/dashboards/import \
    -u admin:admin \
    -H "Content-Type: application/json" \
	-d "{\"dashboard\":$(curl -sL https://raw.githubusercontent.com/kata-containers/documentation/master/how-to/data/dashboard.json )}"
```