Envoy doesn't load binary filters at runtime (WASM is a different story), or (I believe) have a stable internal API. The recommeded way to build exernal filters is to create a new Bazel workspace that builds both your Envoy filter and your version of Envoy.

Everyone that does this just copies the Envoy example filter [repository](https://github.com/envoyproxy/envoy-filter-example).

The [documentation](https://github.com/envoyproxy/envoy/blob/main/bazel/README.md#enabling-and-disabling-extensions) for configuring an Envoy workspace to select which filters are compiled in does work.

## Out of tree filters

| Filter | Repository |
| --- | --- |
 PagePpeed | https://github.com/apache/incubator-pagespeed-mod
 ModSecurity | https://github.com/octarinesec/ModSecurity-envoy |
 
 I haven't found a workspace that tries to bundle multiple external filters into the same Envoy build, but I imagine it's tricky.