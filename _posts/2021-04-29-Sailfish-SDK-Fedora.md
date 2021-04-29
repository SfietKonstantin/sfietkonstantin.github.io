---
layout: post
title: Sailfish OS and Fedora 33
---

TL;DR: Do not use the Docker-based Sailfish SDK on an out-of-the-box installation of Fedora. Either use the VirtualBox 
based setup or use Docker after enabling cgroups v1 compatibility. 

If you really want to try to use Podman, you can try the "wrapper script" from 
[this repository](https://github.com/SfietKonstantin/sailfishos-sdk-rust-hacks/tree/main/sdk-podocker).

-----

This blog post covers my many attempts to use the Sailfish SDK on Fedora 33 to build a Rust-based application.

Spoiler alert: it didn't went well :confused:.

## The Fedora experience

Long time ago, after distro-hopping from Mandrivas, Ubuntus and OpenSUSE, I have settled on using Fedora for my daily 
driver. It is a nice distro, being both reliable and innovative. I'm trying to not tweak it's out-of-the-box 
configuration, to "benefit" from all the features that are shipped with it.

While most of these features were welcoming or at least invisible, some of them are more controversial for end-users.
For example, I still don't see the use of SELinux, that often get into my way.

But today, we are going to talk about another of these kind of features, introduced by Fedora 31: cgroups v2. While I
don't know what it brings (and didn't read about it), this change made Docker unavailable for Fedora [^1]. 
Fortunately, [podman](https://podman.io/) is proposed as a drop-in replacement. The project website even recommend 
aliasing `docker` to `podman` !

After using Podman for some time, I was ready to state that it was indeed a good replacement for Docker, until that day 
where I wanted to (re)use the Sailfish SDK to build some Rust code.

## The not-so drop-in replacement

The Sailfish SDK packages a cross-compiler, Scratchbox 2, inside an OS image. The SDK boots this image to start the 
"build engine". It can either use Virtual Box or Docker. When the build engine is started, the SDK communicates with it
with SSH.

Armed with my trust in Podman, I created a symbolic link from `podman` to `docker` and installed the SDK by choosing 
the Docker option. The installation went well but unfortunately, inside the SDK, the build engine could not be started.

{% 
  include image.html 
  url="/images/2021-04-29/sailfish-sdk-no-vm.png" 
  alt="An error message about virtual machine sailfish-os-build-engine that cannot be found" 
  description="I'm pretty sure my installation went well ..."
%}

After looking at the logs [^2] and into the code [^3], I found out that the SDK is trying to find an image named 
`sailfish-os-build-engine` and, when found, starting a container with the same name. This works fine with Docker, but
Podman prefixes all local images with `localhost`. On my machine, the image is then named 
`localhost/sailfish-os-build-engine`.

I tried to cheat a bit and configure a build engine named `localhost/sailfish-os-build-engine`, but this didn't worked
either. The SDK will always use the image name for the container name and slashes are forbidden in container 
names.

After countless hours searching for various solution on the Internet, in the hope of finding the magic switch that 
disable this `localhost` prefix, I ended by concluding that this is impossible [^4]. Podman isn't a drop-in replacement 
for Docker after all.

I still had a way to make the SDK work. By introducing a little change in the SDK, I could remove the prefix when 
listing images. This should make the SDK compatible with Podman. After a rebuild of Qt Creator and a tentative 
pull-request, my patched SDK was ready.

## A bus problem

The rebuilt SDK allowed me to start the engine, but didn't seems to acknowledge that it started. Trying to SSH also
lead to an error.

```shell
$ ssh -p 2222 -i ~/Dev/SailfishOS/SDK/vmshare/ssh/private_keys/engine/mersdk mersdk@localhost 
kex_exchange_identification: read: Connection reset by peer
Connection reset by ::1 port 2222
```

The engine seems to reject SSH connections, so as a last resort, I executed a shell directly in the container ...

```shell
$ podman exec -it sailfish-os-build-engine /bin/bash 
Failed to connect to bus: No such file or directory

```
... and discovered another class of issues.

Remember that the Sailfish OS SDK build engine run either as a VM or a container. When running in a VM, a fully
fledged Linux distribution is booted, including a SSH server and a web-service. 

The Docker based build engine should behave similarly as the VM. In fact, the entrypoint of the container is 
`/sbin/init` ie `systemd`, so when the container is started, `systemd` will be in charge of initializing various daemons 
in the container, including the SSH server.

The bus error I'm receiving seems to indicate that the container could not interact with systemd running on host. I'm 
suspecting that the change to cgroups v2 is still causing trouble here. Enabling cgroups v1 compatibility seems to fix
this issue as reported by 
[this thread](https://forum.sailfishos.org/t/issues-with-installing-sailfish-sdk-docker-with-debian-bullseye-11).

## Introducing second order hacks

In the meantime, SDK developers suggested to use an alternative docker executable. In fact, the docker command to
be used can be configured via an ini file.

```ini
# <sdk-root>/share/qtcreator/SailfishSDK/libsfdk.ini

# Set the path to your docker executable
DockerPath=/home/sk/Dev/SailfishOS/sdk-podocker/target/debug/sdk-podocker
```

So instead of changing how the SDK lists images or starts container, I could simply write a script that wraps Docker or
Podman. This script would be in charge of

- Patching the output of `podman images`
- Start the SSH server inside the build engine container after `podman start`

I have written a very small Rust program, located in
[this repository](https://github.com/SfietKonstantin/sailfishos-sdk-rust-hacks/tree/main/sdk-podocker). You can try it
but, as you will see later, I do not recommend using it as it didn't resolve all my issues.

## Will it build ?

When using the wrapper script, the SDK finally recognizes the build engine and could start to build. However, I faced
a lot of permissions errors when writing to the host's file system from inside the container.

Podman in a rootless mode performs id mapping: files on the host file system and mapped using volumes will appear as
owned by `root` inside the container. And when root writes files on the mapped volumes, they will be owned by the host's
(non-root) user. The build engine do not use root for most of it's operations. Instead it is using the `mersdk` user,
that do not have permission to write in `root`-owned folders.

While this permissions issue is still something manageable [^5], I feel that the giant pile of hacks I'm starting to 
build didn't feel sustainable. But the final nail in the ~~coffin~~ container came with Rust: when following this
[forum.sailfishos.org](https://forum.sailfishos.org/t/rust-howto-request/3187) thread to install Rust,
I managed to break my build engine [^6].

## Conclusion

{%
include imgur.html
id="rQIb4Vw"
description="Replacing a lightbulb"
%}

At start, I simply wanted to build some Rust code. But I got carried away quite far. In the meantime, we have seen
the differences between Docker and Podman, Podman specificities and how the Sailfish SDK works.

For Fedora users, I recommend to stick with Virtual Box based SDK. You can also try to using Docker and reenable 
cgroups v1 by passing`systemd.unified_cgroup_hierarchy=false` to the kernel command line. I do not recommend using
Podman for the SDK [^7].

-----

[^1]: I have learned today that Docker is now compatible with cgroups v2. I might revisit one day my decision to live 
      without Docker.

[^2]: I take this opportunity to give Kudos to Jolla developers for caring and improving their SDK. I was amazed by
      the amount of changes since the last time I used it ! Some highlights: Docker support and the CLI. Running the
      CLI with debug logs helped me a lot.

[^3]: [https://github.com/sailfishos/sailfish-qtcreator](https://github.com/sailfishos/sailfish-qtcreator)

[^4]: Comment in [this pull-request](https://github.com/containers/buildah/issues/1034) suggest that the prefix is 
      intended and cannot be disabled.

[^5]: `chmod 777` is still an option.

[^6]: More on this on a follow-up article.

[^7]: I do recommend using Podman for containers though. The daemon-less, root-less design is really nice, but simply
      doesn't work with the Sailfish SDK.
