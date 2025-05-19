# Setting Up Docker Dev Environment 

**_TODO:_**

- Test the cmd sequence below with copy & paste - do I get a running fpan site?
- Update README with a reference to this (or replace contents with this file's contents and delete this file)
- Address # TODO: in command below

## Instructions

Create a workspace for `fpan` and add files necessary for Docker builds:

```bash
mkdir fpan-workspace && cd fpan-workspace

# clone our arches fork
git clone https://github.com/legiongis/arches
cd arches && git fetch --all && git checkout dev/6.2.x-hms-cli
cd ..

# clone this repo
git clone https://github.com/legiongis/fpan

# create settings_local.py
touch fpan/fpan/settings_local.py
cat << 'EOF' > fpan/fpan/settings_local.py
from .settings import DATABASES, INSTALLED_APPS, MODE
import os

# TODO: should this be moved to settings.py to be checked into git?
INSTALLED_APPS += ('arches_extensions',)

DEBUG = True
MODE = 'DEV'

# pull from env vars defined in docker-compose.yml > arches
DATABASES['default']['USER'] = os.environ.get('PGUSERNAME', 'username')
DATABASES['default']['PASSWORD'] = os.environ.get('PGPASSWORD', 'password')
DATABASES['default']['HOST'] = os.environ.get('PGHOST', 'localhost')
DATABASES['default']['PORT'] = os.environ.get('PGPORT', '5432')
DATABASES['default']['NAME'] = os.environ.get('PGDBNAME', 'fpan')
DATABASES['default']['POSTGIS_TEMPLATE'] = "template_postgis"
EOF
```

Once the above command sequence has run, you should have this directory structure:

```
fpan-workspace
├── arches
└── fpan
```

Build and run the docker containers & network, and watch `fpan`'s server logs (via the `arches` container):

```bash
cd arches  # root of arches 6.2 fork repo
docker compose up -d
docker compose logs arches -f
```

This builds the containers for Arches core, postgres, couchdb, elasticsearch, and nginx (with letsencrypt). It also mounts the host machine's `fpan` as a volume to the Arches core Docker container `arches`, so you can work on fpan without rebuilding the `arches` core docker image.

> ⚠️ **Starting the docker containers could take several minutes**, and several more, the first time you run `docker compose`. It will likely appear to hang after you see this output in the logs:

```
# log output
arches  | loading post sql
arches  | indexing database
arches  | deleting index : fpan_terms
arches  | deleting index : fpan_concepts
arches  | deleting index : fpan_resources
arches  | deleting index : fpan_resource_relations
arches  | creating index : fpan_terms
arches  | creating index : fpan_concepts
arches  | creating index : fpan_resources
arches  | package load complete  # <-- last line before it hangs
```

> All `fpan` python requirements are installed at container runtime, which has the advantage of not needing to rebuild Docker images when `fpan` dependencies change, but the disadvantage of fetching and installing many python packages every time you run `docker compose up -d`

If successful, the last message you should see in the `arches` container log is:

```
# log output
arches  | Watching for file changes with StatReloader
```

You should now see the site in the browser at `http://localhost:8000`.

`cd ../fpan` and make your changes.

## Helpful Commands:

```bash
# kill the containers but keep persistent data generated into volumes during runtime
cd arches
docker compose down

# kill the containers and wipe out persistent data volumes
cd arches
docker compose down -v

# run the containers in the background
cd arches
docker compose up -d

# check which containers are running
cd arches
docker compose ps

# watch fpan logs:
cd arches
docker compose logs arches -f

# ssh into the server running in the `arches` container:
cd arches
docker compose exec arches bash
```

## Running on an arm64 or Apple Silicon Host Machine

`fpan` can run on an M1 Mac (arm64) using Docker's own virtualization that's currently in beta (Docker Desktop > Settings > General > Virtual Machine Options > Docker VMM). There are some system dependencies for Arches core 6.2 that require amd64 architecture, so amd64 architecture is forced for the `arches` container in docker-compose.yml.
