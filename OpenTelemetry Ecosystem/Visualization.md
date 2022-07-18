# Visualization

Grafana is the most common graphing tool, usually showing time series data from Prometheus. In an OpenTelemetry world, we want to be able to combine metrics, logs, profiles, events, and whatever other telemetry signal are available. We want to be able to slice and dice quickly and across arbitrary columns.

## SigNoz
https://signoz.io/

## Superset
https://superset.apache.org/

Supports Clickhouse. Intended for exploratory visualizations.

Making releases. Community seems fairly active.

## Snorkel
https://snorkel.logv.org/

Based on Scuba experiences.

## Kibana
https://www.elastic.co/kibana/

Obviously.

## Loki
https://grafana.com/oss/loki/

Log visualizer from Grafana. Indexes tags/metadata not full text, so I suppose benefits more from structured logging. Some OpenTelemetry support coming in [#5363](https://github.com/grafana/loki/pull/5363).

## Existing Systems Reports
### LinkedIn ThirdEye
https://engineering.linkedin.com/blog/2019/01/introducing-thirdeye--linkedins-business-wide-monitoring-platfor

### Netflix
https://netflixtechblog.com/lessons-from-building-observability-tools-at-netflix-7cfafed6ab17

### Cloudflare
https://blog.cloudflare.com/how-cloudflare-analyzes-1m-dns-queries-per-second/

* Move from Druid to Clickhouse
* Some Superset but ended up with Grafana dashboarding. Generally negative Superset experiences (from 2017 though).
