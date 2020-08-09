+++
title = "Dynamic DNS records with Scaleway DNS"
slug = "dynamic-dns-records-scaleway-dns"
date = "2020-08-09T12:06:00+02:00"
lastmod = "2020-08-09T12:06:00+02:00"
categories = ["automation"]
tags = ["ddns", "scaleway"]
+++

Dynamic DNS (DDNS) is a service that keeps the DNS updated with the correct IP address, even if that IP address is being updated. It is particularly interesting for places where the IP address is dynamic and also needs to be tracked with a DNS record (for remote connection for instance). This situation is particularly common as some ISPs do not commit to static IP addresses.

[Scaleway](https://www.scaleway.com/) offers a DNS service with an API available. It gives the possibility to host DNS zones & records and can be configured manually from the web console or through the API. The developers documentation for the API is available [here](https://developers.scaleway.com/en/products/domain/api/). Both nameservers (`ns0.scaleway.com` and `ns1.scaleway.com`) are hosted in France.

{{< alert "warning" "exclamation-triangle" >}}
Scaleway DNS is currently in public beta with no service guarantee. Although the service is stable, it should not be used in production environments yet.
{{< /alert >}}

This article explains how to register dynamic DNS records with Scaleway DNS using `scaleway-ddns`, a software specially designed for this purpose. It's available on Linux, macOS and Windows.

The first step is to download [the latest version on GitHub](https://github.com/aerialls/scaleway-ddns/releases).

```bash
$ curl -sLO https://github.com/aerialls/scaleway-ddns/releases/download/v0.1.2/scaleway-ddns_0.1.2_linux_amd64.tar.gz
$ tar xf scaleway-ddns_0.1.2_linux_amd64.tar.gz
```

{{< alert "success" "server" >}}
The source code is available on GitHub at [aerialls/scaleway-ddns](https://github.com/aerialls/scaleway-ddns).
{{< /alert >}}

The software is designed to run in the background to detect new IP addresses at regular intervals and automatically perform DNS changes if necessary.

Move the downloaded file to the `/usr/local/bin` directory and make sure the binary can be executed.

```bash
$ mv scaleway-ddns_0.1.2_linux_amd64/scaleway-ddns /usr/local/bin
$ chmod a+x /usr/local/bin/scaleway-ddns
```

In order to work, the software needs a YAML config file which contains the credentials for the Scaleway API, the domain information and also some optional parameters. Create an empty `scaleway-ddns.yml` file. Credentials for the Scaleway API should be under the `scaleway` key as shown in the following example.

```yaml
scaleway:
  organization_id: b84d38dd-48b2-acde-0af89ff0f24e
  access_key: SCW7XVT60TF2ECM2EK0A
  secret_key: 418c21eb-ec4c-9df0-e4b5029d2d55
```

The `organization_id` can be found under the **Credentials** page of the **Organization Account** section on the [Scaleway console](https://console.scaleway.com/account/organization/credentials). Generate a new token to obtain the `access_key` and the `secret_key`.

![Scaleway API Tokens](/images/scaleway/api-tokens.png)

The next step is to make sure that the domain is registered on Scaleway DNS. The [web console](https://console.scaleway.com/domains/external) or the [public API](https://developers.scaleway.com/en/products/domain/api/) can be used for that. For now, it's only possible to register top domain (for instance `aerialls.io` and not `ddns.aerialls.io`). Follow the workflow to add the domain. In the end, it should be visible on the web console.

![Scaleway Domains](/images/scaleway/domains.png)

DNS domain information can now be added in the configuration file.

```yaml
domain:
  name: madalynn.net
  record: ddns
  ttl: 60
```

The previous configuration will generate a DNS record `ddns.madalynn.net` with a TTL of 60 seconds. IPv4 and IPv6 are both supported. By default, IPv6 is disabled and need to be enabled manually.

For each entry, it's possible to select the HTTP endpoint that will provide the public IP address.

```yaml
ipv4:
  enabled: true
  url: https://api-ipv4.ip.sb/ip

ipv6:
  enabled: true
  url: https://api-ipv6.ip.sb/ip
```

{{< alert "info" "keyboard" >}}
HTTP endpoints should directly return the IP address with a plain text response.
{{< /alert >}}

It's also possible to specify the interval between two checks with the `interval` key (in seconds). The default value is set at 60 seconds. The following configuration will check every 5 minutes if the IP has changed. Note that it's not possible to go lower than 60 seconds.

```yaml
interval: 300
```

## Notification

It's also possible to send a Telegram notification when a change has been detected.

```yaml
telegram:
  enabled: true
  token: 123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11
  chat_id: 74423
  template: DNS record *{{ .RecordName }}.{{ .Domain }}* has been updated
```

The message (in Markdown) can be configured with the `template` parameter. Multiple variables are available to fully personalize the message. The full list can be found in the README file in the [GitHub repository](https://github.com/aerialls/scaleway-ddns#telegram).

## Systemd

A systemd service can be used to automatically launch `scaleway-ddns` at startup. Create a service file at `/etc/systemd/system/scaleway-ddns.service` with the following content.

```text
[Unit]
Description=Dynamic DNS service based on Scaleway DNS.
Wants=network-online.target
After=network-online.target

[Service]
User=scaleway-ddns
Group=scaleway-ddns
Type=simple
ExecStart=/usr/local/bin/scaleway-ddns --config /etc/scaleway-ddns/scaleway-ddns.yml

[Install]
WantedBy=multi-user.target
```

```bash
$ sudo systemctl start scaleway-ddns
$ sudo systemctl enable scaleway-ddns
```

The previous example will work only if the user and group `scaleway-ddns` exists and the config file is located inside the `/etc/scaleway-ddns` directory.

## Full configuration

```yaml
scaleway:
  organization_id: b84d38dd-48b2-acde-0af89ff0f24e
  access_key: SCW7XVT60TF2ECM2EK0A
  secret_key: 418c21eb-ec4c-9df0-e4b5029d2d55

domain:
  name: madalynn.net
  record: ddns
  ttl: 60

interval: 300

ipv4:
  enabled: true
  url: https://api-ipv4.ip.sb/ip

ipv6:
  enabled: true
  url: https://api-ipv6.ip.sb/ip

telegram:
  enabled: true
  token: 123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11
  chat_id: 74423
  template: DNS record *{{ .RecordName }}.{{ .Domain }}* has been updated
```

The full configuration is available on GitHub in the [aerialls/scaleway-ddns](https://github.com/aerialls/scaleway-ddns) repository.
