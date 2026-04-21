---
description: deploy skill for RobboHome automation.
---

# Skill: Deploy / Release

## Patch release (bug fix)
  make bump-patch
  git push && git push --tags

## Minor release (new feature)
  make bump-minor
  git push && git push --tags

## Check deployment logs
  make logs

## Roll back to previous version
  ssh robbohome-server
  cd ~/data/PROJECT_NAME
  VERSION=1.x.x docker compose -f docker-compose.prod.yml up -d

## Verify live
  curl https://SUBDOMAIN.robbohome.com/health

## GUI options available on server
  Start GNOME:   sudo systemctl start graphical.target
  Stop GNOME:    sudo systemctl isolate multi-user.target
  Cockpit:       http://svr002:9090
  Portainer:     http://svr002:9000
