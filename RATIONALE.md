# Project 2: Design rationale

## Base images and multi-stage build

This project serves a static HTML page. I chose node:24 for the build
stage because it supplies Node.js and npm for the provided build process.
The Dockerfile copies package.json and package-lock.json, runs npm ci,
and then runs npm run build. In this demonstration, the build step copies
index.html into the build directory.

For the final stage, I chose nginx:alpine because the running application
only needs a web server to serve the generated HTML. It does not need
Node.js or npm at runtime. The final stage copies /app/build from the
build stage into Nginx's HTML directory. Docker's multi-stage documentation
explains how selectively copying artifacts between stages leaves build
tools out of the final image [1].

My T02 measurements, saved in evidence/t02-image-sizes.txt, were:

| Image | First reported size column |
| --- | --- |
| demo:single | 1.65 GB |
| demo:multi | 93.6 MB |

Using 1.65 GB = 1,650 MB, the multi-stage image was approximately 17.6
times smaller, a reduction of about 94.3%. These calculations use rounded
displayed values from the same column. The evidence also contains a second
size column, which I did not mix into this comparison.

The single-stage comparison retains the Node environment and installs
http-server. The multi-stage version instead ships the Nginx runtime and
built output. These measurements support choosing the smaller runtime
for this application; they do not prove it is the smallest possible
image or guarantee its security.

## Service choices and separation

The Compose stack separates web serving from the Redis backing-service
demonstration. The web service builds my Dockerfile, while cache uses
the prebuilt redis:7-alpine image. Separate containers give each service
its own process, configuration, and lifecycle, rather than combining
both programs inside one container.

This illustrates the course's framing of containers between IaaS and
PaaS. I control the application image, port mapping, service composition,
and readiness checks, while accepting maintained base images and a
prebuilt Redis implementation instead of implementing those components
myself. The host and container runtime still require management.

Redis has a redis-cli ping healthcheck. The web service uses depends_on
with condition: service_healthy, so startup waits for Redis readiness,
as described in Docker's Compose documentation [2].

The static webpage does not actually read from or write to Redis. In
T05, I stopped cache and observed that web remained running and returned
HTTP 200. This demonstrates startup ordering, not an ongoing application
dependency or successful recovery from a real cache outage.

## References

[1] Docker: Multi-stage builds.
https://docs.docker.com/build/building/multi-stage/

[2] Docker: Control startup and shutdown order in Compose.
https://docs.docker.com/compose/how-tos/startup-order/

Measurement evidence: evidence/t02-image-sizes.txt.
