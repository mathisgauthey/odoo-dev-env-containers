# About This Repo

This repo is meant to **simplify the dev environment for odoo development**.

The only prerequisite is to have [Docker](https://www.docker.com/) installed on your machine (or [Rancher Desktop](https://rancherdesktop.io/)), and Git to clone this repo.

Using Docker allows us to :

1. Have a simple onboarding process for new developers without having to struggle with unmet dependencies or different OS. It's literraly a matter of cloning the repo and running `docker-compose up`.
2. Have a consistent environment across all developers.
3. Have a similar environment for testing and production.

> No more manual set up of the database, no more manual set up of a venv, no more manual set up of the odoo configuration and launch arguments, and no more struggling with unmet dependencies on different operating systems.

## What Directory Structure Should I Use to Develop Odoo Addons ?

Based on [odoo docs](https://www.odoo.com/documentation/17.0/developer/tutorials/getting_started/01_architecture.html#module-structure), the following directory structure should be applied when developping odoo addons:

```txt
REPO_NAME/
    ├── .git/
    ├── MODULE_NAME/
    │   ├── __init__.py
    │   ├── __manifest__.py
    │   └── ... # Other module files and folders (Detailled below)
    ├── .gitignore
    ├── LICENSE
    ├── README.md
    └── requirements.txt
```

Odoo.sh is handling `requirements.txt` files in [the parent folder](https://www.odoo.com/documentation/17.0/administration/odoo_sh/advanced/containers.html#overview) of the modules, hence the file structure above.

> The requirements.txt files of submodules are taken into account as well. The platform looks for requirements.txt files in each folder containing Odoo modules: **Not in the module folder itself, but in their parent folder.**

If you don't use Odoo.sh, you might need to use a bash script that looks for `requirements.txt` files and install the dependencies.

You can find informations about [coding guidelines in details here](https://www.odoo.com/documentation/17.0/contributing/development/coding_guidelines.html).

> A module is organized in important directories. Those contain the business logic; having a look at them should make you understand the purpose of the module.
>
> - data/ : demo and data xml
> - models/ : models definition
> - controllers/ : contains controllers (HTTP routes)
> - views/ : contains the views and templates
> - static/ : contains the web assets, separated into css/, js/, img/, lib/, …
>
> Other optional directories compose the module:
>
> - wizard/ : regroups the transient models (models.TransientModel) and their views
> - report/ : contains the printable reports and models based on SQL views. Python objects and XML views are included in this directory
> - tests/ : contains the Python tests

## Database (PostgreSQL) Configuration

We create a database (you need to keep it named `postgres`, but more about this in a [later section](#some-notes-on-the-odoo-and-database-configuration)), a user ([which is not a superuser as required by odoo](https://www.odoo.com/documentation/17.0/administration/install/deploy.html#configuring-odoo)) and a password for the database in the `docker-compose.yml` file.

```yml
environment:
            - POSTGRES_DB=postgres
            - POSTGRES_USER=odoo
            - POSTGRES_PASSWORD=odoo
            - PGDATA=/var/lib/postgresql/data/pgdata
```

You can see `PGDATA=/var/lib/postgresql/data/pgdata` in the `docker-compose.yml` file. This is because of the following:

> [PGDATA](https://hub.docker.com/_/postgres)
>
> Important Note: when mounting a volume to /var/lib/postgresql, the /var/lib/postgresql/data path is a local volume from the container runtime, thus data is not persisted on the mounted volume.
>
> This optional variable can be used to define another location - like a subdirectory - for the database files. The default is /var/lib/postgresql/data. If the data volume you're using is a filesystem mountpoint (like with GCE persistent disks), or remote folder that cannot be chowned to the postgres user (like some NFS mounts), or contains folders/files (e.g. lost+found), Postgres initdb requires a subdirectory to be created within the mountpoint to contain the data.

Note that we used port `5433` for the host and `5432` for the container. This is because we want to avoid conflicts with other PostgreSQL instances that might be running on the host, an existing postgreSQL installation on WSL for example.

## Odoo Configuration

We need to specify the database container `db`, the port (`5432` by default with PosgreSQL), the database user and password in the `docker-compose.yml` file.

```yml
environment:
            - HOST=db
            - PORT=5432
            - USER=odoo
            - PASSWORD=odoo
```

Then, to configure Odoo, we get to choose if we prefer to use :

- [The `odoo.conf` file](https://www.odoo.com/documentation/17.0/administration/install/deploy.html)
- [The command line arguments](https://www.odoo.com/documentation/17.0/developer/reference/cli.html#cmdoption-odoo-bin-d) by replacing or adding to the default Docker `CMD` with `odoo --ARG_NAME ARG_VALUE` in the `Dockerfile` file.

A good practice is to **use only one** of the two methods to configure Odoo.

Note that some arguments are [named differently](https://www.odoo.com/documentation/17.0/developer/reference/cli.html#configuration-file) depending on the method used :

- `--db-filter` becomes `dbfilter` in `odoo.conf`
- `--database` becomes `db_name` in `odoo.conf`
- `--addons-path` becomes `addons_path` in `odoo.conf`
- `--data-dir` becomes `data_dir` in `odoo.conf`

## Docker Volumes

We use Docker volumes to persist the database data and the Odoo addons.

```yml
volumes:
    odoo-data:
    odoo-db-data:
```

Should you want to start fresh, you need to :

1. Stop the containers : `docker-compose down`
2. Remove the containers : `docker-compose rm`
3. Remove the volumes : `docker volume rm VOLUME_NAMES`

## Some Notes on the Odoo and Database Configuration

We decided to use `odoo-db` as database name. By default, Odoo uses `postgres` as we can see on [DockerHub](https://hub.docker.com/_/odoo/) or on their docs.

If we decide to use ours, we need to be sure that the `odoo.conf` file or the command line arguments are correctly configured.

We need :

- The database host `db_host`
- The database port `db_port`
- The database filter `db_filter`
- The database name `db_name`
- The database user `db_user`
- The database password `db_password`

Based on the [Dockerhub instructions](https://hub.docker.com/_/odoo/), we need to setup the database using the `POSTGRES_DB=postgres` environment variable in the `docker-compose.yml` file.

This is because the database need to be initialiwed when Odoo first start. That's why we gave it a different name in the `odoo.conf` file. Hence when Odoo first starts, it will create the database `odoo-db` and initialise it.

If we'd used `POSTGRES_DB=odoo-db` in the `docker-compose.yml` file, the database would have been created with the name `odoo-db` and Odoo would have failed to start because it would have been unable to find the database `postgres`. It would have asked us to force the database initialisation using `--init base` flag on our first start-up.

## How to access Odoo source code from the container

You can access it by connecting to your container using the following command :

```bash
docker exec -it CONTAINER_NAME bash
```

And then you need to go to `/usr/lib/python3/dist-packages/odoo` to access the source code.
