# Performance tuning

## In short

For large disks read over the network transports (NBDSSL and NBD), two settings make the biggest difference: read
each disk over **several NFC sessions**, and ask the host for **larger transfer buffers**. Read-ahead is already on by
default. HotAdd needs no tuning.

Put these two lines in a configuration file:

```
OpenVDDK.nbd.sessions = "4"
vixDiskLib.nfcAio.Session.BufSizeIn64KB = "32"
```

and pass that file to the library (examples below). Every setting is described in the kit's README, section
"Configuration".

Please measure in your own environment: storage, network and CPU decide what you get.

## What the settings do

| Setting | Default | Effect |
|---|---|---|
| `OpenVDDK.nbd.sessions` | `1` | Opens each disk in this many NFC sessions (1 to 16) and spreads the reads over them. VMware's library has no such option. |
| `vixDiskLib.nfcAio.Session.BufSizeIn64KB` | host default (64 KiB) | VMware's own key: the size of the host's buffer per session, in 64 KiB units (`32` = 2 MiB). |
| Read-ahead (`OpenVDDK.nbd.readAheadKB`, `OpenVDDK.nbd.readAheadDepth`) | on: 2 MiB × 8 per disk | Serves sequential readers from memory, also tools that send one request at a time. Nothing to set. |

## Example 1: virt-v2v

Point virt-v2v at the OpenVDDK kit and add the configuration file:

```
virt-v2v -ic 'vpx://user@vcenter/Datacenter/cluster/host?no_verify=1' \
  -it vddk \
  -io vddk-libdir=/opt/openvddk-evalkit \
  -io vddk-thumbprint=XX:XX:...:XX \
  -io vddk-transports=nbdssl \
  -io vddk-config=/etc/openvddk/openvddk.conf \
  -o local -os /var/tmp/out \
  VM-NAME
```

## Example 2: nbdkit

Add `config=` to the parameters you already pass to nbdkit's VDDK plugin (server, user, thumbprint, the VM, the
snapshot and the disk, as the plugin's manual describes them):

```
nbdkit -r vddk libdir=/opt/openvddk-evalkit transports=nbdssl \
  config=/etc/openvddk/openvddk.conf \
  ...your usual connection and disk parameters...
```

Your own program passes the file as the last argument of `VixDiskLib_InitEx`.

Apache CloudStack's VMware import builds the virt-v2v command itself. Whether your CloudStack version lets you add a
VDDK configuration file is not checked here. Without one, OpenVDDK's read-ahead still applies by itself.

## What we measured

The measured read rates for each of these settings, against VMware's library on the same worker and disk, are on
[openvddk.com](https://openvddk.com/#results) (section "One disk over several NFC sessions"). The data read was
identical with both libraries.

## The host's limit: plan how many disks run at once

Every NFC session uses part of the ESXi host's NFC memory, with either library.

- With 2 MiB buffers, each session takes about 4 MiB. A default host has 96 MiB.
- In our lab tests, a default host served 16 such sessions and refused the 24th, with either library. With default
  buffers, 40 readers at once ran fine.
- So with 4 sessions and 2 MiB buffers, plan on **about 4 disks at once per ESXi host**. For more disks at once, use
  fewer sessions or the default buffers.
- The host's NFC memory can be raised: on ESXi 7 and 8, `maxMemory` under `nfcsvc` in hostd's `config.xml`
  (VMware KB 389223); on ESXi 9, the ConfigStore setting `/plugins/nfcsvc/max_memory`. A hostd restart is needed. In
  a lab test, doubling it doubled the sessions the host served. Changing a host setting is the host administrator's decision.

When the host refuses a session, the read fails with VixError 14009 in OpenVDDK (VMware's library reports error 1 for
the same refusal). Readers that already run are not affected.
