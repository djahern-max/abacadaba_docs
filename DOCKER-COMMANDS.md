# Docker commands for the abacadaba Droplet

A practical reference. Everything here runs as `deploy` on the Droplet.

---

## The one thing that trips everyone up

Every command needs `-f docker-compose.prod.yml`. Without it, Compose looks for
`docker-compose.yml`, finds the *local development* one in the repo, and does
something you did not intend. There is no error, which is the annoying part.

Fix it once, in `~/.bashrc`:

    echo "alias dc='docker compose -f /srv/abacadaba/docker-compose.prod.yml'" >> ~/.bashrc
    source ~/.bashrc

Now `dc ps` works from any directory. Because Compose resolves paths relative to
the compose file rather than your current directory, the build context and the
`.env` file are still found correctly.

The rest of this file uses `dc`. If you skip the alias, substitute
`cd /srv/abacadaba && docker compose -f docker-compose.prod.yml` everywhere.

---

## Looking around

    dc ps                 # what is running, and is it healthy
    dc ps -a              # include stopped and exited containers
    docker ps             # every container on the box, not just this project

`dc ps` is the one you want 90 percent of the time. Read the STATUS column:

- `Up 3 hours (healthy)` — good
- `Up 3 hours` — running, no healthcheck defined (currently normal for `api`)
- `Restarting (1)` — crash looping, go read the logs
- `Exited (137)` — killed, usually out of memory

---

## Logs

    dc logs api                    # everything the api has said
    dc logs -f api                 # follow live, Ctrl-C to stop
    dc logs --tail 50 api          # last 50 lines only
    dc logs --since 10m api        # last 10 minutes
    dc logs                        # both services, interleaved

`--tail 50` is usually what you want. Full logs can be 30 MB.

Not Docker, but you will need them:

    sudo tail -f /var/log/nginx/error.log
    sudo tail -f /var/log/nginx/access.log

Rule of thumb: a 502 in the browser means nginx is fine and the container is
not, so check `dc logs api`. A connection refused or certificate error means
nginx, so check the nginx logs.

---

## Starting and stopping

    dc up -d                 # start everything, detached
    dc up -d api             # start just the api
    dc restart api           # restart without rebuilding, for a config change
    dc stop                  # stop containers, keep them
    dc down                  # stop and remove containers and networks

`down` does NOT delete your database. The data lives in a named volume that
survives. See the warning at the bottom for the flag that does delete it.

After editing `.env`, `restart` is not enough — environment variables are read
when the container is created. Use:

    dc up -d --force-recreate api

---

## Deploying a change

The order matters. Migrations run in a throwaway container before the api
restarts, so a bad migration stops you cleanly instead of leaving the api
serving against a schema it does not match.

    cd /srv/abacadaba
    git pull
    dc build api
    dc run --rm api alembic upgrade head
    dc up -d api
    dc logs --tail 30 api

Frontend changes do not involve Docker at all. Build on your laptop and rsync
`dist/` to `/var/www/abacadaba/`. The Droplet has no Node and should keep it
that way.

---

## Running a one-off command

`run --rm` starts a temporary container from the api image, runs one thing, and
removes itself. This is how you run scripts.

    dc run --rm api alembic upgrade head
    dc run --rm api alembic current
    dc run --rm api python -m scripts.seed
    dc run --rm api python -m scripts.make_admin someone@example.com

`--rm` matters. Without it you accumulate dead containers.

To poke around interactively in a fresh container:

    dc run --rm api bash

To get a shell inside the *already running* api, which is what you want when
debugging live state:

    dc exec api bash

`run` gives you a new container; `exec` gives you the one serving traffic.
Note that `/app` is read-only in the api container, so you cannot edit files
there. That is deliberate.

---

## The database

    dc exec db psql -U abacadaba -d abacadaba

That opens psql. Useful once inside:

    \dt                              list tables
    \d users                         describe the users table
    SELECT count(*) FROM users;
    SELECT email, is_admin FROM users;
    \q                               quit

One-liner without the interactive shell:

    dc exec db psql -U abacadaba -d abacadaba -c "SELECT count(*) FROM lessons;"

Manual backup, until the real backup script exists:

    dc exec db pg_dump -U abacadaba abacadaba | gzip > ~/backup-$(date +%F).sql.gz

Note that file is on the same disk as the database, so it protects against a
bad migration but not against losing the Droplet.

---

## Health checks

    curl -s localhost:8080/api/v1/health          # from the Droplet
    curl -s https://api.abacadaba.com/api/v1/health   # through nginx, from anywhere

Want `{"status":"ok","database":"connected"}`. If the first works and the second
does not, the problem is nginx or TLS, not the app.

---

## Resource usage

    docker stats              # live CPU and memory per container, Ctrl-C to exit
    docker stats --no-stream  # one snapshot
    df -h                     # disk, watch for Docker filling it
    free -h                   # memory and swap

`docker stats` is the first thing to check if the site feels slow. The api is
capped at 1 CPU and 768 MB; if it sits at its memory limit, something is wrong.

---

## Cleaning up

Docker accumulates old images and build layers and will eventually fill the
disk.

    docker system df           # how much space is being used
    docker image prune         # remove untagged images, safe
    docker system prune        # also removes stopped containers and networks

`docker system prune` will ask for confirmation and lists what it will remove.
Read the list. It does not touch named volumes unless you add `--volumes`.

---

## When something is wrong

In order:

    dc ps                        # is it running? healthy?
    dc logs --tail 50 api        # what did it say before it died?
    df -h                        # is the disk full? this causes weird failures
    free -h                      # is memory exhausted?
    dc exec db pg_isready -U abacadaba    # is the database reachable?
    ls -la /srv/abacadaba/.env   # does .env still exist, still 600?
    sudo nginx -t                # is the nginx config valid?

An `Exited (137)` is the kernel OOM killer. Raise `mem_limit` in the compose
file or find the leak.

---

## Two commands to be careful with

    dc down -v

The `-v` deletes named volumes, which means it **deletes your entire database**.
There is no confirmation prompt. There is no undo. The only reason to run this
is deliberately wiping the environment to start clean.

    docker system prune -a --volumes

Same problem, larger blast radius. Removes all unused images and all unused
volumes.

Neither of these is ever part of a normal deploy. If a set of instructions
tells you to run one to fix a problem, stop and back up first.


# backend
cd /srv/abacadaba/backend
source .venv/bin/activate      # or however the service is set up
pip install -r requirements.txt
alembic upgrade head           # no-op this time, but keep the habit
sudo systemctl restart abacadaba-api   # whatever the unit is actually named

# frontend — this is the part people forget
cd /srv/abacadaba/frontend
npm ci
npm run build                  # reads .env.production