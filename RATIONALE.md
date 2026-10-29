# Project 2: Design Rationale

## Why I chose these images

For this project, I used `node:24` to build the application and `nginx:alpine` to serve 
the finished webpage. Node provides the tools needed to run `npm ci` and `npm run build`.
The build step in this project is simple: it copies `index.html` into the `build` folder.

Once that step is done, the webpage does not need Node.js or npm to run. It only needs a 
web server. That is why I used `nginx:alpine` for the final stage. The Dockerfile copies
the finished files from the build stage into Nginx’s HTML folder, leaving the Node build
tools behind. This follows the multi-stage build approach explained in Docker’s documentation [1].

I built both versions and recorded these results from the first size column in my Docker output:

| Image | Recorded size |
| --- | --- |
| Single-stage (`demo:single`) | 1.65 GB |
| Multi-stage (`demo:multi`) | 93.6 MB |

Using 1,650 MB for the single-stage image, the multi-stage version was about 94.3% 
smaller. These are approximate figures because Docker rounds the displayed sizes. My 
original results are saved in `evidence/t02-image-sizes.txt`.

The single-stage image keeps the full Node environment and adds `http-server`. The 
multi-stage image uses Nginx and the finished webpage instead. For this small application,
keeping the build tools in the final image would add size without helping it serve the page.

## Why I kept Redis separate

I used two services in Compose: `web` for Nginx and `cache` for Redis. Keeping them separate
means each runs in its own container and can be started, stopped, and checked separately.
It also keeps the web-server configuration separate from the cache configuration.

This connects to Week 3’s explanation of containers sitting between IaaS and PaaS. I decide
how the application is built and how the services work together, but I use existing images
for the tools underneath it. With `redis:7-alpine`, I am using a ready-made cache service
instead of writing a cache myself. I still manage the Compose configuration and need a 
working Docker environment.

The Redis healthcheck runs `redis-cli ping`. Compose waits for Redis to become healthy 
before starting the web service because I configured `depends_on` with `service_healthy`. 
Docker’s documentation explains this readiness check [2].

One important limitation is that my static webpage does not actually use Redis. During T05, 
I stopped Redis and the webpage still returned HTTP `200`. This showed that the configuration 
controls startup order. It did not show that the webpage depends on Redis while running or 
that it can recover from a real cache failure.

## References

[1] Docker. [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/).

[2] Docker. [Control startup and shutdown order in Compose](https://docs.docker.com/compose/how-tos/startup-order/).

My image-size measurements: `evidence/t02-image-sizes.txt`.
