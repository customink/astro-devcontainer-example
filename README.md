# Astro Devcontainer Example

This is an example project that illustrates how to use the base images provided by Astronomer in order to leverage local [devcontainers for development](https://containers.dev) rather than the Astro CLI (specifically, the `astro dev` sub-commands).

## How to Use

1. Clone this repo
2. Use the command palette in VSCode to open the project in a devcontainer (the `Open Folder in Container` option)
3. After `postCreate` is finished running, run `honcho start` in the terminal, which will start the airflow webserver, scheduler, and triggerer.

**Note:** postgres is running on 5436 to avoid conflicting with the default 5433 that the Astro CLI (`astro dev`) uses. The webserver is running on 8082 for similar reasons.

### Honcho

[Honcho](https://github.com/nickstenning/honcho) is a Procfile runner written in Python. It uses the Procfile format, [which is used by Heroku](https://devcenter.heroku.com/articles/procfile) and others.

## Why Use Devcontainers vs `astro dev`?

For users of the Astronomer platform the default method of working with a local Astro/Apache Airflow setup is provided by the Astro CLI. Specifically it is provided by the subcommands under `astro dev`. However, there are some limitations with that approach which can make development a little bit of a bumpy experience.

### 1. `astro dev start` always rebuilds the container

This is due to the base images (i.e. `quay.io/astronomer/astro-runtime:9.6.0`) using the `ONBUILD` instruction of Docker. Here is a snippet of [the Docker docs about `ONBUILD`](https://docs.docker.com/engine/reference/builder/#onbuild):

> The ONBUILD instruction adds to the image a trigger instruction to be executed at a later time, when the image is used as the base for another build. The trigger will be executed in the context of the downstream build, as if it had been inserted immediately after the FROM instruction in the downstream Dockerfile.

You can check the `ONBUILD` instructions by running `docker inspect
quay.io/astronomer/astro-runtime:9.6.0`, which will then show something
like:

```json
"OnBuild": [
  "COPY packages.txt .",
  "USER root",
  "RUN if [[ -s packages.txt ]]; then     apt-get update && cat packages.txt | tr '\\r\\n' '\\n' | sed -e 's/#.*//' | xargs apt-get install -y --no-install-recommends     && apt-get clean     && rm -rf /var/lib/apt/lists/*;   fi",
  "COPY requirements.txt .",
  "RUN if grep -Eqx 'apache-airflow\\s*[=~>]{1,2}.*' requirements.txt; then     echo >&2 \"Do not upgrade by specifying 'apache-airflow' in your requirements.txt, change the base image instead!\";  exit 1;   fi;   pip install --no-cache-dir -r requirements.txt",
  "USER astro",
  "COPY --chown=astro:0 . ."
]
```

What that means is that when you use `FROM quay.io/astronomer/astro-runtime:9.6.0` as the base image, your default file will essentially look like this:

```
FROM quay.io/astronomer/astro-runtime:9.6.0

COPY packages.txt .
USER root
RUN if [[ -s packages.txt ]]; then     apt-get update && cat packages.txt | tr '\\r\\n' '\\n' | sed -e 's/#.*//' | xargs apt-get install -y --no-install-recommends     && apt-get clean     && rm -rf /var/lib/apt/lists/*;   fi
COPY requirements.txt .
RUN if grep -Eqx 'apache-airflow\\s*[=~>]{1,2}.*' requirements.txt; then     echo >&2 \"Do not upgrade by specifying 'apache-airflow' in your requirements.txt, change the base image instead!\";  exit 1;   fi;   pip install --no-cache-dir -r requirements.txt
USER astro
COPY --chown=astro:0 . .

# your-stuff-here
```

Because development inherently means changing files in the project, that last line, `COPY --chown=astro:0 . .` is not able to use the Docker cache between `astro dev stop` and `astro dev start`, which results in everything after it (all of the user's additions) rebuilding. This essentially means that every time you stop and start the astro dev containers they have to rebuild. If you have anything meaningful in your Dockerfile, this can be quite a painful experience.

### 2. No development tools in the image

The `astro dev` tooling uses the same `Dockerfile` as production does. There are some good things about this. Running something in development close to what is in production means you can be a bit more certain that your changes will work in production. However, there are a number of downsides as well. One those downsides is that it cannot be customized for development, at least not using the `astro dev` tooling and that is the only mechanism provided. Even though [the `astro dev` tooling uses docker compose under the hood](https://github.com/astronomer/astro-cli/blob/v1.21.0/airflow/include/composeyml.yml). We do have some capabilities by using a `docker-compose.override.yml`, but that has its limitations. The shell is still just sh, there are no helpful utilities installed in the image (nslookup, dig, etc.), no user customizations, and no debugging tools. I personally attach my VSCode to the scheduler container and install all my extensions in order to be able to run a debugger, the Airflow CLI, etc. Due to the constant rebuilding of the image as described above, even if you do install some helpful tools into the image after it's built, you cannot stop the container and restart it, because that will lead to the image being rebuilt, which means all your customizations will be gone and they will have to be reinstalled all over again.

### 3. Only specific directories are bound when using `astro dev`

When we use the `astro dev` tooling, [only specific directories from the project are bind mounted volumes in the container](https://github.com/astronomer/astro-cli/blob/v1.21.0/airflow/include/composeyml.yml#L65-L68). This means that any files that you change that don't exist in those directories will not be reflected in the container. All the volumes are also read-only within the container. So you can only change files in the container from outside the container by default. The only way to fix this that plays with `astro dev` is to use a `docker-compose.override.yml` and change the volume so that it's writable from within the container. Otherwise you are stuck editing files from outside of the container, which means you do not have access to the airflow CLI, or anything else that only exists within the container. Refer back to the second point here.

### Others?

Those are the main issues, but there are other smaller ones as well and nuances to those mentioned above which were not discussed.
