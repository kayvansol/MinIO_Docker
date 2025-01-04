# Deploy MinIO Single-Node Multi-Drive On Docker

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/1.webp?raw=true)

`MinIO` is an object storage system released under GNU Affero General Public License, It is API compatible with the Amazon S3 cloud storage service. It is capable of working with unstructured data such as photos, videos, log files, backups, and container images with the maximum supported object size being 50TB.

The procedures on this page cover deploying MinIO in a Single-Node Multi-Drive (SNMD) configuration. SNMD deployments provide drive-level reliability and failover/recovery with performance and scaling limitations imposed by the single node.

For production environments, MinIO strongly recommends using the MinIO Kubernetes Operator to deploy Multi-Node Multi-Drive (MNMD) or “Distributed” Tenants.

<hr />

Storage Requirements

The following requirements summarize the Storage section of MinIO’s hardware recommendations :

Use Local Storage

Direct-Attached Storage (DAS) has significant performance and consistency advantages over networked storage (NAS, SAN, NFS). MinIO strongly recommends flash storage (NVMe, SSD) for primary or “hot” data.

Use XFS-Formatting for Drives

MinIO strongly recommends provisioning XFS formatted drives for storage. MinIO uses XFS as part of internal testing and validation suites, providing additional confidence in performance and behavior at all scales.

MinIO does not test nor recommend any other filesystem, such as EXT4, BTRFS, or ZFS.

Use Consistent Type of Drive

MinIO does not distinguish drive types and does not benefit from mixed storage types. Each pool must use the same type (NVMe, SSD)

Use Consistent Size of Drive

MinIO limits the size used per drive to the smallest drive in the pool.

For example, deploy a pool consisting of the same number of NVMe drives with identical capacity of 7.68TiB. If you deploy one drive with 3.84TiB, MinIO treats all drives in the pool as having that smaller capacity.

Persist Drive Mounting and Mapping Across Reboots

Use /etc/fstab to ensure consistent drive-to-mount mapping across node reboots.

<hr />

Deploy Single-Node Multi-Drive MinIO

The following procedure deploys MinIO consisting of a single MinIO server and a multiple drives or storage volumes.

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/2.webp?raw=true)

1) Pull the Latest Stable Image of MinIO
```
docker pull quay.io/minio/minio
```

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/3.webp?raw=true)

2) Create the Environment Variable File

Create an environment variable file at /etc/default/minio.The MinIO Server container can use this file as the source of all environment variables.

The following example provides a starting environment file:
```
# MINIO_ROOT_USER and MINIO_ROOT_PASSWORD sets the root account for the MinIO server.
# This user has unrestricted permissions to perform S3 and administrative API operations on any resource in the deployment.
# Omit to use the default values 'minioadmin:minioadmin'.
# MinIO recommends setting non-default values as a best practice, regardless of environment.

MINIO_ROOT_USER=myminioadmin
MINIO_ROOT_PASSWORD=minio-secret-key-change-me

# MINIO_VOLUMES sets the storage volumes or paths to use for the MinIO server.
# The specified path uses MinIO expansion notation to denote a sequential series of drives between 1 and 4, inclusive.
# All drives or paths included in the expanded drive list must exist *and* be empty or freshly formatted for MinIO to start successfully.

MINIO_VOLUMES="/data-{1...4}"

# MINIO_OPTS sets any additional commandline options to pass to the MinIO server.
# For example, `--console-address :9001` sets the MinIO Console listen port
MINIO_OPTS="--console-address :9001"
```

create the paths you want to assign to the minio storage :
```
mkdir /mnt/disk1
mkdir /mnt/disk2
mkdir /mnt/disk3
mkdir /mnt/disk4
```

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/4.webp?raw=true)

3) Create and Run the Container
```
docker run -dt                                  \
  -p 9000:9000 -p 9001:9001                     \
  -v /mnt/disk1:/data-1                              \
  -v /mnt/disk2:/data-2                              \
  -v /mnt/disk3:/data-3                              \
  -v /mnt/disk4:/data-4                              \
  -v /etc/default/minio:/etc/config.env         \
  -e "MINIO_CONFIG_ENV_FILE=/etc/config.env"    \
  --name "minio_local"                          \
  quay.io/minio/minio server --console-address ":9001"
```
![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/5.webp?raw=true)

4) Validate the Container Status
```
docker logs minio_local
```

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/6.webp?raw=true)

The command should return output similar to the following:
```
INFO: Formatting 1st pool, 1 set(s), 4 drives per set.
INFO: WARNING: Host local has more than 2 drives of set. A host failure will result in data becoming unavailable.
MinIO Object Storage Server
Copyright: 2015-2024 MinIO, Inc.
License: GNU AGPLv3 - https://www.gnu.org/licenses/agpl-3.0.html
Version: RELEASE.2024-12-18T13-15-44Z (go1.23.4 linux/amd64)

API: http://172.17.0.2:9000  http://127.0.0.1:9000 
   RootUser: myminioadmin 
   RootPass: minio-secret-key-change-me 

WebUI: http://172.17.0.2:9001 http://127.0.0.1:9001  
   RootUser: myminioadmin 
   RootPass: minio-secret-key-change-me 

CLI: https://min.io/docs/minio/linux/reference/minio-mc.html#quickstart
   $ mc alias set 'myminio' 'http://172.17.0.2:9000' 'myminioadmin' 'minio-secret-key-change-me'

Docs: https://docs.min.io
INFO: Exiting on signal: TERMINATED
MinIO Object Storage Server
Copyright: 2015-2025 MinIO, Inc.
License: GNU AGPLv3 - https://www.gnu.org/licenses/agpl-3.0.html
Version: RELEASE.2024-12-18T13-15-44Z (go1.23.4 linux/amd64)

API: http://172.17.0.2:9000  http://127.0.0.1:9000 
   RootUser: myminioadmin 
   RootPass: minio-secret-key-change-me 

WebUI: http://172.17.0.2:9001 http://127.0.0.1:9001  
   RootUser: myminioadmin 
   RootPass: minio-secret-key-change-me 

CLI: https://min.io/docs/minio/linux/reference/minio-mc.html#quickstart
   $ mc alias set 'myminio' 'http://172.17.0.2:9000' 'myminioadmin' 'minio-secret-key-change-me'

Docs: https://docs.min.io
```

5) Connect to the MinIO Service

. MinIO Web Console :

You can access the MinIO Web Console by entering http://192.168.56.156:9001 in your preferred browser. Any traffic to the MinIO Console port on the local host redirects to the container.

Log in with the MINIO_ROOT_USER and MINIO_ROOT_PASSWORD configured in the environment file specified to the container.

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/7.webp?raw=true)

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/8.webp?raw=true)

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/9.webp?raw=true)

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/10.webp?raw=true)

. MinIO CLI (mc) :

You can access the MinIO deployment over a Terminal or Shell using the MinIO Client (mc). See MinIO Client Installation Quickstart for instructions on installing mc.

Create a new alias corresponding to the MinIO deployment. Use a hostname or IP address for your local machine along with the S3 API port 9000 to access the MinIO deployment. Any traffic to that port on the local host redirects to the container.

```
mc alias set kayvan http://192.168.56.156:9000 myminioadmin minio-secret-key-change-me
```

or via access key :

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/11.webp?raw=true)

```
mc alias set kayvan http://192.168.56.156:9000
```

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/12.webp?raw=true)

```
mc alias ls
```

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/13.webp?raw=true)

You can then interact with the container using any mc command :

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/14.webp?raw=true)

make a bucket named play to store your objects in it :
```
mc mb kayvan/play
```

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/15.webp?raw=true)

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/16.webp?raw=true)

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/17.webp?raw=true)

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/18.webp?raw=true)

```
mc od if=b.mp3 of=kayvan/play/moein/baroon.mp3 size=5MiB parts=3
```

The mc od command copies a local file to a remote location in a specified number of parts and part sizes :

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/19.webp?raw=true)

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/20.webp?raw=true)

```
mc get kayvan/play/moein/baroon.mp3 .
```

The mc get command downloads an object from a target S3 deployment to the local file system.

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/21.webp?raw=true)

```
mc mv  b.mp3  kayvan/play/moein/2025/namaz.mp3
```

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/22.webp?raw=true)

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/23.webp?raw=true)

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/24.webp?raw=true)


![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/25.webp?raw=true)

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/26.webp?raw=true)

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/27.webp?raw=true)


![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/28.webp?raw=true)

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/29.webp?raw=true)

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/30.webp?raw=true)

![alt text](https://raw.githubusercontent.com/kayvansol/MinIO_Docker/refs/heads/main/img/31.webp?raw=true)

Thanks for your attention to this article. 🍹

