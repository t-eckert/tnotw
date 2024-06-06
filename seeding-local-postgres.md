# Seeding Local Postgres

Postgres is my go-to database for any project I develop. It's fast, well-documented, and available in so many places. When I'm working on a project locally, I'll want to run a local instance of Postgres to test the behavior of the project end-to-end. I've always done this using Docker and the [official Postgres image](https://hub.docker.com/_/postgres).

This worked fine, but I always had to develop some external solution for migrating and seeding the database. Migrations add tables to the database to match what the application expects. Seeding inserts example data, sometimes a copy of production data, to make interactions in the local environment closer to what the user experiences.

I used to write scripts to shell into the running container to perform the migration and seeding, but this presented a few challenges. This process can be slow and flakey if you mess up your script. It was hard to know if the database was already seeded. And seeding, unlike migrations, is not idempotent.

Trying to fix this for my latest project, I wondered if I could build a custom container image on top of Postgres that included my migration and seed by default. That way, I would always know that the database was set up properly by the time it was running. It turns out, this is a well-supported way of working with Postgres in Docker.

All I had to do is place the migration and seed SQL into the `/docker-entrypoint-initdb.d/` directory before startup. When Postgres starts, it will run all `*.sql`, `*.sql.gz`, and `*.sh` scripts within this directory. [See the docs](https://github.com/docker-library/docs/blob/master/postgres/README.md#initialization-scripts). I wrote a Dockerfile with a copy step for `migrations` and the `seed` directories. It works great!

```Dockerfile
FROM postgres:alpine

COPY ./migrations/ /docker-entrypoint-initdb.d/
COPY ./seed/ /docker-entrypoint-initdb.d/

CMD ["postgres"]
```

Do note that the naming here matters. Postgres will run the files in this directory in alphabetical order. Here, that works well because "m" comes before "s" so the database will be migrated before being seeded. But it's something to be aware of.

One of the immediate benefits to this approach is how fast it is when compared to shelling into a running container and running migrations. In the gif below, the image is built and run within milliseconds (because I already had the Postgres image locally).

I highly recommend this approach for making sure your Postgres database is properly set up when you run your project locally. Let me know if you try it and find it useful too!
