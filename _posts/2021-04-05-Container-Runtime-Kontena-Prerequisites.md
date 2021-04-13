---
layout: post
title:  "Container Runtime, Kontena, Prerequisites : part 1"
date:   2021-04-05 13:18:54 +0400
categories: jekyll update
---

I have been working with containers for quite a while now and although I have known about the building blocks of containers (namespaces, cgroups) the idea 
has never been intimate with me beyond listing namespaces and such. This project, **Kontena**, is an attempt to build a container runtime system that will 
run a hello world nginx app accessible from the browser. Before we get started there are tons of excellent guides of making your own container runtime system notably 
[this](https://unixism.net/2020/06/containers-the-hard-way-gocker-a-mini-docker-written-in-go/) and [this](http://ifeanyi.co/posts/linux-namespaces-part-1/) but I decided to go ahead and make my own from my interpretation of the research. 

A container can be loosely described as a sandboxed process running in the OS
kernel. ( A more stricter sandbox definition can be achieved through the use of
tools like gvisor ) The essential components of the sandbox are 

1. **Namespaces** - These give a process an isolated view of a computing resource eg the filesystem, network etc

2. **Capabilities** - In linux the root user assumes 'god-like' capabilities whereby they can do anything in the system, capabilities on the other hand break up the 'god-mode' into distinct capabilities which can be attached/removed from a process independently. This makes containers more secure in that they don't get capabilities they don't require. 

3. **Cgroups** - These distribute tasks along a heirarchy of system resources by using controllers which deal with specific resources. A good example in this is specifying the maximum amount of memory a process can use regardless of the amount of memory in the host system.


From the above definitions we can see that containers are not a single construct
in the linux kernel rather a container created by launching a process and
attaching to it the relevant namespaces and cgroups followed by removal or
addition of the relevant capabilities. Ofcourse there is alot of plumbing work to be done which is what
the bulk of this series is going to be about.

The language of choice for building kontena will be golang simply because it's
what am familiar with at the moment. Golang abstracts the head aches of running
concurrent code through the use of goroutines. Under the hood this concurrency is implememted by multiplexing the
goroutines on a bunch of OS threads which are launched by the go runtime by
making use of the **clone** sycall. The multiplexing of the goroutines is handled 
by the  golang scheduler which decides, based on some events, which OS thread is going to execute a
goroutine. One of the events that may cause the scheduler to switch OS threads
is if the current goroutine is making a blocking call eg a __syscall__, this will
necessitate the scheduler to schedule other goroutines on other OS threads. To
see these OS threads in action we can make use of **strace** to trace a simple
hello world golang program. 

{% highlight code %}


package main

import "fmt"

func main() {
	fmt.Println("hello world")
}

{% endhighlight  %}


Build the code into an executable 

{% highlight code %}go build -o hello{% endhighlight %}

Trace using strace

{% highlight code %}strace -e trace=clone ./hello{% endhighlight %}

The following is an excerpt from my local setup

{% highlight code %}

....

clone(child_stack=0xc00004a000, flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM) = 109767

... other similar lines with a different ID 

{% endhighlight %}


As we can see the clone flags match the ones specified in code. Another
observation to note is the use of the **CLONE_THREAD** flag which means that the
launched OS thread will have the same PID as that of the parent. This explains
why in this case we wont be able to 'grep' for a process with PID 109767 But we
can instead get the parents PID and then issue 
{% highlight code %}sudo ps -To pid,tid,tgid,tty,time,comm -p <the_parent_pid>{% endhighlight %}
and then observe the tid  . So basically 109767 in this case is a thread id
(TID).

From this observation we can conclude that most golang programs will mostly be
multithreaded program from the kernel's point of view. As we are now in
multithreaded territory 2 major issues arise from this.

### **1. No fork syscall in golang**

Traditionally whenever a program needs to create a new process it makes use of
the ***fork*** syscall which duplicates the calling process into a new
child process. However, if the parent process has child threads then only the calling thread is
duplicated. This has the potential of leaving the child in a deadlocked state
especially in cases where the other threads acquired locks, from the new
process's perspective it has a lock that does not have anyone to release it since the other
threads were not duplicated into this new process. So basically thread states are
not duplicated. With this quirk and Golang's implementation model of
concurrency the fork syscall is not directly exposed on the standard
library syscall utils.This puts us in a tricky situation since we need to launch a new process and do some of the
sandbox magic inorder to create a container. Instead we are going to use
***exec*** which is another way of launching new processes.The major
difference from fork sycall is that instead of creating a child process the
parent process is replaced by the new executable. So for us to launch our container process we are going
to 'self exec' inorder to launch a completely new process.

code showing the self exec process

{% highlight code %}

import (
	"fmt"
	"os"
	"os/exec"
	"syscall"
)

func main() {
	//cmd := exec.Command("/bin/sh") //execute a new shell process
    cmd := exec.Command("/proc/self/exe") // /proc/self/exe refers to the
current running process . This is the self exec'ng
	cmd.Stdin = os.Stdin
	cmd.Stderr = os.Stderr
	cmd.Stdout = os.Stdout

	if err := cmd.Run(); err != nil {
		fmt.Println("Error running %v", err)
	}
}

{% endhighlight %}

### **2. Safely using Namespaces**

To demonstrate this we have the code below. In a separate section we are going
to do a deep dive about network namespaces but for now we can 'accept' that the
code below creates a new network namespace and prints the ID of the network namespace before and
after creation. 

{% highlight code %}
package main

import (
	"fmt"
	"sync"

	"github.com/vishvananda/netns"
)

func printCurNetNs(where string) {
	cur := getCurNetNs()
	fmt.Printf("[%s ] current net namespace %s \n", where, cur.String())
}

func getCurNetNs() netns.NsHandle {
	net, err := netns.Get()
	if err != nil {
		panic(err)
	}
	return net
}

func newNetNs() {
	if _, err := netns.New(); err != nil {
		panic(err)
	}

}

func main() {
	/*
		runtime.LockOSThread()
		defer runtime.UnlockOSThread()
	*/
	printCurNetNs("Before")
	newNetNs()
	printCurNetNs("After")
	curNet := getCurNetNs()
	var wg sync.WaitGroup
	numWorkers := 100
	matched := make(chan int, numWorkers)
	for i := 0; i <= numWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			cur := getCurNetNs()

			if cur.Equal(curNet) == true {
				matched <- 1
			}
		}()
	}
	wg.Wait()
	close(matched)
	matchedNet := 0
	for count := range matched {
		matchedNet += count
	}
	fmt.Printf("matched network %d \n", matchedNet)
}
{% endhighlight %}

{% highlight code %}
[Before ] current net namespace NS(3: 4, 4026532008) 
[After ] current net namespace NS(5: 4, 4026533561) 
matched network 22
{% endhighlight %}

From a high level view of the code the network namespace creation is 
sequential so we expect the subsequent references to the network namespace to
always point to the newly created one. However, as we can see from the sample
output the net namespaces that match the newly created namespace varies in
between runs. This is all due to multiplexing of the goroutines on the OS
threads. The following illustrations best show this. 


**Expected Sequential creation of net namespace**

![Expected Sequential](/assets/sequential_expected.png){:class="img-responsive"}



**What is happening under the hood**
![multiplexing](/assets/multiplexing.png){:class="img-responsive"}



This goroutine multiplexing issue is not new and can be fixed by locking the OS
thread but we also have to be careful as no other goroutine can run on that
thread. To see what this means just uncomment the locking code segment and the
results show consistently that no network matches the newly created one.
Basically the OS thread that creates the namespace is not the one fetching it.
All in all we need to be aware of these gotchas as they could be causes of
subtle bugs in our code.

On a final note we will be using **Cgroups Version 2** which is supposed to be an
improvement from the previous version. The journey for us will begin by us
creating a process , attaching it to the various namespaces , dealing with
capabilities and then finally applying the various cgroups.


**References**

1. [Os thread launch flags](https://github.com/golang/go/blob/dev.boringcrypto.go1.14/src/runtime/os_linux.go#L127)
2. [Gvisor](https://github.com/google/gvisor)
3. [Scheduling of goroutines](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)
4. [Golang no forking](https://thorstenball.com/blog/2014/10/13/why-threads-cant-fork/)
5. [Golang lock OS thread](https://devdocs.io/go/runtime/index#LockOSThread)
6. [Golang and namespaces](https://www.weave.works/blog/linux-namespaces-and-go-don-t-mix) and [follow up](https://www.weave.works/blog/linux-namespaces-golang-followup)
