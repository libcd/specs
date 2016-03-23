This document defines the libcd intermediate representation (ir). This intermediate representation is used to represent complex container workflows in a simple tree structure that is agnostic to the runtime implementation.

The reference implementation and runtime for this specification can be found at [https://github.com/libcd/libcd](https://github.com/libcd/libcd).

* [file format](#format)
* [file structure](#structure)
  * [volumes](#volumes)
  * [objects](#objects)
  * [nodes](#nodes)
      * [root](#rootnode)
      * [list](#listnode)
      * [parallel](#parallelnode)
      * [recover](#recovernode)
      * [defer](#defernode)
      * [run](#runnode)
  * [example](#example)

## Format

The intermediate representation is encoded as a JSON document.

## Structure

The intermediate representation has three different top-level sections:

name    | desc
--------|-------
volumes | container volumes that should be created at runtime
objects | container definitions and runtime configurations
nodes   | tree representation of the container execution process

This is an example representation of the base structure:

```json
{
    "objects": [],
    "volumes": [],
    "nodes": {}
}
```

### Volumes

The volumes section defines a list of volumes that should be created when the runtime starts. These volumes can be shared by on or many containers at runtime by explicitly referencing the volume name in the container definition.

name   | desc
-------|-------
name   | unique name for the volume
driver | volume driver, defaults to local

This is an example representation of the volumes section:

```json
{
    "volumes": [
        {
            "driver": "local",
            "name": "volume_0"
        }
    ]
}
```

This is an example of how volumes are referenced in container definitions:

```json
{
    "objects": [
        {
            "image": "golang",
            "volumes": [
                "volume_0:/path/inside/container",
            ]
        }
    ]
}
```

### Objects

The objects section defines a list of containers that are referenced by the execution tree. The container object has the following properties:

name             | type     | required | desc
-----------------|----------|----------|-----
name             | string   | yes      | unique name for the container definition
image            | string   | yes      | name of the container image
pull             | bool     | no       | pull the latest image from the registry
auth_config      | struct   | no       | pull the image with these registry credentials
detach           | bool     | no       | run the container in the background. does not block
privileged       | bool     | no       | run the container in privileged mode
environment      | []string | no       | container environment variables
entrypoint       | []string | no       | container entrypoint
command          | []string | no       | container command
extra_hosts      | []string | no       |
volumes          | []string | no       |
volumes_from     | []string | no       |
devices          | []string | no       |
network_mode     | string   | no       |
dns              | []string | no       |
dns_search       | []string | no       |
memswap_limit    | int64    | no       |
mem_limit        | int64    | no       |
cpu_quota        | int64    | no       |
cpu_shares       | int64    | no       |
cpuset           | string   | no       | comma separated list of cpus, ie 0,1,2
oom_kill_disable | bool     |  no      | disables oom kill

The auth config has the following properties:

name      | type     | desc
----------|----------|----------
username  | string   | registry username
password  | string   | registry password
email     | string   | registry email (optional as of docker 1.11)
token     | string   | registry auth token, alternate to username / password

This is an example representation of the objects section:

```json
{
    "objects": [
        {
            "name": "container_0",
            "image": "golang:1.5"
        }
    ]
}
```

This is an example of how objects are referenced in the execution tree:

```json
{
    "nodes": [
        {
            "type": "run",
            "name": "container_0"
        }
    ]
}
```

## Nodes

The nodes section defines a list of containers that are referenced by the execution tree. This section of the documentation defines the tree structure and node elements in the intermediate representation.

This is a simple example hello-world configuration:

```json
{
    "nodes": [
        {
            "type": "list",
            "body": [
                {
                    "name": "container_0",
                    "type": "run"
                }
            ]
        }
    ],
    "objects": [
        {
            "name": "container_0",
            "image": "busybox",
            "entrypoint": [ "/bin/bash", "-c" ],
            "command": [ "echo hello world"]
        }
    ]
}
```

### RootNode

The root node must be of type ListNode defined in the next section.

### ListNode

The list node defines a list of nodes that are executed sequentially. If an error is encountered at runtime the node exits immediately and returns the error to the parent; pending sibling nodes are not executed.

The list node has the following properties:

name  | type     | desc
------|----------|----------
type  | string   | must be set to 'list'
body  | []node   | list of child nodes

This is an example representation of the list node:

```json
{
    "type": "list",
    "body": [
        {
            "name": "container_0",
            "type": "run"
        }
    ]
}
```

### ParallelNode

The parallel node defines a list of nodes that are executed in parallel. If an error is encountered at runtime the node returns the error to the parent. Unlike the list node the sibling nodes are executed even when an error is encountered, due to the async nature of this step.

The parallel node has the following properties:

name  | type     | desc
------|----------|----------
type  | string   | must be set to 'parallel'
body  | []node   | list of child nodes to execute in parallel
limit | int      | limit the parallel execution to N concurrent nodes

This is an example representation of the parallel node:

```json
{
    "type": "parallel",
    "limit": 2,
    "body": [
        {
            "name": "container_0",
            "type": "run"
        }
        {
            "name": "container_1",
            "type": "run"
        }
        {
            "name": "container_2",
            "type": "run"
        }
    ]
}
```

### RecoverNode

The recover node wraps a node and prevents errors from bubbling up to the parent node. If an error is encountered it is ignored and will not terminate the process.

The recover node has the following properties:

name  | type     | desc
------|----------|----------
type  | string   | must be set to 'recover'
body  | node     | child node that should recover from failure

This is an example representation of the recover node:

```json
{
    "type": "recover",
    "body": {
        "name": "container_0",
        "type": "run"
    }
}
```

### DeferNode

The defer node executes a primary child node. When the primary child node execution completes, it executes a deferred node. The deferred node is guaranteed to execute even if the primary child node fails, but will still return the error to the parent.

The defer node has the following properties:

name  | type     | desc
------|----------|----------
type  | string   | must be set to 'defer'
body  | node     | child node to execute
defer | node     | child node to execute after the body node

This is an example representation of the defer node:

```json
{
    "type": "defer",
    "body": {
        "name": "container_0",
        "type": "run"
    },
    "defer": {
        "name": "container_1",
        "type": "run"
    }
}
```

### RunNode

The run node is a leaf node that executes a container (ie `docker run`). The run node includes the name of the container definition, which is used to lookup the container definition from the objects section of the document. If the container exits with a non-zero status code an error is returned to the parent node.

If the container definition specifies the container should be run in detached mode, the container is started and sent to the background (non-blocking) and the exit code is ignored.

The container node has the following properties:

name  | type     | desc
------|----------|----------
type  | string   | must be set to 'run'
name  | string   | name of the container definition

This is an example representation of the container node:

```json
{
    "type": "run",
    "node": "container_0"
}
```

## Example

This is an example intermediate representation file (this will not actually run, though). It will execute a golang and node container in parallel, and once both complete, regardless of failure, will post a message to a hipchat room.

```json
{
    "objects": [
        {
            "name": "container_0",
            "image": "golang:1.5",
            "command": [ "/bin/sh", "-c", "go get; go build; go test"]
        },
        {
            "name": "container_1",
            "image": "node:5.0.0",
            "command": [ "/bin/sh", "-c", "npm install; npm test"]
        },
        {
            "name": "container_2",
            "image": "hipchat/hipchat-cli",
            "environment": {
                "TOKEN": "a45728beac6102",
                "ROOM": "watercooler"
            },
            "command": [
                "/bin/sh",
                "-c",
                "echo 'finished build' | ./hipchat_room_message -t $TOKEN -r $ROOM"
            ]
        }
    ],
    "nodes": {
        "type": "list",
        "body": [
            {
                "type": "defer",
                "body": {
                    {
                        "type": "parallel",
                        "body": [
                            {
                                "type": "run",
                                "name": "contianer_0"
                            },
                            {
                                "type": "run",
                                "name": "contianer_1"
                            }
                        ]
                    }
                },
                "defer": {
                    "type": "run",
                    "name": "contianer_2"
                }
            }
        ]
    }
}
```
