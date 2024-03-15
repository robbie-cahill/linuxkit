# LinuxKit with bare metal on Packet

[Packet](http://packet.net) is a bare metal hosting provider.

You will need to [create a Packet account] and a project to
put this new machine into. You will also need to [create an API key]
with appropriate read/write permissions to allow the image to boot.

[create a Packet account]:https://app.packet.net/#/registration/
[create an API key]:https://help.packet.net/quick-start/api-integrations

Linuxkit is known to boot on the [Type 0] 
and [Type 1] servers at Packet.
Support for other server types, including the [Type 2A] ARM server,
is a work in progress.

[Type 0]:https://www.packet.net/bare-metal/servers/type-0/
[Type 1]:https://www.packet.net/bare-metal/servers/type-1/
[Type 2A]:https://www.packet.net/bare-metal/servers/type-2a/

The `linuxkit run packet` command can mostly either be configured via
command line options or with environment variables. see `linuxkit run
packet --help` for the options and environment variables.

By default, `linuxkit run` will provision a new machine and remove it
once you are done. With the `-keep` option the provisioned machine
will not be removed. You can then use the `-device` option with the
device ID on subsequent `linuxkit run` invocations to re-use an
existing machine. These subsequent runs will update the iPXE data so
you can boot alternative kernels on an existing machine.

There is an example YAML file for [x86_64](../examples/packet.yml) and
an additional YAML for [arm64](../examples/packet.arm64.yml) servers
which provide both access to the serial console and via ssh and
configures bonding for network devices via metadata (if supported).

For x86_64 builds for Intel servers we strongly recommend adding
`ucode: intel-ucode.cpio` to the kernel section in the YAML. This
updates the Intel CPU microcode to the latest by prepending it to the
generated initrd file. The `ucode` entry is only recommended when
booting on baremetal. It should be omitted (but is harmless) when
building images to boot in VMs.

**Note**: The update of the iPXE configuration sometimes may take some

Remember, your first boot might take some time, and it might not be successful. If you hit return on the console, it should allow you to retry the boot, which typically resolves the issue.

## Boot

LinuxKit on Packet boots the `kernel+initrd` output from moby. This is done through [iPXE](https://help.packet.net/technical/infrastructure/custom-ipxe), and the process requires a script for iPXE. You also need a HTTP server to store your images during the iPXE booting. The `-base-url` option will allow you to specify the URL for the HTTP server, from which `<name>-kernel`, `<name>-initrd.img`, and `<name>-packet.ipxe` can be downloaded during the boot.

If you have your own HTTP server, you can use `linuxkit push packet` to create the files that need to be available, including the iPXE script.

If you do not have a public HTTP server at hand, you can use the `-serve` option. This will create a local HTTP server for you. You can then either run this on another Packet machine, or you can use tunneling tools like [Tunnelmole](https://tunnelmole.com) and [ngrok](https://ngrok.com/).

Here is an example of how to boot the [example](../examples/packet.net) with a local HTTP server:

First, build the yaml file:
```sh
linuxkit build packet.yml
```

Next, run the web server. For this, you can use [Tunnelmole](https://tunnelmole.com/docs), a free and open source tunneling tool or [ngrok](https://ngrok.com), a popular closed source tunneling tool.

To use Tunnelmole, run `tmole 8080` in another terminal window. This should give you http and https URLs (use the https one for better security). In the following command, replace `<Tunnelmole URL>` with the https URL given by `tmole 8080`. 
```sh
PACKET_API_KEY=<API key> PACKET_PROJECT_ID=<Project ID> \
linuxkit run packet -serve :8080 -base-url <Tunnelmole URL> packet
```

Alternatively, for ngrok, you should run `ngrok http 8080` in another terminal window. Your `ngrok` URL is then used in place of `<ngrok URL>` in the following command.
```sh
PACKET_API_KEY=<API key> PACKET_PROJECT_ID=<Project ID> \
linuxkit run packet -serve :8080 -base-url <ngrok URL> packet
```

To boot an `arm64` image for Type 2a machine (`-machine baremetal_2a`), you need to build using this command: 

```sh
linuxkit build packet.yml packet.arm64.yml
```
You then need to un-compress both the kernel and the initrd before booting, as follows:
```sh
mv packet-initrd.img packet-initrd.img.gz && gzip -d packet-initrd.img.gz
mv packet-kernel packet-kernel.gz && gzip -d packet-kernel.gz
```

The LinuxKit image can be booted with either Tunnelmole or ngrok. For Tunnelmole:
```sh
PACKET_API_KEY=<API key> PACKET_PROJECT_ID=<Project ID> \
linuxkit run packet -machine baremetal_2a -serve :8080 -base-url <Tunnelmole URL> packet
```

Or for ngrok: 
```sh
PACKET_API_KEY=<API key> PACKET_PROJECT_ID=<Project ID> \
linuxkit run packet -machine baremetal_2a -serve :8080 -base-url <ngrok URL> packet
```

Alternatively, `linuxkit push packet` will uncompress the kernel and initrd images on `arm` machines (or explicitly via the `-decompress` flag). You can also use the `linuxkit serve` command to start a local HTTP server serving the specified directory.

**Note**: It might take several minutes for a new server to be deployed. If you are on the console, you will be able to see the BIOS and boot messages during this time.

## Console


By default, `linuxkit run packet ...` will connect to the
Packet
[SOS ("Serial over SSH") console](https://help.packet.net/technical/networking/sos-rescue-mode). This
requires `ssh` access, i.e., you must have uploaded your SSH keys to
Packet beforehand.

You can exit the console vi `~.` on a new line once you are
disconnected from the serial, e.g. after poweroff.

**Note**: We also require that the Packet SOS host is in your
`known_hosts` file, otherwise the connection to the console will
fail. There is a Packet SOS host per zone.

You can disable the serial console access with the `-console=false`
command line option.


## Disks

At this moment the Linuxkit server boots from RAM, with no persistent
storage.  We are working on adding persistent storage support on Packet.


## Networking

On the baremetal type 2a system (arm64 Cavium Thunder X) the network device driver does not get autoloaded by `mdev`. Please add:

```
  - name: modprobe
    image: linuxkit/modprobe:<hash>
    command: ["modprobe", "nicvf"]
```

to your YAML files before any containers requiring the network to be up, e.g., the `dhcpcd` container.

Some Packet server types have bonded networks; the `metadata` package has support for setting
these up, and also for adding additional IP addresses.


## Integration services and Metadata

Packet supports [user state](https://help.packet.net/technical/infrastructure/user-state)
during system bringup, which enables the boot process to be more informative about the
current state of the boot process once the kernel has loaded but before the
system is ready for login.
