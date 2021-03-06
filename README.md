# Private Contact Discovery Service (Beta)

The private contact discovery micro-service allows clients to discover which of their
contacts are registered users, but does not reveal their contacts to the service operator
or any party that may have compromised the service.

## Building the SGX enclave (optional)

### Building reproducibly with Docker

#### Prerequisites:
- GNU Make
- Docker (able to run debian image)

`````
$ make -C <repository_root>/enclave
`````

The default docker-install target will create a reproducible build environment image using
enclave/Dockerfile, build the enclave inside a container based on the image, and install
the resulting enclave and jni libraries into service/src/main/resources/. The Dockerfile
will download a stock debian Docker image and install exact versions of the build tools
listed in enclave/docker/build-deps. Make will then be run inside the newly built Docker
Debian image as in the [Building with Debian](#building-with-debian) section below:

If you need to update a package in the build environment, remove it from
enclave/docker/build-deps, run `make docker`, and check in the resulting changes to the
build-deps file.

If you need to add a package to the build environment, add it to enclave/debian/control
and repeat the same steps.

### Building with Debian

#### Prerequisites:
- GNU Make
- gcc-6
- devscripts (debian package)
- [Intel SGX SDK v1.9 SDK](https://github.com/01org/linux-sgx/tree/sgx_1.9) build dependencies

`````
$ make debuild derebuild
`````

`debuild` is a debian tool used to build debian packages after it sanitizes the
environment and installs build dependences. The primary advantage of using debian
packaging tools in this case is to leverage the [Reproducible
Builds](https://wiki.debian.org/ReproducibleBuilds) project. While building a debian
package, `debuild` will record the names and versions of all detected build dependencies
into a *.buildinfo file. The Reproducible Builds Project's `derebuild.pl` script can then
read the buildinfo file to drill down in the [Debian Snapshot
Archive](http://snapshot.debian.org/) to output the list of packages and generate an apt
sources.list which should contain all of those packages. The list of packages should then
be checked in as build-deps in the enclave/docker/ folder, along with sources.list and
buildinfo, which will then be used to reproduce the build when running `make docker`
again in the future.

The `debuild` target also builds parts needed from the Intel SGX SDK v1.9 after cloning it
from github.

### Building without Docker or Debian:

#### Prerequisites:
- GNU Make
- gcc-6
- [Intel SGX SDK v1.9 SDK](https://github.com/01org/linux-sgx/tree/sgx_1.9) (or its build dependencies)

`````
$ make -C <repository_root>/enclave all install
`````

The `all` target will probably fail to reproduce the same binary as above, but doesn't
require Docker or Debian Linux.

If SGX_SDK_DIR, or SGX_INCLUDEDIR and SGX_LIBDIR, are not specified, the Intel SGX SDK
v1.9 will be cloned from github and any required libraries will be built. The SDK build
prerequisites should be present in this case.

The `install` target copies the enclave and jni libraries to service/src/resources/, which
should potentially be checked in to be used with the service.

NB: the installed enclave will be signed with SGX_FLAGS_DEBUG enabled by an automatically
generated signing key. Due to Intel SGX licensing requirements, a debug enclave can
currently only be run with the SGX debug flag enabled, allowing inspection of its
encrypted memory, and invalidating its security properties. To use an enclave in
production, the generated libsabd-enclave.signdata file must be signed using a signing key
whitelisted by Intel, which can then be saved as libsabd-enclave.sig with public key at
libsabd-enclave.pub, and signed using `make signed install`.

## Building the service

`````
$ cd <repository_root>
$ mvn package
`````

## Running the service

### Runtime requirements:
- [Intel SGX SDK v1.9 PSW](https://github.com/01org/linux-sgx/tree/sgx_1.9#install-intelr-sgx-psw)

`````
$ cd <repository_root>
$ java -jar service/target/contactdiscovery-<version>.jar server service/config/yourconfig.yml
`````

