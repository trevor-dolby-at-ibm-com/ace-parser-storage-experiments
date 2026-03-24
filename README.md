# ace-parser-storage-experiments
App Connect Enterprise (ACE) storage options experiments

By default, ACE flows will return storage to the C++ memory heap when possible, but the
memory is still associated with the DataFlowEngine/IntegrationServer process and cannot
be used by other processes. This is good from a memory caching point of view for a specific
integration server, but can be unhelpful for the system as a whole, and so there are some
new options for ACE servers that allow the behaviour to be changed.

## Overview

The default behaviour as follows (note that the numbers are for illustration only):

![default-memory-usage](/pictures/default-memory-usage.png)

and using several options at the same time can allow for storage to be returned to the
operating system for use by other processes (numbers for illustration only):

![mmap-reduced-storage](/pictures/mmap-reduced-storage.png)

## Environment variables

### MQSI_SYNTAX_ELEMENT_POOL_CHUNK_SIZES

This environment variable is intended to be used to return syntax element storage to the operating
system when the flow is finished with the parsers after a flow run has completed. This is made possible
by increasing the size of the syntax element chunks (other than the first one) to a value large enough
to cause the heap manager to use mmap() on Linux/Unix rather than using the standard heap storage.

Example:
```bash
MQSI_SYNTAX_ELEMENT_POOL_CHUNK_SIZES="json=16777216,xmlnsc=16777216,MQROOT=16777216,dfdl=16777216" 
```

### GLIBC_TUNABLES

This is not an ACE variable, but can be used to cause the glibc allocator to use mmap() instead of
the normal heap allocator. This means it uses munmap() on free, returning the pages to the OS, and
is enabled using the `glibc.malloc.mmap_threshold` tunable.

Reducing `glibc.malloc.mxfast` to 100 also helps reduce the storage overhead for the default string
buffer size in ACE.

Example:
```bash
GLIBC_TUNABLES="glibc.malloc.mxfast=100:glibc.malloc.mmap_max=1073741824:glibc.malloc.mmap_threshold=131072" 
```

### MQSI_PARSER_SHRINK_ON_IDLE_TIME

This environment variable is intended to be used to free storage used by parsers when
a thread becomes idle, either when it has returned to the thread pool or during active
operation. When an idle time is exceeded, the parser manager code will shrink parsers
associated with that thread if they are too large.

Example:
```bash
MQSI_PARSER_SHRINK_ON_IDLE_TIME="default=100:20" 
```

### MQSI_PARSER_FREE_ON_POOL_RETURN

This environment variable is intended to be used to free storage used by parsers when
a thread used by an HTTP Input node becomes idle and return to the thread pool. As of
ACE v13, the default behavior is to not free parsers on pool return (unlike the other
input nodes). With this setting enabled, when a thread returns to the thread pool the
input node message group is told to scale down, and this means to either free parsers
(this setting) or shrink them (MQSI_PARSER_SHRINK_ON_POOL_RETURN) if too large.

Example:
```bash
MQSI_PARSER_FREE_ON_POOL_RETURN="default=0"
```

## User variables

Setting the following in server.conf.yaml
```yaml
UserVariables:
  "parser-manager-log-scale-down-activity": true
```
will cause messages to be printed when the server frees parser storage:
```
2026-03-23 12:31:31.277630: BIP8099I: HTTPFlow.dfdl parser reduced storage: 2000004  -  40
2026-03-23 12:31:46.279424: BIP8099I: dfdl parser freed on scale down: 40  -  0
2026-03-23 12:31:46.279668: BIP8099I: WSRoot parser freed on scale down: 1  -  0
```

## CP4i artifacts

CP4i containers can be told to use these options, and this can help reduce storage use within
the cluster. This is possible in the 12.0.12-r22 SC2 release but no others at present.

### IntegrationRuntime

The [parser-storage-ir YAML](/cp4i/parser-storage-ir.yaml) file shows an example (which is actually
using the 12.0.12-r22 container with a CD operator, leading to slightly unusual config) of how to 
set the environment variables:
```yaml
            - name: MQSI_SYNTAX_ELEMENT_POOL_CHUNK_SIZES
              value: json=16777216,xmlnsc=16777216,dfdl=16777216
            - name: MQSI_PARSER_SHRINK_ON_IDLE_TIME
              value: default=100:20
            - name: MQSI_PARSER_FREE_ON_POOL_RETURN
              value: default=0
            - name: GLIBC_TUNABLES
              value: >-
                glibc.malloc.mxfast=100:glibc.malloc.mmap_max=1073741824:glibc.malloc.mmap_threshold=131072
```

### server.conf.yaml

A server.conf.yaml Configuration is needed for the 
```yaml
UserVariables:
  "parser-manager-log-scale-down-activity": true
```
setting; in the example IR this is called `verbose-parser-messages`.
