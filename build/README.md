# Building Kubernetes

To build Kubernetes you need to have access to a Docker installation through either of the following methods:

## Requirements

1. Be running Docker.  2 options supported/tested:
  1. **Mac OS X** The best way to go is to use `boot2docker`.  See instructions [here](https://docs.docker.com/installation/mac/).
  2. **Linux with local Docker**  Install Docker according to the [instructions](https://docs.docker.com/installation/#installation) for your OS.  The scripts here assume that they are using a local Docker server and that they can "reach around" docker and grab results directly from the file system.
2. Have python installed.  Pretty much it is installed everywhere at this point so you can probably ignore this.
3. *Optional* For uploading your release to Google Cloud Storage, have the [Google Cloud SDK](https://developers.google.com/cloud/sdk/) installed and configured.

## Overview

While it is possible to build Kubernetes using a local golang installation, we have a build process that runs in a Docker container.  This simplifies initial set up and provides for a very consistent build and test environment.

There is also early support for building Docker "run" containers

## Key scripts

* `make-binaries.sh`: This will compile all of the Kubernetes binaries
* `run-tests.sh`: This will run the Kubernetes unit tests
* `run-integration.sh`: This will build and run the integration test
* `make-cross.sh`: This will make all cross-compiled binaries (currently just `kubecfg`)
* `copy-output.sh`: This will copy the contents of `_output/build` from any remote Docker container to the local `_output/build`.  Right now this is only necessary on Mac OS X with `boot2docker`.
* `make-clean.sh`: Clean out the contents of `_output/build` and remove any local built container images.
* `shell.sh`: Drop into a `bash` shell in a build container with a snapshot of the current repo code.
* `release.sh`: Build everything, test it, and (optionally) upload the results to a GCS bucket.

## Releasing

The `release.sh` script will build a release.  It will build binaries, run tests, (optionally) build runtime Docker images and then (optionally) upload all build artifacts to a GCS bucket.

The main output is a tar file: `kubernetes.tar.gz`.  This includes:
* Cross compiled client utilities.
* Script (`cluster/kubecfg.sh`) for picking and running the right client binary based on platform.
* Examples
* Cluster deployment scripts for various clouds
* Tar file containing all server binaries
* Tar file containing salt deployment tree shared across multiple cloud deployments.

In addition, there are some other tar files that are created:
* `kubernetes-client-*.tar.gz` Client binaries for a specific platform.
* `kubernetes-server-*.tar.gz` Server binaries for a specific platform.
* `kubernetes-salt.tar.gz` The salt script/tree shared across multiple deployment scripts.

The release utilities grab a set of environment variables to modify behavior.  Arguably, these should be command line flags:

Env Variable | Default | Description
-------------|---------|------------
`KUBE_SKIP_CONFIRMATIONS` | `n` | If `y` then no questions are asked and the scripts just continue.
`KUBE_GCS_UPLOAD_RELEASE` | `n` | Upload release artifacts to GCS
`KUBE_GCS_RELEASE_BUCKET` | `kubernetes-releases-${project_hash}` | The bucket to upload releases to
`KUBE_GCS_RELEASE_PREFIX` | `devel` | The path under the release bucket to put releases
`KUBE_GCS_MAKE_PUBLIC` | `y` | Make GCS links readable from anywhere
`KUBE_GCS_NO_CACHING` | `y` | Disable HTTP caching of GCS release artifacts.  By default GCS will cache public objects for up to an hour.  When doing "devel" releases this can cause problems.
`KUBE_BUILD_RUN_IMAGES` | `n` | *Experimental* Build Docker images for running most server components.
`KUBE_GCS_DOCKER_REG_PREFIX` | `docker-reg` | *Experimental* When uploading docker images, the bucket that backs the registry.

## Basic Flow

The scripts directly under `build/` are used to build and test.  They will ensure that the `kube-build` Docker image is built (based on `build/build-image/Dockerfile`) and then execute the appropriate command in that container.  If necessary (for Mac OS X), the scripts will also copy results out.

The `kube-build` container image is built by first creating a "context" directory in `_output/images/build-image`.  It is done there instead of at the root of the Kubernetes repo to minimize the amount of data we need to package up when building the image.

Everything in `build/build-image/` is meant to be run inside of the container.  If it doesn't think it is running in the container it'll throw a warning.  While you can run some of that stuff outside of the container, it wasn't built to do so.

When building final release tars, they are first staged into `_output/release-stage` before being tar'd up and put into `_output/release-tars`.

## Runtime Docker Images

This support is experimental and hasn't been used yet to deploy a cluster.

The files necessarily for the release Docker images are in `build/run-images/*`.  All of this is staged into `_output/images` similar to build-image.  The `base` image is used as a base for each of the specialized containers and is generally never pushed to a shared repository.

If the release script is set to upload to GCS, it'll do the following:
* Start up a local `google/docker-registry` registry that is backed by GCS.
* Rename/push the runtime images to that registry.

## TODOs

These are in no particular order

* [ ] Harmonize with scripts in `hack/`.  How much do we support building outside of Docker and these scripts?
* [ ] Get a cluster up and running with the Docker images.  Perhaps start with a local cluster and move up to a GCE cluster.
* [ ] Implement (#186)[https://github.com/GoogleCloudPlatform/kubernetes/issues/186].  This will make it easier to develop Kubernetes.
* [ ] Deprecate/replace most of the stuff in the hack/
* [ ] Create an install script that'll let us do a `curl https://[URL] | bash` to get that tarball down and ensure that other dependencies (cloud SDK?) are installed and configured correctly.
* [ ] Support/test Windows as a client.
* [ ] Finish support for the Dockerized runtime. Issue (#19)[https://github.com/GoogleCloudPlatform/kubernetes/issues/19].  A key issue here is to make this fast/light enough that we can use it for development workflows.
* [ ] Support uploading to the Docker index instead of the GCS bucket.  This'll allow easier installs for those not running on GCE
