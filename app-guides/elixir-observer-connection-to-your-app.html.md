---
title: Connecting Observer to your app in production
layout: docs
sitemap: false
nav: firecracker
author: mark
categories:
  - elixir
date: 2021-06-15
---

Elixir, Erlang, and really just the BEAM has a feature called "[Observer](https://elixir-lang.org/getting-started/debugging.html#observer)". It's a powerful UI that connects to a running Elixir or Erlang node and let's you "observe" what's going on inside. It has some limited ability to modify things as well, most notably you can kill running processes. This can help when something is misbehaving or you just want to play "chaos monkey" and kill parts of the system to see how it recovers.

This shows a process tree for the application. Using this I can inspect individual processes or even kill them!

![Observer screen shot](/docs/images/observer-tictac-local-application-pane.png?card&2/3&centered)

One very cool way to run Observer is to run it on your local machine (which has the ability to display the UI) and connect to a production server (with no windowing UI available) and "observe" it from a distance. So yeah... have a problem in production? Not sure what's going on? You can literally tunnel in, crack the lid and poke, prod, and peek around to see what's going on.

Next we'll cover how your project can support this feature and see how to do it on Fly.io!

## What We Will Do

Fly.io natively supports [WireGuard](https://www.wireguard.com/), Jason Donenfeld's amazing VPN protocol. If you’ve ever lost hours of your life trying to set up an IPSec VPN, you’ll be blown away by how easy WireGuard is. It’s so flexible and performant that Fly uses it as our network fabric. And it’s supported on [every major platform](https://www.wireguard.com/install/), including macOS, iOS, Windows, and Linux. What that means for you is that if your app runs on Fly, you can open a secure, private, direct connection from your dev machine to your production network, in less time than it took me to write this paragraph. Cool, right?

This is what we're going to do.

![WireGuard observer connection](/docs/images/elixir-wireguard-observer-tunnel.png?2/3&centered)

We will bring up a secure WireGuard tunnel that links to your servers on Fly. In this graphic, there are two `my_app` Elixir nodes clustered together running on Fly.

From the local machine, we can open an IEx terminal configured to **join** that cluster of remote Elixir nodes. Our local machine supports running Observer and drawing the UI. We use our local observer to talk to the remote nodes in the cluster!

Let's do it. This will be fun!

## The Cookie Situation

Before two Elixir nodes **can** cluster together, they must share a secret cookie. The cookie itself isn't meant to be a super secret encryption key or anything like that, it's designed to let you create multiple sets of small clusters on the same network that don't all just connect together. Different cookies means different clusters. For instance, only the nodes that all use the cookie "abc" will connect together.

For us, this means that in order for `my_remote` node to connect to the cluster on Fly, I need to share the same cookie value used in production.

### The Cookie Problem

When you build a `mix release`, it generates a long random string for the cookie value. When you **re-run** the `mix release` command, it keeps the same cookie value. That is, when you don't run it in Docker. The Dockerfile we're using is building a fresh release every time we run it. That's kind of the point of a Docker container. So **our cookie value is being randomly generated every time we deploy**. This means after every deploy, I would have to figure out what the new cookie value is so my local node can use it.

### The Cookie Solution

The easiest solution here is to **specify** the value to use for our cookie. One that we will know outside of the build and that won't keep changing on us.

## Making the Changes

Let's walk through making the changes on the [`hello_elixir` project](https://github.com/fly-apps/hello_elixir-dockerfile) used in the [Elixir guide](/docs/getting-started/elixir/) to see what's involved.

### Release Section

In your `mix.exs` file, you can add a `releases/0` function that returns the configuration.

This is following the [mix release](https://hexdocs.pm/mix/Mix.Tasks.Release.html) docs, these are the changes we'll make:


```elixir
defmodule HelloElixir.MixProject do
  use Mix.Project

  def project do
    [
      app: :hello_elixir,
      # ...
      releases: releases()
    ]
  end

  # ...

  defp releases() do
    [
      hello_elixir: [
        include_executables_for: [:unix],
        cookie: "YOUR-COOKIE-VALUE"
      ]
    ]
  end
end
```

The `releases` function returns a keyword list. To clarify, the `:hello_elixir` atom isn't actually important. It could be named `demo` or `full_app`. The name becomes important when you are defining **multiple** release configurations. However, that's beyond the scope of this guide. When only defining a single configuration, it uses that regardless of the name. So for simplicity, I'll just name it the same as the application.


For the `:cookie` value, you can generate a unique value using the following Elixir command:

```elixir
Base.url_encode64(:crypto.strong_rand_bytes(40))
```


Once your pre-defined cookie value is set, deploy your updated app.

```cmd
fly deploy
```

If desired, you can verify that the cookie value was set correctly in production, here's how:

```
$ fly ssh console
Connecting to icy-leaf-7381.internal... complete

/ #  cat app/releases/COOKIE
YOUR-COOKIE-VALUE
```

With a known and unchanging cookie value deployed in our application, we are ready for the next step!
### Umbrella Note

If you have an umbrella project, you may want to add an option called `:applications`. In it, you specify the name of all "entrypoint" applications. For instance, if I had two apps in my umbrella: `web` and `core` where `web` is a Phoenix application, that is the entrypoint. My configuration would look like this:

```elixir
  defp releases() do
    [
      hello_elixir: [
        include_executables_for: [:unix],
        applications: [web: :permanent],
        cookie: "YOUR-COOKIE-VALUE"
      ]
    ]
  end
```

This instructs which applications should be started and in which order for this release configuration. In my application `web` has a dependency on `core`, so specifying `web` is all I need to do.

See the [documentation here](https://hexdocs.pm/mix/Mix.Tasks.Release.html#module-customization) for more details and what other options are available.

## WireGuard Tunnel

First, setup WireGuard on your local machine. Follow the Fly.io [Private Network VPN](/docs/reference/privatenetwork/#private-network-vpn) guide to walk through that.


## Connecting to Production

To make the connection, there are several steps in the process. We'll create a short script to automate the process for us!

### Knowing the Cookie

Our script needs to know the cookie value. The easiest way to do this and keep the script generic is to set the value in the ENV. This lets us copy the script unchanged to multiple projects.

To help manage project-specific ENV values, I like using [direnv](https://direnv.net/). When changing into a directory with an `.envrc` file, it loads those values into my ENV, when I leave that directory, it unloads them. This means I can set a COOKIE value (or other config) specific to each project and it works great.

You don't have to use a tool like `direnv` though. The `./observer` script file can be customized to set the COOKIE value explicitly if you prefer that approach. Refer to the full [script file here](https://github.com/fly-apps/hello_elixir-dockerfile/blob/explicitly-set-release-cookie/observer#L14) in the comments to see how you can  do that.

### Script File

This is a simple bash script file to kick off a correctly configured local IEx session, connect our node to the remote cluster, and start Observer.

Create a file titled `observer`. Here are the important contents. (See [here for full script file](https://github.com/fly-apps/hello_elixir-dockerfile/blob/explicitly-set-release-cookie/observer))

```bash
#!/bin/bash

set -e

if [ -z "$COOKIE" ]; then
    echo "Set the COOKIE your project uses in the COOKIE ENV value before running this script"
    exit 1
fi

# Get the first IPv6 address returned
ip_array=( $(fly ips private | awk '(NR>1){ print $3 }') )
IP=${ip_array[0]}

# Get the Fly app name. Assumes it is used as part of the full node name
APP_NAME=`fly info --name`
FULL_NODE_NAME="${APP_NAME}@${IP}"
echo Attempting to connect to $FULL_NODE_NAME

# Export the BEAM settings for running the "iex" command.
# This creates a local node named "my_remote". The name used isn't important.
# The cookie must match the cookie used in your project so the two nodes can connect.
(export ELIXIR_ERL_OPTIONS="-proto_dist inet6_tcp"; iex --sname my_remote --cookie ${COOKIE} -e "IO.inspect(Node.connect(:'${FULL_NODE_NAME}'), label: \"Node Connected?\"); IO.inspect(Node.list(), label: \"Connected Nodes\"); :observer.start")
```

This should work fine on Linux and MacOS. On Windows, if you are using [WSL2](https://docs.microsoft.com/en-us/windows/wsl/install-win10) then it will work because it's Linux. Otherwise refer to the manual steps outlined below.

Make the script file executable:

```cmd
chmod +x observer
```

Execute the script:

```cmd
./observer
```
```output
Attempting to connect to icy-leaf-7381@fdaa:0:1da8:a7b:ac2:baed:c434:2
Erlang/OTP 24 [erts-12.0.1] [source] [64-bit] [smp:4:4] [ds:4:4:10] [async-threads:1] [jit]

Node Connected?: true
Connected Nodes: [:"icy-leaf-7381@fdaa:0:1da8:a7b:ac2:baed:c434:2"]

...

Interactive Elixir (1.12.1) - press Ctrl+C to exit (type h() ENTER for help)
```

When observer first opens, it might looks something like this:

![Observer connected to local node](/docs/images/observer-local-node.png?centered)

Notice that the window title shows `my_remote@...`? This means it's showing the stats of my local IEx node that isn't actually running any of my code. So this data isn't very interesting yet.

If everything worked and it's connected, under the Nodes menu you should see the connected remote node.

![Observer connected to local node](/docs/images/observer-local-node-menu.png?card&2/3&centered)

When the remote node is selected, then all the stats and information changes to reflect what's going on in the selected node.

![Observer connected to local node](/docs/images/observer-local-node-connected.png?scentered)

It worked! I'm seeing the information for the production node!

### Success!

Let's review briefly what was accomplished.

* I setup a WireGuard tunnel from my personal computer into my private Fly network.
* I started a local Elixir node that shares the same cookie value.
* My local node connected over WireGuard to the production cluster.
* I launched Observer.

Now, using WireGuard and this script, I can easily launch Observer and observe any node in the cluster!

## Disconnecting

When done, close Observer. It leaves you with an open IEx shell that is still connected to the remote cluster. You can safely CTRL+C, CTRL+C to exit it.

At this point you can shutdown your WireGuard connection as well if desired.

## Tips and Troubleshooting

The script is a simple tool to make it easy to launch observer and connect to the cluster. It doesn't diagnose or handle all the things that can fail. For instance, if your WireGuard connection isn't up, it just won't find the server but it also won't complain. If you encounter issues, you can go through the manual steps below to help diagnose any problems.

In order for everything to work, here's the checklist overview:

- Your wireguard connection must be up.
- Your application defines a release that specifies the cookie value to use.
- The local COOKIE value must be the same as the cookie value used in production.
- Observer needs to be working in your local environment. That requires WxWidget support in your Erlang install.

### Manual Script Steps

If you encounter issues, this can help you diagnose what's going on. The script automates 4 things.

1. Getting the cookie value from the ENV - make sure the correct cookie value is either available in the ENV or explicitly set in the script.
2. Uses the `fly info --name` command to get the app name. This is used to build the fully qualified node name.
3. Get the first IPv6 address for your server using `fly ips private | awk '(NR>1){ print $3 }'`. If you have multiple servers, it just returns the first one. You only need one. Once you join to any node you are introduced and connected to all of them.
4. Set up a local node and executes multiple commands.
  1. It runs a command like `Node.connect(:'icy-leaf-7381@fdaa:0:1da8:a7b:ac2:baed:c434:2')` to connect to the remote node. It returns `true` when it succeeds or `false` when it fails. The app name and the IP address used to make up the node's name are assembled from the previous steps.
  2. Launch observer with the command `:observer.start`. If this fails, check the other tip for WxWidgets.

To do it manually, once you get the IP address, you can customize the following command to launch Observer.

```
ELIXIR_ERL_OPTIONS="-proto_dist inet6_tcp" iex --sname my_remote --cookie YOUR-COOKIE-VALUE -e "Node.connect(:'APP_NAME@IP_ADDRESS'); :observer.start"
```

You need to substitute in your `YOUR-COOKIE-VALUE` value, the `APP_NAME` and the `IP_ADDRESS`.

### WxWidgets Support

If you are using [asdf-vm](https://asdf-vm.com/) for managing your Elixir and Erlang versions, check out the [Erlang plugin's documentation](https://github.com/asdf-vm/asdf-erlang) for getting WxWidget support in your Erlang environment. This is required for using Observer.
