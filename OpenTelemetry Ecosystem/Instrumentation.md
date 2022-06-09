# Instrumentation
## Header trace propagation
Two common standards in use:
* Trace Context -  https://www.w3.org/TR/trace-context/
* B3 https://github.com/openzipkin/b3-propagation

Trace Context will be the standard going forward, so most projects are standardizing on that. B3 is probably legacy at this point.

ClickHouse supports Trace Context, see [here](https://clickhouse.com/docs/en/operations/opentelemetry/).

Envoy does not currently support OpenTelemetry Trace Context, but there is an implementation in [#20281](https://github.com/envoyproxy/envoy/pull/20281) that will probably land for 1.24. Contour support is tracked in [#399](https://github.com/projectcontour/contour/issues/399).

## Agent management
[OpAmp](https://github.com/open-telemetry/opamp-spec) is a protocol for collector agents to connect to a policy server to receive configuration.

OpAmp Go implementation https://github.com/open-telemetry/opamp-go. Looks super early. Is it even possible to write a generic control plane, or is this stuff intrinsically specific to companies?

## Node instrumentation
OpAMP + [osquery](https://osquery.io/) would be interesting. The control plane can push a query and you immediately get monitoring everywhere.
