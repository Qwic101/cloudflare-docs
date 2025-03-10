---
_build:
  publishResources: false
  render: never
  list: never
inputParameters: healthCheckFrequencyURL;;productName;;onboardingURL;;configureTunnelEndpointsURL;;urlChangeHealthCheckType
---

# Tunnel health checks

A tunnel health check probe contains an [ICMP (Internet Control Message Protocol)](https://www.cloudflare.com/learning/ddos/glossary/internet-control-message-protocol-icmp/) reply packet that originates from an IP address on the origin side of the tunnel and whose destination address is a public Cloudflare IP.

Cloudflare encapsulates the ICMP reply packet and sends the probe across the tunnel to the origin. When the probe reaches the origin router, the router decapsulates the ICMP reply and forwards it to the specified destination IP. The probe is successful when Cloudflare receives the reply.

{{<render file="_icmp-mfirewall.md">}}

Every Cloudflare data center configured to process your traffic sends tunnel health check probes. The rate at which these health check probes are sent varies based on tunnel and location. This rate can also be tuned up or down on a per tunnel basis by modifying the `health_check` rate of a tunnel [with the API]($1).

When a probe attempt fails for a [healthy](#health-state-and-prioritization) tunnel, each server detecting the failure quickly probes up to two more times to obtain an accurate result. We also do the same if a tunnel has been down and probes start returning success. Because Cloudflare global network servers send probes up to every second, you can expect your network to receive several hundred health check packets per second — each Cloudflare data center will only send one health check packet as part of a probe. This represents a relatively trivial amount of traffic.

{{<Aside type="note" header="Note">}}

To avoid control plane policies enforced by the origin network, tunnel health checks use an encapsulated ICMP reply instead of an ICMP echo request. To use echo request packets, change your health check type to **Request** in your tunnels. Refer to [Configure tunnel endpoints]($5) to learn how to change this setting.

{{</Aside>}}

{{<details header="Wireshark example of health check packets">}}

![Wireshark example for tunnel health checks with ICMP reply packet](/images/magic-transit/tunnel-health-check-packets.png)

{{</details>}}

## Health state and prioritization

There are three tunnel health states: healthy, degraded, and down.

Healthy tunnels are preferred to degraded tunnels, and degraded tunnels are preferred to those that are down.

$2 steers traffic to tunnels based on priorities you set when you [assign tunnel route priorities during onboarding]($3). Tunnel routes with lower values have priority over those with higher values.

{{<Aside type="note" header="Note">}}

Cloudflare global network servers may be able to reach the origin infrastructure from some locations at a given time but not others. This occurs because Cloudflare does not synchronize health checks among global network servers and because the Internet is not homogeneous.

As a result, tunnel health may be in different states in different parts of the world at the same time. In the example from the previous paragraph, both tunnels could receive traffic simultaneously, even though Tunnel 1 has priority over Tunnel 2.

{{</Aside>}}

## Tunnel state determination

### Degraded

- When at least 0.1% or more of tunnel health checks fail in the previous five minutes (with at least two failures), $2 considers the link lossy and sets the tunnel state to degraded (assuming the tunnel is not down).
- $2 requires two failures so that a single lost packet does not trigger a penalty.
- $2 then immediately sets the tunnel status to degraded and applies a priority penalty.

### Down

- When all health checks of at least three samples in the last one second fail, $2 immediately transitions the tunnel from healthy or degraded to down, and applies a priority penalty to routes through that tunnel.
- A down state determination takes precedence over a degraded state determination. This means that a tunnel can only be one of the following: down, degraded, or healthy.

When $2 identifies a route that is not healthy, it applies these penalties:

- **Degraded**: Add `500,000` to priority.
- **Down**: Add `1,000,000` to priority.

The values for failure penalties are intentionally extreme so that they always exceed the priority values assigned during [routing configuration]($3).

Applying a penalty instead of removing the route altogether preserves redundancy and maintains options for customers with only one tunnel. Penalties also support the case when multiple tunnels are unhealthy.

### Cloudflare data centers and tunnels

In the event a Cloudflare data center is down, Cloudflare’s global network does not advertise your prefixes, and your packets are routed to the next closest data center. To check the system status for Cloudflare’s global network and dashboard, refer to [Cloudflare System Status](https://www.cloudflarestatus.com/).

## Recovery

Once a tunnel is in the down state, global network servers continue to emit probes according to the cadence described above. When a probe returns healthy, the global network server that received the healthy packet immediately sends two more probes. If the two probes return healthy, $2 sets the tunnel status to degraded (as three consecutive successful probes no longer satisfy the condition for a down state).

Tunnels in a degraded state transition to healthy when the failure rate for the previous 30 probes is less than 0.1%. This transition may take up to 30 minutes.

$2’s tunnel health check system allows a tunnel to quickly transition from healthy to degraded or down, but tunnel transition occurs slowly from degraded or down to healthy. This scenario is referred to as hysteresis — which is when a system’s output depends on its history of past inputs — and dampens changes to tunnel routing caused by flapping and other intermittent network failures.

{{<Aside type="note" header="Note">}}
Cloudflare always attempts to send traffic over available tunnel routes with the highest priority (lowest route value), even when all configured tunnels are in an unhealthy state.
{{</Aside>}}

## Example

Consider two tunnels and their associated routing priorities. Remember that lower route values have priority.

- Tunnel 1, route priority `100`
- Tunnel 2, route priority `200`

When both tunnels are in a healthy state, routing priority directs traffic exclusively to Tunnel 1 because its route priority of 100 beats that of Tunnel 2. Tunnel 2 does not receive any traffic, except for tunnel health check probes. Endpoint health checks only flow over Tunnel 1 to their destination inside the origin network.

### Failure response

If the link between Tunnel 1 and Cloudflare becomes unusable, Cloudflare global network servers discover the failure on their next health check probe, and immediately issue two more probes (assuming the tunnel was initially healthy).

When a global network server does not receive the proper ICMP reply packets from these two additional probes, the global network server labels Tunnel 1 as down, and downgrades Tunnel 1 priority to `1,000,100`. The priority then shifts to Tunnel 2, and $2 immediately steers packets arriving at that global network server to Tunnel 2.

### Recovery response

Suppose the connectivity issue that set Tunnel 1 health to down becomes resolved. At the next health check interval, the issuing global network server receives a successful probe and immediately sends two more probes to validate tunnel health.

When all three probes return successfully, $2 transitions the tunnel from down to degraded. As part of this transition, Cloudflare reduces the priority penalty for that route so that its priority becomes `500,100`. Because Tunnel 2 has a priority of `200`, traffic continues to flow over Tunnel 2.

Global network servers will continue probing Tunnel 1. When the health check failure rate drops below 0.1% for a five minute period, $2 sets tunnel status to healthy. Tunnel 1’s routing priority is fully restored to `100`, and traffic steering returns the data flow to Tunnel 1.

## Types of health checks

### Endpoint health checks

Endpoint health checks evaluate connectivity from Cloudflare distributed data centers to your origin network. Designed to provide a broad picture of Internet health, endpoint probes flow over available tunnels and do not inform tunnel selection or steering logic.

Cloudflare global network servers issue endpoint health checks outside of customer network namespaces and typically target endpoints beyond the tunnel-terminating border router.

During onboarding, you specify IP addresses to configure endpoint health checks.

### ​Tunnel health checks

Tunnel health checks monitor the status of the {{<glossary-tooltip term_id="GRE tunnel">}}Generic Routing Encapsulation (GRE){{</glossary-tooltip>}} and {{<glossary-tooltip term_id="IPsec tunnel">}}IPsec{{</glossary-tooltip>}} tunnels that route traffic from Cloudflare to your origin network. $2 relies on health checks to steer traffic to the best available routes.

During onboarding, you [specify the tunnel endpoints]($4) the tunnel probes originating from Cloudflare’s global network will target.

Tunnel health check results are exposed [via API](/analytics/graphql-api/tutorials/querying-magic-transit-tunnel-healthcheck-results/). These results are aggregated from individual health check results done on Cloudflare servers.

#### Bidirectional health checks

To check for tunnel health, Cloudflare sends packets in the form of ICMP echo replies. These packets are destined for the Cloudflare side of the interface address field set on the IPsec tunnel, and are sourced from the client of the tunnel. For example, if the interface address is `10.100.0.8/31`, then the packet will be destined for `10.100.0.9` and sourced from `10.100.0.8`.

Note that the interface address field is always a `/30` or `/31` CIDR range. In the case of a `/31` range, the IP provided will be the Cloudflare side, whereas the other will be the client side. For example, if the interface address is `10.100.0.8/31`, `10.100.0.8` is the Cloudflare side, and `10.100.0.9` is the client side. In case of a `/30` range, the IP provided will be the Cloudflare side whereas the other IP (excluding the broadcast and network identifier) will be the client side. For example, if the interface address is `10.100.0.9/30`, `10.100.0.9` will be the Cloudflare side and `10.100.0.10` will be the client side.

These packets will flow to and from Cloudflare over the IPsec tunnels you have configured to provide full visibility into the traffic path between our network and your sites. You will need to configure traffic selectors to accept the health check packets.

#### Legacy health checks system

{{<render file="_legacy-hc-system.md" >}}