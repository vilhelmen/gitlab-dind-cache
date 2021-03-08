Gitlab's out of the box docker in docker solution is hell on the network as it doesn't cache anything.
This enables a shared docker cache between jobs which greatly speeds up build times.


# WARNING
This works by providing a shared docker in docker, this means that any CI jobs of this class can read, modify, and delete docker containers being pulled/built/pushed by other CI jobs.
This may even extend to volumes and other resources.
Jobs operating on sensitive information should be siloed, or use the default docker in docker system with no caching.


# Docker DIND Caching Service

1. `docker network create dind-net`
1. Register a gitlab runner for the cached dind service, you should tag it appropriately.
1. Edit gitlab config file, add following bits to the runner:
    ```
    [[runners]]
      environment = ["DOCKER_HOST=tcp://dind-cache:2375", "DOCKER_DRIVER=overlay2"]
      [runners.docker]
        privileged = true
        network_mode = "dind-net"
    ```
1. Stop the runner service
1. Install the dind/wipe services/timers and runner override
3. Do a daemon-reload
4. Restart gitlab-runner, it should now bring up and rely on the dind-cache service.

**NOTE:** Gitlab runner will now take up to one hour to shutdown, which means system shutdown will be equally tied up!

Dind cache will be wiped once a week and will cause all CI to halt during the wipe.

### TODO:
1. Update wipe to optionally reformat a volume partition. So much faster than having docker delete files but, ya know, it's wiping a disk.
1. Check it works on things besides Cent7
