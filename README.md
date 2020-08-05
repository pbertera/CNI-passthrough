# CNI Passtrough

CNI `passthrough` is a CNI plugin that do nothing but logging all the information received via stdin and environment variables.
The `passthrough` plugin is useful for learning and debugging purposes; since the plugin doesn't create any network must be used chained to other real [plugins](https://github.com/containernetworking/plugins)

The plugin supports the following json coinfiguration fields:

* `type`: must be `passthrough` 
* `logFile`: the path of the log file, the field can contain environment variable names (eg. `/tmp/debug0${CNI_CONTAINERID}_${CNI_COMMAND}`)
* `hookADD`: a list of commands executed on the host via `eval` during the **ADD** action
* `hookDEL`: a list of commands executed on the host via `eval` during the **DEL** action
* `hookVERSION`: a list of commands executed on the host via `eval` during the **VERSION** action
* `hookCHECK`: a list of commands executed on the host via `eval` during the **CHECK** action

## Manually testing the plugin

CNI plugins can be triggered manually withouth a container runtime via the [`cnitool`](https://github.com/containernetworking/cni/tree/master/cnitool) command.

1) Install the `cnitool` command following the [instructions](https://github.com/containernetworking/cni/tree/master/cnitool)
2) create the CNI configuration, the following example defines a network named `testing-net` invoking the following plugins: `bridge -> passthrough -> bridge`. The first instance of the passthrough will log the the CNI data into `/tmp/debug-0_${CNI_CONTAINERID}_${CNI_COMMAND}.log`, the second instance will log into `/tmp/debug-1_${CNI_CONTAINERID}_${CNI_COMMAND}.log`. This is useful to see how the `bridge` plugin changes the CNI data trough the plugin chain.
```
$ cat conf/10-chained.conflist
{
  "cniVersion": "0.4.0",
  "name": "testing-net",
  "plugins": [
    {
      "type": "passthrough",
      "logFile": "/tmp/debug-0_${CNI_CONTAINERID}_${CNI_COMMAND}.log",
    },
    {
      "type": "bridge",
      "bridge": "cni-testing0",
      "isGateway": true,
      "ipMasq": true,
      "hairpinMode": true,
      "ipam": {
        "type": "host-local",
        "routes": [{ "dst": "0.0.0.0/0" }],
        "ranges": [
          [
            {
              "subnet": "10.99.0.0/16",
              "gateway": "10.99.0.1"
            }
          ]
        ]
      }
    },
    {
      "type": "passthrough",
      "logFile": "/tmp/debug-1_${CNI_CONTAINERID}_${CNI_COMMAND}.log",
    }
  ]
}
```
3) create a network namespace:
```
$ sudo ip netns add testing
```
4) add the namespace to the `testing-net` network:
```
$ sudo CNI_PATH=./bin NETCONFPATH=./conf cnitool add testing-net /var/run/netns/testing
```
5) check the generated logs under the `tmp` folder
6) check the namespace connectivity:
```
$ sudo ip -n testing addr
$ sudo ip netns exec testing ping -c 1 8.8.8.8
```
7) clean up:
```
$ sudo CNI_PATH=./bin NETCONFPATH=./conf cnitool del testing-net /var/run/netns/testing
$ sudo ip netns del testing
```

## Using the passthrough plugin with podman

**NOTE:** rootless podman do not use CNI to plumb the container networking, that's why podman should be executed as root here

1) put the plugin into the cni plugin folder (you can find the folder checking the `cni_plugin_dir` parameter in `/usr/share/containers/libpod.conf` or `/etc/containers/libpod.conf`:
```
$ sudo cp passthrough /usr/libexec/cni 
```

2) add the `passthrough` plugin as a chained plugin into podman CNI network configuration:
```
cat /etc/cni/net.d/87-podman-bridge.conflist 
{
  "cniVersion": "0.4.0",
  "name": "podman",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni-podman0",
      "isGateway": true,
      "ipMasq": true,
      "hairpinMode": true,
      "ipam": {
        "type": "host-local",
        "routes": [{ "dst": "0.0.0.0/0" }],
        "ranges": [
          [
            {
              "subnet": "10.88.0.0/16",
              "gateway": "10.88.0.1"
            }
          ]
        ]
      }
    },
    {
      "type": "passthrough",
      "logFile": "/tmp/cni-pre-portmap_${CNI_COMMAND}.log",
      "hookADD": ["/usr/sbin/iptables -L -n -v -t nat", "/usr/sbin/iptables -L -n -v"],
      "hookDEL": ["/usr/sbin/iptables -L -n -v -t nat", "/usr/sbin/iptables -L -n -v"]
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    },
    {
      "type": "passthrough",
      "logFile": "/tmp/cni-post-portmap_${CNI_COMMAND}.log",
      "hookADD": ["/usr/sbin/iptables -L -n -v -t nat", "/usr/sbin/iptables -L -n -v"],
      "hookDEL": ["/usr/sbin/iptables -L -n -v -t nat", "/usr/sbin/iptables -L -n -v"]
    },
    {
      "type": "firewall"
    },
    {
      "type": "tuning"
    }
  ]
}
```

3) create a pod attached to the CNI network mapping the port `8080` on the host.

```
# podman pod create -p 8080:8080 -n test
```

4) spawn a container listeining on the port `8080` into the `test` pod:
```
# podman run --pod test --rm -it registry.redhat.io/rhel8/httpd-24
```

4) check the logs `/tmp/cni-pre-portmap_ADD.log` `/tmp/cni-post-portmap_ADD.log`, you should also see the `iptables` rules changed

5) after removing the pod you should see the iptables rules cleaned up into the files `/tmp/cni-pre-portmap_DEL.log` `/tmp/cni-post-portmap_DEL.log`
