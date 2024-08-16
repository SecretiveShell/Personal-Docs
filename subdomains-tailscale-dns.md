# Adding subdomains to tailscale MagicDNS

**TLDR** magicDNS does not support this you need a custom DNS resolver

This aims to allow you to set subdomains for hosts on the tailscale network, for example, `http://www.server-pc:80` or `http://homeassistant.server-pc:80`, instead of being stuck with just using different ports like `http://server-pc:8080`.

## setup some test services

to demo this setup here is an example docker compose file that sets up an Apache HTTP server and a traefik proxy.

```yaml
services:
  traefik:
    image: traefik:v2.10
    command:
      - "--api.insecure=true" # Expose Traefik's dashboard
      - "--providers.docker=true" # Enable Docker provider for Traefik
      - "--entrypoints.web.address=:80" # Set entry point to listen on port 80
    ports:
      - "80:80"  # Expose port 80
      - "8080:8080" # Expose Traefik dashboard on port 8080
    networks:
      - web
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock # Needed for Traefik to access Docker API

  httpd:
    image: httpd:latest
    networks:
      - web
    labels:
      - "traefik.http.routers.httpd-router.rule=Host(`www.<YOUR HOST>.internal`)" # Route traffic to httpd service
      - "traefik.http.services.httpd-service.loadbalancer.server.port=80" # Define the port to route to

networks:
  web:
    external: false
```
Make sure to set `www.<YOUR HOST>.internal` to the name of your host

## Install a DNS server

I used pihole because it's lightweight, you don't need the AdBlock functions, only the ability to set local DNS records and CNAME entries. If you already have a DNS server setup you can skip this step. Here is an example docker-compose file with DHCP stuff removed.

```yaml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8081:80/tcp"
    environment:
      TZ: 'America/Chicago'
      WEBPASSWORD: 'password'
    # Volumes store your data between container upgrades
    volumes:
      - './etc-pihole:/etc/pihole'
      - './etc-dnsmasq.d:/etc/dnsmasq.d'
    restart: unless-stopped
```
Make sure to use a better password than `password`. Port 80 is mapped to 8081 to avoid collisions with traefik.

## Setup tailscale to recognize your DNS server

Go to the [Tailscale control panel](https://login.tailscale.com/admin/dns) and add the tailscale ip address of your server as a DNS resolver for the `.internal` domain name. Enable split DNS; This will mean that only `*.internal` domain names are resolved by the pihole server. You can copy tailscale IP addresses from the tray icon.

Under `Search Domains` add `internal`. This means that any names failed to resolve on the `*.ts.net` domain are tried on the `.internal` domain. Basically you don't need to write `.internal` on the end of your URLs.

## Add DNS records

Add a DNS record for your server at `<YOUR HOST>.internal` with the IP address as your tailscale IP.

Add a CNAME record for `<SUBDOMAIN>.<YOUR HOST>.internal` pointing to `<YOUR HOST>.internal` for each subdomain you want to use. The example uses `www` as the subdomain. There might be a way to use wildcard CNAME records but I am not sure how to do this in pihole.

## Test the site

You should now be able to access `www.<YOUR HOST>` in a browser and see the example homepage of the httpd web server.