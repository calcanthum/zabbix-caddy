# zabbix-caddy

Zabbix monitoring template for Caddy.

Forked from [lonelyidea/zabbix-caddy](https://github.com/lonelyidea/zabbix-caddy). See the [original blog post](https://lonelyidea.com/posts/caddy-monitoring/) for background.

## Changes from upstream

The original template uses an `HTTP_AGENT` item, which means the Zabbix Server fetches the metrics endpoint directly over the network. This fork switches the master item to `Zabbix agent (active)`, so the agent running on the monitored host scrapes `localhost` instead. This keeps the metrics endpoint safely bound to `127.0.0.1` with no need to expose port 8080 beyond loopback.

A trigger prototype is also included that fires when any reverse proxy upstream goes unhealthy.

## Requirements

- Zabbix 7.2+
- Zabbix Agent 2 installed on the Caddy host
- `curl` available on the Caddy host

## Caddy configuration

Enable metrics collection and expose the endpoint on localhost in your `Caddyfile`:

```
{
    metrics
    servers :8080 {
        protocols h1 h2
    }
}

http://127.0.0.1:8080 {
    metrics /metrics
}
```

Then reload Caddy:

```bash
caddy reload --config /etc/caddy/Caddyfile
```

Verify it works:

```bash
curl http://127.0.0.1:8080/metrics
```

## Agent configuration

Create a UserParameter file on the monitored host:

```bash
# /etc/zabbix/zabbix_agent2.d/caddy.conf
UserParameter=caddy.raw,curl -s http://127.0.0.1:8080/metrics
```

Then restart the agent:

```bash
systemctl restart zabbix-agent2
```

## Zabbix setup

Import `zabbix-caddy.yaml` via **Data collection → Templates → Import** and apply the `Caddy` template to your host. No macros need to be configured.

## Included trigger prototype

| Name | Severity | Condition |
|---|---|---|
| Upstream {#UPSTREAM} is unhealthy on {HOST.NAME} | High | upstream health = 0 |

The trigger is instantiated automatically for each discovered upstream.
