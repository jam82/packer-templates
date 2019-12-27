# packer-templates and monolithic repositories

This repository does not contain any code, just some thoughts on the topic.

Please feel free to comment and share your opinion.

## Some notes

The packer cli is not very handsome when it comes to automation with monolithic repositories that bear lots of different images in it.
It does not seem to be built by a data driven design approach, and you see quite some people struggle with it. Just take a look at the external resources mentionend in the [official docs](https://www.packer.io/community-tools.html#templates) and their often complex structure with a lot of duplicate code.

Ok ok, maybe this was a little too harsh towards packer, because it is not their fault and packer is a cool tool for which I could not find any substitute. Let me say the home turf is the application developer domain and the cli does not make it easy to automate the parallel builds of many images for different hypervisors in different flavours for different operating systems and distributions without a lot of duplicate code or without some management tools around it. You see the cartesian product shining through here? :) It is more intended to be used like... wait for it... docker... just with VMs, ... or to build docker images as well. Sounds strange, hm?

Well, it is not, if you look closer. Packer gives you a single interface for doing so, but that is a different topic.

So if you want to deal with the cartesian problem, you either have to

* write a lot of duplicate code
* use a wrapper to dynamically generate json instructions for packer
* or finally: isolate your images in a myriade of single repositories and be consistent ;-)

Some existing wrappers are mentioned in the [official docs](https://www.packer.io/community-tools.html#templates), but I could not and did not want to wrap my head around one. Some are using their own DSL, some (all I could find) are not supporting all builders I need, i.e. proxmox... I do not say that it is bad work, it just does not fit my needs.

So maybe someday I will build some jinja-based template engine myself for it... but right now I need to pass on... :P

On <cloud.google.com> you can find an [article](https://cloud.google.com/solutions/automated-build-images-with-jenkins-kubernetes?hl=en) that suggest to isolate the different images in theor own repositories and have the builds automated by jenkins. This seems to be the most reasonable approach so far as parallel builds are possible, orchestrated by jenkins. But to be honest, this is not the typical "use-at-home" scenario.

## My thoughts about it

If you want to go with a monolithic repository for the ease of maintenance, and you do not have a cloud infrastructure but only your pc at hand, the best way I could figure out was to do it like the following:

* build a template json file for each builder-type, i.e. qemu.json, proxmox.json, virtualbox.json with your desired default settings for that platform and include variables for the values you want to be flexible
* build var files for all kind of operating systems, distributions and dependant flavours (can be quite a lot...)
* for every flavour you also have a corresponding preseed/kickstart file which you can generalize a bit by passing some settings as kernel arguments to the `boot_command` in the template json file via the var file directly

I went with the separation by [builder](https://www.packer.io/docs/builders/index.html), because you cannot build qemu and virtualbox on the same machine in parallel with hardware virtualization extensions enabled, as they use kernel modules that require exclusive access to the CPUs virtualization extensions...

If you want to build in parallel in an automated manner and make use of [packers parallel functionality](https://www.packer.io/intro/getting-started/parallel-builds.html), you would have to merge all var files and the template json file into one, with a lot of code duplication. The parallel functionality is nice, if you want to build the exact same image for different public cloud providers (except you go on prem with qemu and virtualbox both with hardware virtualization extensions enabled), but not if you want to build the cartesian product of different images for different providers at once.

With merging all flavours in one json file for a builder you could do

```shell
packer build -parallel-builds=4 qemu.json
```

to build all qemu image flavours with a parallel degree of 4.

This is to some point irritating to me, as it seems a common use case out there where a lot of people struggle with, especially when you look at the official docs and again the [external community references]((https://www.packer.io/community-tools.html#templates)) there.

## Examples for monolithc approach

Here are some examples for the hands on usage with monolithic repositories described earlier.

If you want to build a default Ubuntu 18.04 Server image for qemu use:

```shell
packer build -var-file=var/ubuntu-1804-server.json qemu.json
```

This would result in an Ubuntu 18.04 Server default image.

For example a samba dc with special partition layout as proxmox template would be built like:

```shell
packer build -var-file=var/ubuntu-1804-vm-hwe-samba.json proxmox.json
```

This would result in an Ubuntu 18.04 Server minmal image with virtual  hwe kernel, just like selecting the two options F4 => "Install a minimal VM" and then selecting "Install Ubuntu Server with the [HWE](https://wiki.ubuntu.com/Kernel/LTSEnablementStack) kernel"

## Conclusion

If I was too blind to figure out a better way to do it, please let me know.

I personally decided to use a distinct repository for each image, which makes versioning the individual images a bit easier. But it does not help with code duplication...
