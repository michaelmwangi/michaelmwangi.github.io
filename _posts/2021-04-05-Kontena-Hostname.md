---
layout: post
title:  "Kontena, Hostname : part 2"
date:   2021-04-05 13:18:54 +0400
categories: jekyll update
---

From the prerequisites we saw that a container is not a single construct but its something which visually looks like. 



![container](/assets/what_is_a_container.png){:class="img-responsive"}


Our Kontena journey will begin by launching the container process with its own hostname different from the host.
To achieve this we will be using the **UTS** namespace which will essentially allow our process to have an isolated
view of the hostname, meaning that we can change it without affecting the host.

We will be launching the process using the **clone** syscall which will have the 
relevant hostname namespace flag set.

First things first lets get our feet wet with namespaces. The **unshare** linux utility allows us to launch new processes which have separate 
namespaces from the host, all from the command line. To create a UTS namespace we issue. 

{% highlight code %}
$ unshare -u sh
{% endhighlight %}

The above command creates a new shell process in a new UTS
namespace. To confirm this we can compare the current UTS namespace with that
of the host on a different terminal window.

{% highlight code %}
//inside the new shell
# readlink /proc/self/ns/uts
uts:[4026533486]
{%  endhighlight %}

{% highlight code %}
//in the host. On a new terminal window
# readlink  /proc/self/ns/uts
uts:[4026531838]
{%  endhighlight %}

The old  mantra of everything in unix is a file still applies even in namespaces. The namespaces are symbolic links under the directory
 {% highlight code %} /proc/<pid>/ns {% endhighlight %}
To see all the namespaces that the current process has access to it's a simple as just listing  the contents of that directory.

{% highlight code %}
$ ls -l /proc/self/ns
{% endhighlight %}

The readlink simply resolves the name of the symbolic link which in our case is the UTS namespace. Now back to our experiment, we can see that the the new shell process and the host are in different UTS namespaces. This means that we can change the hostname inside the new shell without changing that of the host. 

With this info we can create a foundation skeleton of our kontena runtime. 

{% highlight code %}
package main

import (
	"fmt"
	"os"
	"os/exec"
	"syscall"
)

func changeHostname(hostname string) error {
	if err := syscall.Sethostname([]byte(hostname)); err != nil {
		return err
	}

	return nil
}

func launchProcess(process string) error {
	cmd := exec.Command(process)
	cmd.Stderr = os.Stderr
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	return cmd.Run()
}

func initializeContainer() error {
	if err := changeHostname("kontenahostname"); err != nil {
		return err
	}

	return launchProcess("/bin/sh")
}

func run() error {
	cmd := exec.Command("/proc/self/exe", "init")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS,
	}
	cmd.Stdin = os.Stdin
	cmd.Stderr = os.Stderr
	cmd.Stdout = os.Stdout

	if err := cmd.Run(); err != nil {
		return err
	}
	return nil
}

func main() {
	args := os.Args
	if len(args) <= 1 {
		if err := run(); err != nil {
			fmt.Printf("error running kontena: %v \n", err)
		}
	} else {

		switch args[1] {
		case "init":
			if err := initializeContainer(); err != nil {
				fmt.Printf("error initializing container: %v \n", err)
			}
		default:
			fmt.Printf("unknown arg passed: %s \n", args[1])
		}
	}

}


{%  endhighlight %}

Running this we get. 

{% highlight code %}
$ go build -o kontena
sudo ./kontena
sh-5.0# hostname
kontenahostname
sh-5.0# exit
{%  endhighlight %}

The code above basically does what our experiment with unshare does, we can now play around with the hostname of the new shell without messing up the 
host. Here we build up on the **self exec'ng** that we highlighted on the prerequisites section. The self exec is triggered when **run()** executes. However, the new konetna copy 
is a little different from the original one in that new one is running in a new UTS namespace. We achieved this by simply passing down 
the **syscall.CLONE_NEWUTS** clone flag. Once we are in a new UTS namespace we change the hostname and then finally launch our intended container process. A visual presentation of 
this looks like.

![Hostname change](/assets/hostname_change.png){:class="img-responsive"}

As we can see it's a 2 step process. In the first step we create a new process
with a new UTS namespace and change the hostname and then launch the contaiener
process , in our case the shell process with all of the relevant changes set.
We will be using most of this pattern in this project. 

As for the next step we will be separating the filesystem that our container
process will use. This will ensure that our container can make changes on the
filesystem, use a different linux distribution from the host and more. We will
achieve this by making use of the linux **mount** namespaces.

**References:**

1. [Unshare](https://man7.org/linux/man-pages/man1/unshare.1.html)
2. [Linux namespaces](https://lwn.net/Articles/531114/)