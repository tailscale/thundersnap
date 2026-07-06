# THUNDERSNAP

Thundersnap is a collection of new and old ideas in distributed filesystem
design. It’s named after the primary two ways this early version is likely
to fail: the thundering herd (its mesh replication protocol) and the Thanos
Snap (which is likely to lose roughly half your data).

Anyway, don’t use it for anything in production right now! It’s a toy.

But, being a toy gives us a chance to be creative. Imagine if we removed all
the requirements of a production-ready distributed system. It doesn’t have
to scale up infinitely. It doesn’t have to have completely predictable
performance. You don’t have to strictly separate persistent storage from
your runtime. You don’t have to build an idempotent container before you
execute it. You can do the fun thing, not the careful thing.

What if your entire computer had an “undo” button? What if you could be
running an entire Debian system and container on one computer, and with a
couple of commands move it to another computer? What if you had a nix-like
content-addressed operating system, but without the nix, because anything
can be content addressed?

That’s what we’re going for. It might not be useful, but it’s fun.

## Getting started

To start with Thundersnap, install one of the distribution packages or build
from source. (TODO: put the distribution packages somewhere.)

Note: you need either a root filesystem on btrfs, or you need to give
thundersnapd a `--data-dir` that’s on a btrfs mounted somewhere. Maybe
someday we won’t depend on btrfs but today we do.

Then you need to activate by logging it into Tailscale. Thundersnap
fundamentally needs Tailscale to work, so that its auth and mesh replication
features are secure. You can use Headscale (open source control server) if
you want, or [Tailscale’s generous free
plan](https://tailscale.com/blog/free-plan). Anyway, ones thundersnapd is
running in the background, use `thundersnapd --activate` and follow the URL
to log in.

Once that’s done, you can use ssh to access your first empty container:

`ssh root@@thundersnap`

(Yes, the double at sign means the container name is empty, which gives you
the default empty space to play in.)

## Durable snapshots

The primary attribute of a Thundersnap system is Durability — the D in
database “ACID” terminology. Thundersnap works pretty hard to make your data
durable. It doesn’t particularly try at Atomicity, Consistency, or
Isolation. In this version it’s probably not super Durable either, but
that’s not an architectural limit.

The underlying thesis of Thundersnap is that modern distributed storage
needs to rely on *snapshots* rather than real-time synchronization. Nothing
else gives you useful multi-user semantics anyway (you really don’t want to
see the other users’ half-finished work!). And because real globally
distributed systems (including VMs on your laptop or phone) have latency and
reliability problems, being full-time connected to all replicas is an
illusion anyway.

In short, Thundersnap assumes about filesystems what git assumes about
source code trees. That’s not a coincidence since its structure is inspired
by my old [2010-era bup project](https://apenwarr.ca/log/20100104) which
itself is inspired by git.

You can run Thundersnap on disposable or just low-quality computers. Rather
than trying to make those computers (or their disks) higher quality or less
disposable, we make it easy to take content-addressable snapshots, and then
to replicate those snapshots efficiently to other computers. If you have
enough computers in enough places, you probably won’t lose your data.

Thundersnap currently uses btrfs to make instantaneous snapshots, but it
doesn’t use btrfs replication since that doesn’t use a content-addressable
format and thus has long term architectural limitations. As a result,
Thundersnap’s *implementation* currently depends on btrfs but its
*architecture* and *protocol* do not. Someday a better snapshotting
filesystem might exist, and we could switch to it transparently.

To make a snapshot, use `ts snap`.

## Lightweight execution frames

Thundersnap defaults to using Linux namespacing (”containers”) instead of
full VMs. The idea is that modern cloud architecture suggests making
everything you do into its own tiny isolated microVM. Although that’s very
safe, it’s also very slow, and somehow we’ve gotten used to it.

In Unix, you can start and stop a small program in a millisecond.
Thundersnap aims to be that fast at starting containers (it’s not quite that
fast yet, but it’s very fast). And those containers are a bit better
isolated than a Unix process. It’s not perfect, but it’s fast.

<aside> 💡

Note: There is also partly-fleshed-out support for VM-based isolation; the
idea is that you can put one or more containers into a microVM, and all of
them still share the same underlying btrfs filesystem. The VMs add a layer
of abstraction, but it’s still quite fast. There’s more work to do there,
but you can play with it today.

</aside>

An execution frame is actually made of three independent snaps:

- root (`/`): the operating system (which you can extract from a docker
container to start with, if you like) - `/home`: the /home directory that
contains your personal preferences and tools - `/work`: the /work directory
where you put the project you’re working on.

You might be surprised that `/home` and `/work` are separate. We did it that
way because you might want to swap out each of the three parts
independently. For example, if you’re working in a Debian container and want
to try your program in a Red Hat build environment, you’d swap out the root.
If you want to give your toy to a friend to continue development, they might
keep the root and `/work`, but use their own `/home` so that all their
personal dotfiles are intact.

To create a new execution frame, use `ts frame <root:home:work>`. Any of
those strings can be blank, which means “keep the one I already have.” Or
they can be `nil` which means that component is initially empty.

You jump into a different frame with `ts go`. For example, `ts go
nil:nil:nil` creates a new empty frame and makes an interactive shell in it;
exiting that shell gets you back to the parent frame. `ts go
<redhat_snap_id>::` creates a new frame that’s a perfect copy of the current
one, except with a redhat root filesystem, and jumps into that.

A slightly weird (for now) variant of `ts go` is `ts undo`. It creates a new
frame from your *previous* snapshot, then enters that, sending you backward
in time. You can then `ts undo` again to go back another snapshot, and so
on. The semantics of this are not quite what I want (shouldn’t it replace
the current frame or something?), but it makes a good demo.

## Refs: named frames

Frames are normally named after a uuid, because we create a lot of them and
they are often for throwaway temporary purposes. Every time you want to do
an experiment, you can spin up a frame from a copy of your current
environment, do the experiment, and throw it away. This is similar to
“commits” in git, that each have their own gibberish name. (Snaps are like
“trees” in git.)

Refs are also like git refs (ie. branches and tags). You can point a ref at
any frame uuid you want. Once you do that, you can refer to that exact frame
by using the ref. Multiple refs can point at the same uuid, but each ref
points at exactly one uuid. You manipulate refs with the `ts ref` command,
or the `--ref` flag when using `ts frame`.

Refs also have private data associated with them that does *not* get
snapshotted. This is intended to be used for secret identity and key files
that should always exist in precisely one frame. For example, Tailscale’s
[tsnet](https://tailscale.com/docs/features/tsnet) library maintains
per-node state information that uniquely identify a given node. The state
for a given ref is in `/id/<ref>/` inside the frame the ref currently points
to. If you move a ref to a different frame, that directory moves too (and
all processes inside the old frame are terminated so that there’s no
leftover key material floating around in RAM).

This lets you do blue-green deployments for example:

- `ts frame --ref blue <whatever>` - `ts go blue` - `app
--state-dir=/id/blue/ --login` # do the Tailscale login flow - …get app
working… - `ts frame --ref=green ::` # duplicate my frame - `ts go green` #
duplicate my frame - `app --state-dir=/id/green --login` # do the Tailscale
login flow - …make changes to app and test it in green mode… - `ts ref move
blue $(ts frame)`

Anyway this part is only lightly tested, but you get the idea.

## Lightweight persistent applications

Speaking of apps, it’s nice to have a way to run them automatically even
when you yourself are not connected to a frame.

Instead of running `/sbin/init` inside each VM or container, we just provide
a *filesystem* that looks like what you want, and a simple `ts autorun`
command that lets you make sure a given program is always running inside a
container.

The idea is you can ssh into a container, edit the program live, kill it,
and it’ll restart.

You’ll feel guilty doing this. It’s like 1990s-era editing php websites on
the production server. But remember, Thundersnap isn’t for production, it’s
for toys. Does it really matter if your toy has some downtime? If you screw
up, `ts snap` snapshotted the old one as often as you wanted.

## Converting docker containers to snaps

Docker containers famously have “names” that can be updated. So when you
download a particular container, you might not be completely sure it’s the
same as the one you downloaded yesterday. This is “neat” if you want to pull
in the latest security fixes, but also means you pull in arbitrary other
changes that might be better or worse, but at the very least will be
surprising.

Thundersnap works differently, If you use for example `ts download-docker
debian:latest` it will download the latest docker container named
`debian:latest`, but then it prints out the content hash of the snap it
created; every time you use that content hash, you get exactly what you
expected.

So for example,

`ts frame --ref=deb (ts download-docker debian:latest)::`

will give you a new frame named `deb` that contains the `debian:latest` OS
and your current home and work trees. You can then enter it with `ts go deb`
or by exiting your ssh session and re-entering it: `ssh
root@deb@thundersnap`.

## Mesh replication

This part is buggy still but the core framework is there. To enable mesh
mode, you need to pass `--mesh` when starting thundersnapd.

When mesh mode is enabled, thundersnapd periodically pings all the machines
on your tailnet that have the same identity as it does (by default, your
username). This is a little hacky and we should fix it, but it’s a good way
to get started. So if you have multiple computers and you install
`thundersnapd —mesh` on each, they’ll see each other. You can then see their
mesh status and peer list by visiting
[http://thundersnap:7575/](http://thundersnap:7575/) if you want. (Replace
“thundersnap” with whatever hostname you assign.)

One you have peers, you can replicate snaps. The only command for that right
now is `ts download-snap <snapid>` — as long as you know the snapid
generated from `ts snap` on one machine, you can download-snap it to
another, and then construct frames from it.

This leaves a lot of room for future automation. For example, I’d like to
automatically replicate snaps to another machine every time I run `ts snap`
on my local machine, automatically prune old snaps, have one-button
migration for a frame from one computer to another, etc. You could build all
these as tools on top of thundersnap’s existing primitives.

## What’s the relationship of Thundersnap with Tailscale?

Tailscale owns the copyright but for now this is a “personal” project of
mine (Avery Pennarun, apenwarr). It’s not actually used for anything inside
Tailscale right now, nor should it be — it’s a toy that I use for some of my
own stuff. Tailscale also has four or five other
thing-for-starting-VMs-or-container tools, some of which are open source,
and some of which will be soon. Some of those are used in production, some
of them heavily. Thundersnap isn’t.

I think Thundersnap is the most fun to play with… but I’m biased.

Tailscale is a “real company” now, and official Tailscale products have to
have a certain level of quality so we can provide actual support to people
who use them for real work. But, to really have fun in our homelabs,
sometimes we have to push the boundary farther than that.

Thundersnap combines a lot of unusual assumptions into a single package. It
will take a while for that to mature.

But I’d love to hear what you think! I’m apenwarr@tailscale.com, or
@apenwarr.ca on Bluesky, which I’m told you can also reach through the
Fediverse through a tool called bridgy.
