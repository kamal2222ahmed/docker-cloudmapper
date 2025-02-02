# Dockerized Cloudmapper

[Cloudmapper](https://github.com/duo-labs/cloudmapper) is a tool to visualize AWS infrastructure.  In early July 2018, [Docker support was
removed](https://github.com/duo-labs/cloudmapper/commit/c430e5faab41e8355052e82fe9dea4d8f4beb654) from their repo.  I wrote this project to bring back support for Docker.

I personally find that running this application is docker is easier because it avoids issues with dependency version conflicts.  Running it in Docker "Just Works™" and keeps my host machine nice and clean.

## Quick Start

Prerequisites: just `Docker` :)

- `git clone https://github.com/sebastientaggart/docker-cloudmapper.git`
- `cd docker-cloudmapper`
- Edit `.env` file and add your AWS key, secret, region, and account name.
- `docker-compose build` (go get a cup of coffee)
- `docker-compose up -d`

Go to http://localhost:8000 and enjoy!

Stop the service with `docker-compose down`.  Back up with `docker-compose up -d`.  View logs with `docker-compose logs -f`.  Do a rebuild with `docker-compose build`, or `docker-compose build --no-cache` if something went wrong.

### Logging into the Container

Once the service is running you can log into the container with `docker exec -it cloudmapper sh`.  You will automatically be put into `/opt/cloudmapper`, from which all the usual Cloudmapper commands will run.  Note that `$ACCOUNT` and other variables are already set for you in the container, so these command will Just Work™

- `pipenv run python cloudmapper.py collect --account $ACCOUNT` Collect info about our AWS infrastructure
- `pipenv run python cloudmapper.py prepare --account $ACCOUNT` Prepare the collected data for serving

## How This Works / Approach

This project starts with a `Dockerfile` (loosely based on the original dockerfile from the Cloudmapper project), based on `python:alpine` and built via `docker-compose build`, which does the following:

1. Install necessary packages, e.g. `git`
2. Clone cloudmapper.git and enable pipenv
3. Install awscli
4. Copy the `entrypoint.sh` and `.env` files into the container
5. Run `entrypoint.sh`, which
    1. `source`'s `.env`
    2. Configures AWS CLI with the variables from `.env`
    3. Run `[...] cloudmapper.py configure [...]`, `collect`, and `prepare` commands with variables from `.env`
    4. Serve the generated site on `:8000` (assuming you didn't change this in `.env`)

Next, we execute `docker-compose up -d`...

1. Call the image we build `cloudmapper`
2. Call the container `cloudmapper`
3. `env_file: .env` negates the need to `source` it in `Dockerfile`, but we do it anyway here in anticipation of the "Further Improvements" (see below)
4. Volume our data:
    - `./account-data` stores the json files that are generated during `collect`.
    - `./web` stores the generated web files that display the visualization generated by Cloudmapper
5. Set the external port based on the value in `.env` (for convenience, and to limit the number of files necessary to configure)

### Debugging Reference

If you want to help contribute and/or debug, and you're new to Docker, these might save you some time:

Build without docker-compose:
`docker build --pull --tag cloudmapper .`
`docker build --tag cloudmapper .` (without force pulling base container)

Problems with the build?  Debug and force a rebuild with no-cache:
`docker build --no-cache --tag cloudmapper .`

Add `CMD tail -f /dev/null` to the `Dockerfile` to keep it alive, disable `entrypoint.sh`

## Further Improvements

Instead of automatically running Cloudmapper `collect`, `prepare`, and `webserver`, we should instead have a container that runs these commands externally for more flexibility.  The approach would then be to run something like `docker exec cloudmapper collect --account FOO`.  This way, we could add another account, for example, and regenerate the visualization as needed, etc.  We would still want a simple "quick start" approach that would get people up and running in just a few steps.  Note that this is possible now, with `docker exec -it cloudmapper sh` and running any commands you want.

The current approach also assumes that you only need to run `collect` and `prepare` once (on container build).  This is not practical in all situations, and the current solution, `docker-compose build --no-cache` (a forced rebuild from scratch) is not an efficient solution.  Building the container as described above will mitigate this problem.

Another improvement would be to serve `/web` via nginx instead of the built-in `cloudmapper.py webserver` to improve performance, and because it's "best practice", but I did not see a real-world, measurable improvement in performance.  We would add nginx to docker-compose.yml for this.

"Debug Mode" is not yet working in this build.  I added the variables for it, and the code is ready, but I don't personally have any need for demo data, so I didn't finish setting it up.

Patch: https://github.com/syucream/docker-cloudmapper/commit/f2f1b5a5bb308828c9d6c4f3c0da3d358109423d

## License

Cloudmapper is `Copyright 2018 Duo Security`, see https://github.com/duo-labs/cloudmapper/blob/master/LICENSE

As with all Docker images, these likely also contain other software which may be under other licenses (such as Python, etc from the base distribution, along with any direct or indirect dependencies of the primary software being contained).
