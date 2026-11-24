services.yaml:
```
- Infrastructure:
    - Kirk (PVE):
        icon: proxmox.png
        href: https://kirk.g-space.co.za
        siteMonitor: https://kirk.g-space.co.za
        description: Proxmox VE node 1
    - Picard (PVE):
        icon: proxmox.png
        href: https://picard.g-space.co.za
        siteMonitor: https://picard.g-space.co.za
        description: Proxmox VE node 2
    - Enterprise-D:
        icon: truenas.png
        href: https://enterprise-d.g-space.co.za
        siteMonitor: https://enterprise-d.g-space.co.za
        description: TrueNAS storage

- Network & Storage:
    - UISP:
        icon: unifi.png
        href: https://uisp.g-space.co.za
        siteMonitor: https://uisp.g-space.co.za
    - Technitium DNS:
        icon: technitium.png
        href: https://dns.g-space.co.za
        siteMonitor: https://dns.g-space.co.za
    - MinIO Console:
        icon: minio.png
        href: https://s3front.g-space.co.za
        siteMonitor: https://s3front.g-space.co.za

- Apps:
    - Uptime Kuma:
        icon: uptime-kuma.png
        href: https://uptime.g-space.co.za
        siteMonitor: https://uptime.g-space.co.za
    - Forgejo:
        icon: forgejo.png
        href: https://forgejo.g-space.co.za
        siteMonitor: https://forgejo.g-space.co.za
    - Immich:
        icon: immich.png
        href: https://immich.g-space.co.za
        siteMonitor: https://immich.g-space.co.za
    - OpenBao:
        icon: openbao.png
        href: https://bao.g-space.co.za
        siteMonitor: https://bao.g-space.co.za
    - Coroot:
        icon: /icons/coroot.png
        href: https://coroot.g-space.co.za
        siteMonitor: https://coroot.g-space.co.za
    - Argo CD:
        icon: argo-cd.png
        href: https://argo.g-space.co.za
        siteMonitor: https://argo.g-space.co.za
        description: GitOps continuous delivery
```

settings.yaml
```
title: G-Space Dashboard

layout:
  Infrastructure:
    style: row
    columns: 3
  Network & Storage:
    style: row
    columns: 3
  Apps:
    style: row
    columns: 3

```

bookmarks.yaml
```
- Developer:
    - GitHub:
        - icon: github.png
          href: https://github.com/GarthvandenBerg/Talos-Proxmox-Homelab

- Social:
    - LinkedIn:
        - icon: linkedin.png
          href: https://linkedin.com/in/garthvandenberg

- Entertainment:
    - YouTube:
        - icon: youtube.png
          href: https://www.youtube.com

```