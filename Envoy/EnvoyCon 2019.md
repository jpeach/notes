# EnvoyCon 2019

https://envoycon2019.sched.com/event/Uxv2/how-spotify-migrated-ingress-http-systems-to-envoy-erik-lindblad-kateryna-nezdoli-alex-sundstrom-spotify

- Moving from legacy proprietary protocols to standards (WSS, HTTP)
- HTTPS balancer was NGINX + HAProxy
    - Configured with Puppet (YAML templating w/ hiera)
    - Service discovery w/ Puppet
- Unified perimeter architecture
- Envoy chosen
    - Moving up stack (no internal infra)
    - xDS
    - open source + community
- Use ext_authz

https://envoycon2019.sched.com/event/UxvS/envoy-mobile-in-depth-from-server-to-multi-platform-library-jose-nino-michael-schore-lyft

- Use cases
    - Consistent metrics from user -> edge -> DC
    - Improved common tooling
- Envoy as a library
    - Bazel building for multiple SDK targets
    - Layered
        - Platform -> Binding -> Envoy Core
    - API as similar as possible across languages
    - Plan for API-compatibility with other libraries

https://envoycon2019.sched.com/event/UxvY/envoy-namespaces-operating-an-envoy-based-service-mesh-at-a-fraction-of-the-cost-thomas-graf-cilium-isovalent

- Network control plane with central policy service subject to DoS
- Perf impact on control plane (network + CPU), churn matters
    - Mesh problems when all envoy sidecars hit control plane (1 envoy per pod)
- Move from envoy-per-pod to envoy-per-node
    - i.e. similar config as reverse proxy
- eBPF to route thru the node Envoy
    - Hooks connect(2)
- Envoy namespaces for resource management
- Need to teach Isio about per-node proxy

https://envoycon2019.sched.com/event/WjiG/managing-tens-of-thousands-of-envoy-how-we-do-it-shubha-rao-aws

- AWS App Mesh
- Envoy routing mesh computed from PoV of each service
- Mesh computation -> Database -> xDS management server
- Integrated with AWS certificate management

https://envoycon2019.sched.com/event/Uxve/spanning-the-globe-with-envoy-at-stripe-dylan-carney-stripe

- Get to the point!
- Rambling about Stripe having network PoPs
- They use Envoy to proxy from edge to DC
- xDS control plane
- They like HTTP/2 a lot
- BBR + fq packext scheduling
- Enable TCP keepalive

https://envoycon2019.sched.com/event/Uxvn/from-microbenchmarks-to-http2-load-testing-5-performance-tools-and-techniques-to-improve-envoy-scalability-joshua-marantz-google-otto-van-der-schaaf-we-amp-bv

- Scalability issues in different configs (see slides for examples)
- PERF macro API for (see vmware kstats, https://dl.acm.org/citation.cfm?id=1899945)
- Loadgen tools
    - Eval narrowed to wrk2 and fortio
    - New loadgen built on envoy core (nighthawk)
- Using fuzz approach for loadgen profiles
    - Work thru surface to find perf problems

https://envoycon2019.sched.com/event/Uxvz/dynamic-request-routing-with-envoy-ben-plotnick-cruise

- Reroute requests by twiddling headers in the request
- Uses "original_dst" cluster x-envoy-original-dst

https://envoycon2019.sched.com/event/Uxw0/building-low-latency-topologies-with-envoy-john-howard-google-snow-petterson-square-liam-white-tetrate

- Minimize cost & latency
    - Keep traffix local but fail over to remote AZ
- Zone aware LB only works with static bootstrap clusters
- Weights doesn't really work for failover
- Spill traffic over priority bands
- Can control failover with subset balancer
- Failover -> long connection setup -> high request latency