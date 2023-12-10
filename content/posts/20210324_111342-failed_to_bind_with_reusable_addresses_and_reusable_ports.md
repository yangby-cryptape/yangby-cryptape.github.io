---
title: Failed to Bind with Reusable Addresses and Reusable Ports
date: 2021-03-24T11:13:42+08:00
categories: [computer]
tags: [linux, network]
---

This article is talked about the errors when failed to bind with reusable
addresses and reusable ports.
<!--more-->

#### Introduction

Enable `SO_REUSEADDR` to reuse addresses,
and enable `SO_REUSEPORT` to reuse ports.

+ An example for servers:

  ```c
  int server_fd = socket(AF_INET, SOCK_STREAM, 0);
  int enable = 1;
  setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &enable, sizeof(enable));
  setsockopt(server_fd, SOL_SOCKET, SO_REUSEPORT, &enable, sizeof(enable));
  bind(server_fd, (struct sockaddr *)(&sockaddr), sizeof(struct sockaddr_in));
  listen(server_fd, backlog);
  ```

+ An example for clients:

  ```c
  int client_fd = socket(AF_INET, SOCK_STREAM, 0);
  int enable = 1;
  setsockopt(client_fd, SOL_SOCKET, SO_REUSEADDR, &enable, sizeof(enable));
  setsockopt(client_fd, SOL_SOCKET, SO_REUSEPORT, &enable, sizeof(enable));
  bind(client_fd, (struct sockaddr *)(&sockaddr), sizeof(struct sockaddr_in));
  connect(client_fd, (struct sockaddr *)(&remote_sockaddr), sizeof(struct sockaddr_in));
  ```

  Note:

  + Should select a specific source IP and port by calling `bind()` before
    calling `connect()` for clients.

  + If the port to `bind()` is `0`, the kernel will randomly choose a port
    from ephemeral ports.

    The range of ephemeral ports could be checked by follow command:

    ```bash
    sysctl net.ipv4.ip_local_port_range
    ```

A typical usage of "reuse" is using one single port to handle both incoming
connections and outgoing connections.

#### Potential Issues

##### Duplicated Links

If that both node `A` and node `B` use one single port to handle both
incoming connections and outgoing connections, and they try to connect to
each other at the same time, the latter will fail.

+ Suppose that `A` connects to `B` first, there will be links created in
  both `A` and `B`.

  ```
  Link                    protocol  source        target
  A::client->B::server    tcp       A_IP:A_PORT   B_IP:B_PORT
  B::server->A::client    tcp       B_IP:B_PORT   A_IP:A_PORT
  ```

+ Then, if `B` tries to connect to `A`, the follow links will be tried to
  create.

  ```
  Link                    protocol  source        target
  B::client->A::server    tcp       B_IP:B_PORT   A_IP:A_PORT
  ```

  And it will fail with `EADDRNOTAVAIL` since the link is duplicated.

Because the tuple `(protocol, source-ip, source-port, target-ip, target-port)`
should be unique in each node.

##### The Ephemeral Ports Problem

When all ephemeral ports are exhausted, calling `bind(port=0)` will fail,
even there still are ports can be reused.

Recall the example for clients, when calling `bind()`, the target ip and
target port is unknown for the kernel, so the kernel will just check the
tuple `(protocol, source-ip, source-port)`.

So the kernel will return `EADDRNOTAVAIL` since no more available ephemeral
ports.

Since kernel 5.7, there is a parameter could skip this error.

```bash
sysctl net.ipv4.ip_autobind_reuse=1
```

The description of `ip_autobind_reuse`:

> By default, bind() does not select the ports automatically even if
> the new socket and all sockets bound to the port have SO_REUSEADDR.
> ip_autobind_reuse allows bind() to reuse the port and this is useful
> when you use bind()+connect(), but may break some applications.
> The preferred solution is to use IP_BIND_ADDRESS_NO_PORT and this
> option should only be set by experts.
> Default: 0

Note: when `ip_autobind_reuse` is set to `1`, if all ephemeral ports are
exhausted, the kernel will randomly set a port, so there also has chance to
fail with `EADDRINUSE` when calling `connect()`.

#### References

- [The Ephemeral Ports Problem (and solution)]
- [Bind before connect]
- [Ephemeral ports and SO_REUSEADDR]
- [How do SO_REUSEADDR and SO_REUSEPORT differ?]
- [tcp: bind(0) remove the SO_REUSEADDR restriction when ephemeral ports are exhausted.]

[The Ephemeral Ports Problem (and solution)]: https://aleccolocco.blogspot.com/2008/11/ephemeral-ports-problem-and-solution.html
[Bind before connect]: https://idea.popcount.org/2014-04-03-bind-before-connect/
[Ephemeral ports and SO_REUSEADDR]: https://gavv.github.io/articles/ephemeral-port-reuse/
[How do SO_REUSEADDR and SO_REUSEPORT differ?]: https://stackoverflow.com/a/14388707
[tcp: bind(0) remove the SO_REUSEADDR restriction when ephemeral ports are exhausted.]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4b01a9674231a97553a55456d883f584e948a78d
