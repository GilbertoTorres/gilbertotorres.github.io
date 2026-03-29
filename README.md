# BiGGie's Place

A personal homelab blog at [blog.biggie.be](https://blog.biggie.be), built with [Jekyll](https://jekyllrb.com/) and the [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy/) theme, hosted on GitHub Pages.

## What's Here

Blog posts documenting my homelab infrastructure - architecture decisions, service configurations, and lessons learned from running a multi-site lab spanning a home cluster and a Hetzner datacenter.

### Categories

- **Homelab** - High-level architecture and department overviews (networking, security, backup, services, IaC)
- **Setup** - Detailed setup and configuration guides for individual services

### Topics Covered

- Proxmox VE clustering with QDevice quorum
- OPNsense firewall and VLAN segmentation
- WireGuard cross-site tunnels
- Proxmox Backup Server with multi-site replication
- Wazuh SIEM, TheHive, and MISP (SOC stack)
- Mailcow, HestiaCP, Pangolin/Newt, PocketID, Passbolt
- Terraform, Ansible, and GitLab CI/CD pipelines

## Local Development

```bash
bundle install
bundle exec jekyll serve --future
```

## License

This work is published under [MIT](https://github.com/cotes2020/chirpy-starter/blob/master/LICENSE) License.
