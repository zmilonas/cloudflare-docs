---
pcx_content_type: how-to
title: Connect private networks
weight: 1
layout: single

---

# Connect private networks

Creating a private network has two components: the server and the client. The server's infrastructure (whether that is a single application, multiple applications, or a network segment) is connected to Cloudflare's edge by Cloudflare Tunnel. This is done by running the `cloudflared` daemon on the server. Simply put, Cloudflare Tunnel is what connects your network to Cloudflare. On the client side, end users connect to Cloudflare's edge using the Cloudflare WARP agent. This agent can be rolled out to your entire organization in just a few minutes using your in-house MDM tooling.

To connect a private network to Cloudflare’s edge, follow the guide below. You can also check out our [tutorial](/cloudflare-one/tutorials/warp-to-tunnel/).

## Prerequisites

{{<render file="_warp-to-tunnel-client.md">}}

## 1. Connect the server to Cloudflare

To connect your infrastructure with Cloudflare Tunnel:

1. Create a Cloudflare Tunnel for your server by following our [dashboard setup guide](/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide/remote/). You can skip the connect an application step and go straight to connecting a network.

2. In the **Private Networks** tab for the tunnel, enter the IP/CIDR range of your private network (for example `10.0.0.0/8`). This makes the WARP client aware that any requests to this IP range need to be routed to your new tunnel.

## 2. (Recommended) Filter network traffic with Gateway

By default, all WARP devices enrolled in your Zero Trust organization can connect to your private network through Tunnel. You can configure Gateway to inspect your network traffic and either block or allow access based on user identity.

### Enable the Gateway proxy

1. In the Zero Trust dashboard, go to **Settings** > **Network**.
2. Enable **Proxy** for TCP.

This will tell Cloudflare to begin proxying any traffic from enrolled devices, except the traffic excluded in your [split tunnel settings](#route-private-network-ips-through-gateway).

### Route private network IPs through Gateway

By default, WARP automatically excludes some IP addresses from Gateway visibility as part of its [Split Tunnel feature](/cloudflare-one/connections/connect-devices/warp/configure-warp/route-traffic/split-tunnels/). For example, WARP automatically excludes RFC 1918 IP addresses such as `10.0.0.0/8`, which are IP addresses typically used in private networks and not reachable from the Internet. You will need to make sure that traffic to the IP/CIDR you are associating with your private network are sent to Gateway for filtering.

To configure Split Tunnels settings:

1. Check whether your [Split Tunnels mode](/cloudflare-one/connections/connect-devices/warp/configure-warp/route-traffic/split-tunnels/#set-up-split-tunnels) is set to **Exclude** or **Include** mode.
2. [Edit the split tunnel entries](/cloudflare-one/connections/connect-devices/warp/configure-warp/route-traffic/split-tunnels/#add-an-ip-address):
    - If you are using **Exclude** mode, the IP ranges you see listed are those that Cloudflare excludes from WARP encryption. If your network's IP/CIDR range is listed on this page, delete it.
    - If you are using **Include** mode, the IP ranges you see listed are the only ones Cloudflare is encrypting through WARP. Add your network's IP/CIDR range to the list.

### Create Zero Trust policies

You can create Zero Trust policies to manage access to specific applications on your network.

1. Go to **Access** > **Applications** > **Add an application**.
2. Select **Private Network**.
3. Name your application.
4. For **Application type**, select _Destination IP_.
5. For **Value**, enter the IP address for your application (for example, `10.128.0.7`).
{{<Aside type="note">}}
If you would like to create a policy for an IP/CIDR range instead of a specific IP address, you can build a [Gateway Network policy](/cloudflare-one/policies/filtering/network-policies/) using the **Destination IP** selector.
{{</Aside>}}

6. Configure your [App Launcher](/cloudflare-one/applications/app-launcher/) visibility and logo.
7. Select **Next**. You will see two auto-generated Gateway Network policies: one that allows access to the destination IP and another that blocks access.
8. Modify the policies to include additional identity-based conditions. For example:

    - **Policy 1**
    | Action | Selector | Operator | Value |
    |--|--|--|--|
    | Allow  | Destination IP |in|`10.128.0.7` |
    |        |User email| Matches regex| `*@example.com`|

    - **Policy 2**
    | Block  | Selector | Operator | Value |
    |--|--|--|--|
    | Block |  Destination IP |in|`10.128.0.7` |

    Access rules are evaluated in order, so a user with an email ending in @example.com will be able to access `10.128.0.7` while all others will be blocked. For more information on building network policies, refer to our [dedicated documentation](/cloudflare-one/policies/filtering/network-policies/).

9. Select **Add application**.

Your application will appear on the **Applications** page.

## 3. Connect as a user

End users can now reach HTTP or TCP-based services on your network by navigating to any IP address in the range you have specified.

### Troubleshooting

#### Device configuration

To check that their device is properly configured, the user can visit `https://help.teams.cloudflare.com/` to ensure that:

- The page returns **Your network is fully protected**.
- In **HTTP filtering**, both **WARP** and **Gateway Proxy** are enabled.
- The **Team name** matches the Zero Trust organization from which you created the tunnel.

#### Router configuration

Check the local IP address of the device and ensure that it does not fall within the IP/CIDR range of your private network. For example, some home routers will make DHCP assignments in the `10.0.0.0/24` range, which overlaps with the `10.0.0.0/8` range used by most corporate private networks. When a user's home network shares the same IP addresses as the routes in your tunnel, their device will be unable to connect to your application.

To resolve the IP conflict, you can either:

- Reconfigure the user's router to use a non-overlapping IP range. Compatible routers typically use `192.168.1.0/24`, `192.168.0.0/24` or `172.16.0.0/24`.

- Tighten the IP range in your tunnel configuration to exclude the `10.0.0.0/24` range. This will only work if your private network does not have any hosts within `10.0.0.0/24`.
