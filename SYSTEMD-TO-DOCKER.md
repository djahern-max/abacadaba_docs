# systemd to Docker Compose

A translation table for the abacadaba Droplet, written for someone who already
thinks in systemd.

Assumes this alias is in `~/.bashrc`:

    alias dc='docker compose -f /srv/abacadaba/docker-compose.prod.yml'

---

## The one conceptual difference

**Your code lives in the image, not on disk.**

With systemd, `git pull` changes the files the service actually reads, so
`systemctl restart` picks them up. With Docker, `git pull` changes files on the
host, but the container runs a built image containing a copy of the code from
build time. Restarting runs the same old image again.

This means `dc restart api` after a pull is almost always wrong. Nothing errors.
The container comes back healthy. Your change simply is not there. It is the
most common Docker mistake for people arriving from systemd.

Rebuild instead: `dc up -d --build api`

---

## The mapping

| systemd | Docker Compose |
|---|---|
| `systemctl status myapp` | `dc ps` |
| `systemctl restart myapp` | `dc up -d --build api` |
| `systemctl stop myapp` | `dc stop api` |
| `systemctl start myapp` | `dc up -d api` |
| `systemctl reload myapp` | `dc up -d --force-recreate api` |
| `journalctl -u myapp -f` | `dc logs -f api` |
| `journalctl -u myapp -n 50` | `dc logs --tail 50 api` |
| `journalctl -u myapp --since 10m` | `dc logs --since 10m api` |
| `systemctl enable myapp` | `restart: unless-stopped` (already set) |
| `systemctl is-enabled myapp` | `dc config \| grep restart` |

`dc restart api` does exist, and is correct for a genuine restart with no code
change: clearing stuck state, recovering from a hung worker.

Use `--force-recreate` after editing `.env`. Environment variables are read when
the container is created, so a plain restart will not pick them up.

---

## Deploying a change

    cd /srv/abacadaba
    git pull
    dc run --rm api alembic upgrade head
    dc up -d --build api
    dc logs --tail 30 api

Migrations run first, in a throwaway container. A failure there stops the
deploy before the running api is replaced, rather than leaving the api serving
against a schema it does not match. This ordering is the whole reason the deploy
is not a single command.

The rebuild is fast after the first one. Docker caches layers, and because
`requirements.txt` is copied before the source, a code-only change skips the pip
install entirely.

Frontend changes do not involve Docker. Build on your laptop, rsync `dist/` to
`/var/www/abacadaba/`. nginx serves those files directly, and the Droplet has no
Node by design.

---

## systemd has not gone away

It still manages the things around the application:

    sudo systemctl status docker
    sudo systemctl status nginx
    sudo systemctl reload nginx
    sudo systemctl list-timers | grep certbot
    sudo journalctl -u nginx -n 50
    sudo journalctl -u ssh -n 50

`restart: unless-stopped` in the compose file is the equivalent of
`systemctl enable`: Docker's own unit is enabled, and Docker brings the
containers back. Worth verifying once, rather than assuming:

    sudo systemctl is-enabled docker    # want: enabled
    sudo reboot
    # wait ~30s, reconnect
    dc ps

---

## Where to look when something breaks

A 502 in the browser means nginx is fine and answered; whatever it proxied to
did not. Go to the container.

    dc ps                        # running? healthy?
    dc logs --tail 50 api        # what did it say before it died?

A connection refused, a certificate error, or a 413 means nginx.

    sudo nginx -t
    sudo tail -f /var/log/nginx/error.log

An `Exited (137)` in `dc ps` is the kernel OOM killer, not an application crash.
