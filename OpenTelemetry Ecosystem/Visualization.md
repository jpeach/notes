# Visualization

Grafana is the most common graphing tool, usually showing time series data from Prometheus. In an OpenTelemetry world, we want to be able to combine metrics, logs, profiles, events, and whatever other telemetry signal are available. We want to be able to slice and dice quickly and across arbitrary columns.


## SigNoz
https://signoz.io/

## Superset
https://superset.apache.org/

Supports Clickhouse. Intended for exploratory visualizations.

Making releases. Community seems fairly active.

## Existing Systems Reports
### LinkedIn ThirdEye
https://engineering.linkedin.com/blog/2019/01/introducing-thirdeye--linkedins-business-wide-monitoring-platfor

### Netflix
https://netflixtechblog.com/lessons-from-building-observability-tools-at-netflix-7cfafed6ab17

### Cloudflare
https://blog.cloudflare.com/how-cloudflare-analyzes-1m-dns-queries-per-second/

* Move from Druid to Clickhouse
* Some Superset but ended up with Grafana dashboarding. Generally negative Superset experiences (from 2017 though).