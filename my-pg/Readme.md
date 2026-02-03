# Lab Handout: Building PostgreSQL 18 for Constrained Environments

## Part 1: The Blueprint (Dockerfile)

Create a file named Dockerfile. This script automates the "Build from Source" process using **Alpine Linux** to minimize OS overhead.

```dockerfile
# STAGE 1: THE FORGE (BUILDER)
FROM alpine:3.19 AS builder

# Install build dependencies for Alpine
# Note: liburing-dev is essential for Postgres 18 AIO
RUN apk add --no-cache \
    build-base \
    curl \
    bison \
    flex \
    liburing-dev \
    openssl-dev \
    zlib-dev \
    perl \
    linux-headers

# WORKDIR /usr/src/postgresql
RUN curl -o pg18.tar.gz https://ftp.postgresql.org/pub/source/v18.0/postgresql-18.0.tar.gz \
    && tar -xzf pg18.tar.gz --strip-components=1

# Optimized for 512MB RAM: Strip unused features
RUN ./configure --prefix=/usr/local/pgsql \
    --with-openssl \
    --with-liburing \
    --without-readline \
    --without-icu \
    CFLAGS="-O2 -march=native"

RUN make -j$(nproc) && make install

# STAGE 2: THE RUNNER (PRODUCTION)
FROM alpine:3.19

# Install only the Alpine runtime libraries
RUN apk add --no-cache \
    liburing \
    openssl \
    zlib \
    musl-locales \
    bash

# Create postgres user and data directory
COPY --from=builder /usr/local/pgsql /usr/local/pgsql
ENV PGDATA=/var/lib/postgresql/data
ENV PATH="/usr/local/pgsql/bin:$PATH"

RUN adduser -D postgres && \
    mkdir -p /var/lib/postgresql/data && \
    chown -R postgres:postgres /var/lib/postgresql /usr/local/pgsql

COPY entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

USER postgres
WORKDIR /var/lib/postgresql
ENTRYPOINT ["entrypoint.sh"]
CMD ["postgres"]
```

## Part 2: Deployment & Resource Caging

Build the image and run it with strict resource limits.

```bash
# Build the custom image (This will take a few minutes)
docker build -t pg18-512mb --no-cache .

# Run the container with 512MB RAM and 0.5 CPU Core
docker run --rm --name pg_30_lab \
    --memory="512m" \
    --memory-swap="512m" \
    --cpus="2.0" \
    --volume=./DATA:/var/lib/postgresql/data \
    --security-opt seccomp=unconfined \
    pg18-512mb \
    postgres -D /var/lib/postgresql/data \
    -c max_connections=30 \
    -c work_mem=16MB \
    -c shared_buffers=128MB

```

## Part 3: Configuration & Tuning (The Mission)

Access the container and modify the settings to fit the 512MB RAM budget.

```bash
# Enter the container shell
docker exec -it pg_30_lab sh

```

> Task: Update the following parameters in postgresql.conf:

Parameter | Recommended Value | Reason
----------|-------------------|--------
max_connections | 20 | Reduces RAM overhead per process.
shared_buffers | 128MB | Reserves 25% of RAM for DB Cache.
work_mem | 4MB | Small buffer for sorting to avoid OOM.
random_page_cost | 1.1 | ptimized for SSD/Flash storage.
effective_cache_size | 200MB | Helps the optimizer understand OS-level caching.



## Part 4: Stress Test (pgbench)
Run a benchmark to see if your configuration remains stable under load.

```bash
# 1. Initialize test data (Scale factor 10)
docker exec -it pg_30_lab pgbench -i -s 10 postgres

# 2. Run the test for 60 seconds with 20 clients
docker exec -it pg_30_lab pgbench -c 20 -j 2 -T 60 postgres
```

## Part 5: docker stats
Run docker stat to see docker stats
```
docker stats
```

## Lab Report Questions

1. Did the container crash during pgbench? If so, check docker logs my_pg_lab for "OOM" messages.
2. What was your final TPS (Transactions Per Second)?
3. Why did we reduce max_connections from the default 100 to 20?
4. What is the benefit of using a Custom Build over a standard docker pull postgres?