---
title: "Docker containers"
weight: 60
---

gokrazy’s goal is to make it easy to build Go appliances. In an ideal world, all
building blocks you need would be available in Go. In reality, that is not
entirely the case. Perhaps you need to run a C program next to your Go
programs. Docker containers make incremental (or partial) adoption of gokrazy
easy.

We’re going to use podman, a drop-in replacement for Docker, because there is a
statically compiled version for amd64 and arm64 available that we could easily
re-package into https://github.com/gokrazy/podman.

## Step 1: Install podman to your gokrazy device

In your `gokr-packer` invocation (see [Quickstart](/quickstart/) if you don’t
have one yet), include the [`iptables`](https://github.com/gokrazy/iptables),
[`nsenter`](https://github.com/gokrazy/nsenter) and
[`podman`](https://github.com/gokrazy/podman) packages:

{{< highlight shell "hl_lines=6-8" >}}
gokr-packer \
  -update=yes \
  github.com/gokrazy/hello \
  github.com/gokrazy/breakglass \
  github.com/gokrazy/serial-busybox \
  github.com/gokrazy/iptables \
  github.com/gokrazy/nsenter \
  github.com/gokrazy/podman
{{< /highlight >}}

## Step 2: Verify podman works

Use [breakglass](https://github.com/gokrazy/breakglass) to login to your gokrazy
instance and run a container manually:

```shell
/tmp/breakglass187145741 $ mount -t tmpfs tmpfs /var
/tmp/breakglass187145741 $ podman run --rm -ti docker.io/library/debian:sid
root@gokrazy:/# cat /etc/debian_version
bookworm/sid
```

## Step 3: Use podman programmatically

Now that you have the required tools, there are a couple of decisions you have
to make depending on what you want to run in your container(s):

1. Should container data be stored ephemerally in `tmpfs` (lost with the next
   reboot), on the permanent partition of your SD card, or somewhere else
   entirely (e.g. network storage)?
1. Do you want to pull new container versions automatically before each run, or
   manually on demand only?
1. Should your container be started as a one-off job only ([→ detached
   mode](https://docs.docker.com/engine/reference/run/#detached--d)), or
   supervised continuously (restarted when it exits)?
1. Should your container use a deterministic name (so that you can `exec`
   commands in it easily), or use a fresh name for each run (so that there never
   are conflicts)?

Aside from these broad questions, you very likely need to set a bunch of detail
options for your container, such as additional environment variables, volume
mounts, networking flags, or command line arguments.

The following program is an example for how this could look like. I use this
program to run [irssi](https://irssi.org/).

```go
package main

import (
	"fmt"
	"log"
	"os"
	"os/exec"
	"strings"
	"syscall"

	"github.com/gokrazy/gokrazy"
)

func podman(args ...string) error {
	podman := exec.Command("/usr/local/bin/podman", args...)
	podman.Env = expandPath(os.Environ())
	podman.Env = append(podman.Env, "TMPDIR=/tmp")
	podman.Stdin = os.Stdin
	podman.Stdout = os.Stdout
	podman.Stderr = os.Stderr
	if err := podman.Run(); err != nil {
		return fmt.Errorf("%v: %v", podman.Args, err)
	}
	return nil
}

func irssi() error {
	// Ensure we have an up-to-date clock, which in turn also means that
	// networking is up. This is relevant because podman takes what’s in
	// /etc/resolv.conf (nothing at boot) and holds on to it, meaning your
	// container will never have working networking if it starts too early.
	gokrazy.WaitForClock()

	if err := mountVar(); err != nil {
		return err
	}

	if err := podman("kill", "irssi"); err != nil {
		log.Print(err)
	}

	if err := podman("rm", "irssi"); err != nil {
		log.Print(err)
	}

	// You could podman pull here.

	if err := podman("run",
		"-td",
		"-v", "/perm/irssi:/home/michael/.irssi",
		"-v", "/perm/irclogs:/home/michael/irclogs",
		"-e", "TERM=rxvt-unicode",
		"-e", "LANG=C.UTF-8",
		"--network", "host",
		"--name", "irssi",
		"docker.io/stapelberg/irssi:latest",
		"screen", "-S", "irssi", "irssi"); err != nil {
		return err
	}

	return nil
}

func main() {
	if err := irssi(); err != nil {
		log.Fatal(err)
	}
}

// mountVar bind-mounts /perm/container-storage to /var if needed.
// This could be handled by an fstab(5) feature in gokrazy in the future.
func mountVar() error {
	b, err := os.ReadFile("/proc/self/mountinfo")
	if err != nil {
		return err
	}
	for _, line := range strings.Split(strings.TrimSpace(string(b)), "\n") {
		parts := strings.Fields(line)
		if len(parts) < 5 {
			continue
		}
		mountpoint := parts[4]
		log.Printf("Found mountpoint %q", parts[4])
		if mountpoint == "/var" {
			log.Printf("/var file system already mounted, nothing to do")
			return nil
		}
	}

	if err := syscall.Mount("/perm/container-storage", "/var", "", syscall.MS_BIND, ""); err != nil {
		return fmt.Errorf("mounting /perm/container-storage to /var: %v", err)
	}

	return nil
}

// expandPath returns env, but with PATH= modified or added
// such that both /user and /usr/local/bin are included, which podman needs.
func expandPath(env []string) []string {
	extra := "/user:/usr/local/bin"
	found := false
	for idx, val := range env {
		parts := strings.Split(val, "=")
		if len(parts) < 2 {
			continue // malformed entry
		}
		key := parts[0]
		if key != "PATH" {
			continue
		}
		val := strings.Join(parts[1:], "=")
		env[idx] = fmt.Sprintf("%s=%s:%s", key, extra, val)
		found = true
	}
	if !found {
		const busyboxDefaultPATH = "/usr/local/sbin:/sbin:/usr/sbin:/usr/local/bin:/bin:/usr/bin"
		env = append(env, fmt.Sprintf("PATH=%s:%s", extra, busyboxDefaultPATH))
	}
	return env
}
```
