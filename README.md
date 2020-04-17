# Tado exporter

| Build status | Vulnerabilities |
|--------------|-----------------|
| ![GitHub Workflow Status](https://img.shields.io/github/workflow/status/easimon/tado-exporter/Docker%20container%20build?label=docker%20build) ![GitHub Workflow Status](https://img.shields.io/github/workflow/status/easimon/tado-exporter/JDK%20compatibility%20tests?label=jdk%20compatibility%20tests) | ![Snyk Vulnerabilities for Tado Exporter](https://img.shields.io/snyk/vulnerabilities/github/easimon/tado-exporter/tado-exporter/pom.xml?label=tado-exporter) ![Snyk Vulnerabilities for Tado API](https://img.shields.io/snyk/vulnerabilities/github/easimon/tado-exporter/tado-api/pom.xml?label=tado-api) ![Snyk Vulnerabilities for Tado Utils](https://img.shields.io/snyk/vulnerabilities/github/easimon/tado-exporter/tado-util/pom.xml?label=tado-util) ![Snyk Vulnerabilities for Parent POM](https://img.shields.io/snyk/vulnerabilities/github/easimon/tado-exporter/pom.xml?label=parent-pom) |

## Introduction

This is a simple Prometheus exporter for Tado° smart home installations, scraping temperatures, humidity etc. from
the Tado° API and presenting them in a Prometheus compatible format.

Idea taken from [this python variant](https://github.com/vide/tado-exporter).

Since the python exporter failed to scrape the API quite often, and I wanted to experiment with
[Micronaut](https://micronaut.io/) anyway, I decided to rebuild this in Kotlin/Micronaut.

## Building

Either install a JDK (tested with OpenJDK 8) and run `./mvnw package` to create an executable JAR only
(to be found in tado-exporter/target then), or install Docker and use
`docker build . -t tado-exporter:latest` to create a Docker image.

## Usage

Configure username and password as environment variables (`TADO_USERNAME` and `TADO_PASSWORD`, respectively),
and start the exporter. The server then listens on port 8080 (plain HTTP), prometheus metrics are
available at `http://host:8080/prometheus`.

### Configuration

The minimal required configuration is a valid Tado° userame and password.
For complete list of configurable items and their defaults, see the
[application.yml](./tado-exporter/src/main/resources/application.yml)

| Environment variable | Description             | Default | Required |
|----------------------|-------------------------|---------|----------|
| TADO_USERNAME        | Tado° account username  | (none)  | yes |
| TADO_PASSWORD        | Tado° account password  | (none)  | yes |
| TADO_CLIENT_ID       | API OAuth client ID     | [application.yml](./tado-exporter/src/main/resources/application.yml) | no |
| TADO_CLIENT_SECRET   | API OAuth client secret | [application.yml](./tado-exporter/src/main/resources/application.yml) | no |
| TADO_SCOPE           | API OAuth Scope         | [application.yml](./tado-exporter/src/main/resources/application.yml) | no |
| TADO_ZONE_DISCOVERY_INTERVAL | Interval to refresh home and zone (room) information for the given account | 5 min | no |
| TADO_API_CACHE_INTERVAL | Interval to cache API calls for zone and weather information. Should be something like the prometheus step interval, minus the duration it takes to scrape the API once (5-10 secs). | 55 sec | no |
| TADO_PROMETHEUS_STEP_INTERVAL | Prometheus step resulution (how often does prometheus scrape the metrics endpoint) | 60 sec | no |

OAuth client id, secret and scope do not need configuration, since they have defaults found at other projects listed in [References](#references).

### Prometheus scraping config

```yaml
scrape_configs:
  - job_name: tado-exporter
    metrics_path: /prometheus
    scheme: http
    static_configs:
      - targets:
        - tado-exporter:8080
```

### Grafana dashboard

There's a simple [Grafana dashboard](./tado-exporter/src/main/grafana/tado-dashboard.json) you can import.
Since I don't know of a way to do I18N in Grafana, it's in German.

![Grafana Dashboard](./tado-exporter/src/main/grafana/tado-dashboard-screenshot.png "Grafana dashboard")

### Available metrics

Defined in [TadoMeterFactory.kt](tado-exporter/src/main/kotlin/click/dobel/tado/metrics/TadoMeterFactory.kt).
There are also some other metrics (automatically provided by Micronaut framework), but these are the ones
this application is about.

| Name                            | Tags                                   | Cardinality | Description                                                  |
|---------------------------------|----------------------------------------|-------------|--------------------------------------------------------------|
| solar_intensity_percentage      | home_id                                | per home    | solar intensity at your home's location, in percent          |
| temperature_outside_celsius     | home_id                                | per home    | outside temperature at your home's location, in deg. celsius |
| temperature_outside_fahrenheit  | home_id                                | per home    | outside temperature at your home's location, in fahrenheit   |
| temperature_measured_celsius    | home_id, zone_id, zone_name, zone_type | per zone    | measured temperature in this zone, in deg. celsius           |
| temperature_measured_fahrenheit | home_id, zone_id, zone_name, zone_type | per zone    | measured temperature in this zone, in fahrenheit             |
| humidity_measured_percentage    | home_id, zone_id, zone_name, zone_type | per zone    | measured humidity in this zone, in percent                   |
| temperature_set_celsius         | home_id, zone_id, zone_name, zone_type | per zone    | target temperature in this zone, in deg. celsius             |
| temperature_set_fahrenheit      | home_id, zone_id, zone_name, zone_type | per zone    | target temperature in this zone, in fahrenheit               |
| heating_power_percentage        | home_id, zone_id, zone_name, zone_type | per zone    | heating power in this zone, in percent                       |
| is_window_open                  | home_id, zone_id, zone_name, zone_type | per zone    | window open detection (presence of an "openWindow" object in the zone state, translated to 0, 1) |
| is_zone_powered                 | home_id, zone_id, zone_name, zone_type | per zone    | power state (ON, OFF, translated to 0, 1)                    |

### How it works

The exporter discovers all homes attached to a single Tado° account at startup, and refreshes the home and zone
layout every 5 minutes by default. Then, metrics are collected for all discovered zones whenever they are requested
by calling the prometheus metrics endpoint (at most every 55 seconds by default).

## References

- [Tadoº exporter for Prometheus](https://github.com/vide/tado-exporter) by Davide Ferrari
- [OpenHAB Tado° Binding](https://github.com/openhab/openhab-addons/blob/2.5.x/bundles/org.openhab.binding.tado/) for Swagger API definition and OAuth client credentials.
- [Tado API Guide](https://shkspr.mobi/blog/2019/02/tado-api-guide-updated-for-2019/) by Terence Eden
- [The Tado API v2](http://blog.scphillips.com/posts/2017/01/the-tado-api-v2/) by Stephen C. Phillips

## Known Issues and TODOs

- Test coverage is incomplete.
- Does not work on OpenJ9 variants of OpenJDK (Runs into stack overflows in tests, Jackson serialization yields empty Strings)

## Disclaimer

This project is not affiliated with Tado° in any way.
The API used here is not a public one, but reverse engineered by multiple people (see above). While it is used by
quite a few other projects already, it might change incompatibly or vanish at any time.
