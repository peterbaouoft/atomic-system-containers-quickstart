# atomic-system-containers-quickstart
A quick start "guide" thrown together to test out system containers with atomic, mostly for personal reference

For an overview on system containers, go to: http://scrivano.org/static/system-containers-demo

On fedora 23:
- Runc 1.0 release is required, and can be found [HERE](https://github.com/opencontainers/runc)
- The requirements for atomic can be found [HERE](http://pkgs.fedoraproject.org/cgit/rpms/atomic.git/tree/atomic.spec)
- Upstream atomic repo is [HERE](https://github.com/projectatomic/atomic)
- An ostree repo must be set up to store images (should be in place by default)

Note that at the moment (July 13), runc 1.0 is not completely integrated into atomic yet. But it should not cause an issue with the rest of this document.

## System Container Examples:

#### Etcd container

Directly from the repo, one can: 'atomic install --system --name=etcd-system jerzhang/spc-etcd'

which will pull the pre-built etcd image from [docker hub](https://hub.docker.com/r/jerzhang/spc-etcd/). Note that the name "etcd-system" is required for the flannel container, as by default, flannel will attempt to use etcd-system.service (This should probably become an option in the future, such as 'atomic install --system --set=etcd=etcd-system').

You can check the status of the container with 'atomic ps', or 'systemctl status etcd-system'

To stop and remove the container, you can directly use 'atomic uninstall etcd-system' (stop doesn't work atm). Don't do yet this if you want to test out flannel as well.

#### Flannel container

The etcd container (or an etcd service) must be running. If you are running the above Etcd container, you can set the network config as such: 'runc exec etcd etcdctl set /atomic.io/network/config '{"Network":"172.17.0.0/16"}''

Again from the repo, one can: 'atomic install --system --name=flannel jerzhang/spc-flannel'. You can check the status of the container with 'atomic ps', or 'systemctl status flannel', or 'ifconfig | grep flannel'.

Similarily, 'atomic uninstall flannel' cleans it up.

#### Helloworld

All this does is when you 'curl localhost:8081' (You can change that with --set), it will respond with a "Hi world". You can build it directly with: 'atomic install --system --name=helloworld jerzhang/spc-helloworld'

One can also play around with parameters, such as '--set=RECEIVER=Jerry', and it will output "Hi Jerry" instead.

Again, 'atomic uninstall helloworld' stops and removes the container.


Note that for the above 3 containers, the "name" field is optional. But currently we don't actually have IDs associated with containers, so the ID is just the name, and the default names would look something like jerzhang-spc-helloworld-service. Giving a --name parameter helps quite a bit to keep track of containers.

## Building an Image

The images from above can be viewed with either 'docker images' or 'atomic images'. You'll notice that for running containers, the corresponding atomic image has a ">" next to it.

The above containers can be found at https://github.com/giuseppe/atomic-oci-containers. Here I would like to point out that the main difference between runc 1.0 and previous versions is how 'runc run' and 'runc start' works. Functionality that was in 'start' is now in 'run' (as of 1.0), and this is reflected in the 'service.template' file's ExecStart.

Once you are satisfied with the config files, you can (for example, with etcd)

'docker build -t etcd .'
'atomic pull docker:etcd  ## the docker: prefix means to pull from the local Docker'
'atomic install --system --name=etcd-system etcd'

Note that you could also set up a docker hub repo and push to that (name the image DOCKER_REPO_NAME/etcd for example). If you don't do either, skopeo will try to pull docker hub by default, and since it does not exist, the install will fail.

To remove a local atomic image, currently you must directly invoke ostree. You can see what images exist with 'ostree refs', and it will look something like 'ociimage/jerzhang/spc-etcd-latest'. To remove said image, you can 'ostree refs --delete ociimage/jerzhang/spc-etcd-latest', and then 'atomic images --prune'. (Currently, am working on making an atomic rmi command for this)

## Troubleshooting (some errors I've ran into)

**My container failed to start (systemctl status shows failed), or immediately exits without an error**

This could be an issue with runc. Remember to check for the correct runc version: 1.0 uses 'runc run', and previous uses 'runc start'.

**My container does not restart, and journal shows that the container exists even though I have already 'atomic uninstall'ed**

Sometimes this happens, and you need to manually delete the container with runc as well (e.g. 'runc delete etcd-system'). Then uninstall/reinstall. Not quite sure how to reproduce.

**I get an error like: [Errno -2] Name or service not known**

Most likely this is a skopeo error, where it is trying to pull from docker hub and failing. View above on building images.

**Atomic uninstall complains "Image xxxxx does not exist"**

Doesn't seem to be an actual error. Will look into at some point.
