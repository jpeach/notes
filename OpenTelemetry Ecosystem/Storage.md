# Storage
No clear consensus on how to store large quantities of OpenTelemetry signals. Probably there is no best system - have to choose one and suffer through it.

Comparison of Clickhouse, Druid and Pinot:
https://leventov.medium.com/comparison-of-the-open-source-olap-systems-for-big-data-clickhouse-druid-and-pinot-8e042a5ed1c7

Netflix realtime data infrastructure:
https://zhenzhongxu.com/the-four-innovation-phases-of-netflixs-trillions-scale-real-time-data-infrastructure-2370938d7f01

Slack KalDB: https://github.com/slackhq/kaldb

Grafana Tempo: https://github.com/grafana/tempo

FrostDB (formerly ArcticDB): https://www.polarsignals.com/blog/posts/2022/05/04/introducing-arcticdb/

## Related readings
* [Why Observability Requires a Distributed Column Store](https://www.honeycomb.io/blog/why-observability-requires-distributed-column-store/)