Gitlab's out of the box docker in docker solution is hell on the network and developer patience as it doesn't cache anything.
This enables a shared docker cache between jobs which greatly speeds up build times.


# WARNING
This works by providing a shared docker in docker container, this means that any CI jobs using this runner can read, modify, and delete docker containers being pulled, built, or pushed by other CI jobs.
This may even extend to volumes and other resources.
Jobs operating on sensitive information should be siloed, or use the default docker in docker system with no caching.


# Docker DIND Caching Service

1. Select a wipe mode
	- Prune uses `docker prune` to clean (all!) docker volumes, but it can take a long time to process files
        - Copy `docker-wipe-prune.service` to `/etc/systemd/system/docker-wipe.service`
    - Format brings docker offline and formats the `/var/lib/docker/volumes` mount
    	- Modify the `docker-wipe-format.service` for your FS type, install to `/etc/systemd/system/docker-wipe.service`
    	- Copy the `docker.service.d` directory to `/etc/systemd/system` to make the mount requirement more explicit
1. Copy `gitlab-runner.service.d`, `dind-cache.service`, `dind-maintenance.service`, `dind-maintenance.timer` to `/etc/systemd/system` 
1. Run `systemctl daemon-reload` to load the new services.
1. Enable the services and timer with `systemctl enable dind-cache.service dind-maintenance.timer`
1. Restart the services by triggering a wipe with `systemctl start dind-maintenance.timer`
1. Register a new runner for the cached dind service, you should tag it appropriately.
1. Edit gitlab config file, add following bits to the runner:
    ```
    [[runners]]
      environment = ["DOCKER_HOST=tcp://dind-cache:2375", "DOCKER_DRIVER=overlay2"]
      [runners.docker]
        privileged = true
        network_mode = "dind-net"
    ```
1. Restart the runner service again to apply the new configuration.


Dind cache will be wiped once a week to help keep the cache from groing out of control and will cause all CI jobs on the server to halt during the wipe.

**NOTE:** The service override adds a 30 minute shutdown timeout to avoid interrupting jobs for maintenence, this can also negatively impact shutdown times on busy servers.

# TODO:
- Change prune to id the dind volumes instead of a blanket wipe