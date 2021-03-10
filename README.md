Gitlab's out of the box docker in docker solution is hell on the network and developer patience as it doesn't cache anything.
This enables a shared docker cache between jobs which greatly speeds up build times.


# WARNING
This works by providing a shared docker in docker container, this means that any CI jobs using this runner can read, modify, and delete docker containers being pulled, built, or pushed by other CI jobs.
This may even extend to volumes and other resources.
Jobs operating on sensitive information should be siloed, or use the default docker in docker system with no caching.


# Docker DIND Caching Service

1. Select a wipe mode
	- Prune deletes the docker volume assocaited with the container, but processing time grows with the cache size
        - Copy `docker-wipe-prune.service` to `/etc/systemd/system/dind-wipe.service`
        - Copy `dind-cache-prune.service` to `/etc/systemd/system/dind-cache.service`
    - Format formats the mount associated with the cache (`/opt/dind_vld` by default) which is much faster
    	- Modify the `docker-wipe-format.service` for your FS type and mount path, install to `/etc/systemd/system/docker-wipe.service`
    	- Modify `dind-cache-format.service` for your mount path and install to `/etc/systemd/system/dind-cache.service`
1. Copy `gitlab-runner.service.d`, `dind-maintenance.service`, `dind-maintenance.timer` to `/etc/systemd/system` 
1. Run `systemctl daemon-reload` to load the new services.
1. Enable the services and timer with `systemctl enable dind-cache.service dind-maintenance.timer`
1. Enable the timer and restart everything by triggering a wipe with `systemctl start dind-maintenance.timer dind-maintenence.service`
1. Register a new runner for the cached dind service.
	- You'll probably want to tag it appropriately and limit it to tags only.
1. Edit gitlab config file, add following bits to the runner:
    ```
    [[runners]]
      environment = ["DOCKER_HOST=tcp://docker:2376", "DOCKER_DRIVER=overlay2", "DOCKER_CERT_PATH=/certs/client", "DOCKER_TLS_VERIFY=1"]
      [runners.docker]
        privileged = true
        network_mode = "dind-net"
        volumes = ["dind_client_certs:/certs/client:ro"]
    ```
1. The runner service should detect the config change and reload


Dind cache will be wiped once a week to help keep the cache from groing out of control and will cause all CI jobs on the server to halt during the wipe.

**NOTE:** The service override adds a 30 minute shutdown timeout to avoid interrupting jobs for maintenence, this can also negatively impact shutdown times on busy servers.

# TODO:
- Change prune to id the dind volumes instead of a blanket wipe