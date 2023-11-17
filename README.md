# Slurm Docker Cluster

This is a multi-container Slurm cluster using docker-compose.  The compose file
creates named volumes for persistent storage of MySQL data files as well as
Slurm state and log directories.

## Containers and Volumes

The compose file will run the following containers:

* mysql
* slurmdbd
* slurmctld
* c1 (slurmd)
* c2 (slurmd)

The compose file will create the following named volumes:

* etc_munge         ( -> /etc/munge     )
* etc_slurm         ( -> /etc/slurm     )
* slurm_jobdir      ( -> /data          )
* var_lib_mysql     ( -> /var/lib/mysql )
* var_log_slurm     ( -> /var/log/slurm )

## Building the Docker Image

Build the image locally:

```console
docker build -t slurm-docker-cluster:21.08 .
```

Build a different version of Slurm using Docker build args and the Slurm Git
tag:

```console
docker build --build-arg SLURM_TAG="slurm-19-05-2-1" -t slurm-docker-cluster:19.05.2 .
```

Or equivalently using `docker-compose`:

```console
SLURM_TAG=slurm-19-05-2-1 IMAGE_TAG=19.05.2 docker-compose build
```


## Starting the Cluster

Run `docker-compose` to instantiate the cluster:

```console
IMAGE_TAG=19.05.2 docker-compose up -d
```

## Register the Cluster with SlurmDBD

To register the cluster to the slurmdbd daemon, run the `register_cluster.sh`
script:

```console
./register_cluster.sh
```

> Note: You may have to wait a few seconds for the cluster daemons to become
> ready before registering the cluster.  Otherwise, you may get an error such
> as **sacctmgr: error: Problem talking to the database: Connection refused**.
>
> You can check the status of the cluster by viewing the logs: `docker-compose
> logs -f`

## Accessing the Cluster from the command line

Use `docker exec` to run a bash shell on the controller container:

```console
docker exec -it slurmctld bash
```

From the shell, execute slurm commands, for example:

```console
[root@slurmctld /]# sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
normal*      up 5-00:00:00      2   idle c[1-2]
```

## Submitting Jobs

The `slurm_jobdir` named volume is mounted on each Slurm container as `/data`.
Therefore, in order to see job output files while on the controller, change to
the `/data` directory when on the **slurmctld** container and then submit a job:

```console
[root@slurmctld /]# cd /data/
[root@slurmctld data]# sbatch --wrap="uptime"
Submitted batch job 2
[root@slurmctld data]# ls
slurm-2.out
```

## Accessing the Cluster via Slurm REST API

Create a JSON Web Token (JWT) for authentication to `slurmrestd`.

```console
docker exec slurmctld /usr/bin/scontrol token username=slurm
```

Copy/paste the result in your shell to create the `SLURM_JWT` env variable.

List all the OpenAPI plugins supported by `slurmrestd`.

```console
docker exec slurmctld /usr/sbin/slurmrestd -s list
```

```console
slurmrestd: Possible OpenAPI plugins:
slurmrestd: openapi/v0.0.35
slurmrestd: openapi/v0.0.37
slurmrestd: openapi/dbv0.0.36
slurmrestd: openapi/dbv0.0.37
slurmrestd: openapi/v0.0.36
```

Note the highest value, here `v0.0.37`, which is part of the base URI `http://localhost:8080/slurm/v0.0.37`.

### Examples

```console
curl --header "X-SLURM-USER-NAME: slurm" --header "X-SLURM-USER-TOKEN: $SLURM_JWT" 'http://localhost:8080/slurm/v0.0.37/ping'
```

```console
{
   "meta": {
     "plugin": {
       "type": "openapi\/v0.0.37",
       "name": "Slurm OpenAPI v0.0.37"
     },
     "Slurm": {
       "version": {
         "major": 21,
         "micro": 6,
         "minor": 8
       },
       "release": "21.08.6"
     }
   },
   "errors": [
   ],
   "pings": [
     {
       "hostname": "slurmctld",
       "ping": "UP",
       "status": 0,
       "mode": "primary"
     }
   ]
 }
```

```console
curl --header "X-SLURM-USER-NAME: slurm" --header "X-SLURM-USER-TOKEN: $SLURM_JWT" --header "Content-Type: application/json" -d @example_job.json 'http://localhost:8080/slurm/v0.0.37/job/submit'
```
```console
{
   "meta": {
     "plugin": {
       "type": "openapi\/v0.0.37",
       "name": "Slurm OpenAPI v0.0.37"
     },
     "Slurm": {
       "version": {
         "major": 21,
         "micro": 6,
         "minor": 8
       },
       "release": "21.08.6"
     }
   },
   "errors": [
   ],
   "job_id": 1,
   "step_id": "BATCH",
   "job_submit_user_msg": ""
 }
```

### See logs

See logs from slurmrestd.

```console
docker-compose logs -f slurmctld
```

See logs from slurmctld.

```console
docker exec slurmctld tail -f /var/log/slurm/slurmd.log
```

See the completed jobs.

```console
docker exec slurmctld tail -f /var/log/slurm/jobcomp.log
```

### See also

- [slurmrestd](https://slurm.schedmd.com/slurmrestd.html)
- [Slurm REST API](https://slurm.schedmd.com/rest.html)
- [Slurm REST API Reference](https://slurm.schedmd.com/rest_api.html)

## Stopping and Restarting the Cluster

```console
docker-compose stop
docker-compose start
```

## Deleting the Cluster

To remove all containers and volumes, run:

```console
docker-compose stop
docker-compose rm -f
docker volume rm slurm-docker-cluster_etc_munge slurm-docker-cluster_etc_slurm slurm-docker-cluster_slurm_jobdir slurm-docker-cluster_var_lib_mysql slurm-docker-cluster_var_log_slurm
```
