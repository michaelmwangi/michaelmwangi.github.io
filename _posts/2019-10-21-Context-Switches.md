---
layout: post
title:  "Cost of Context Switches"
date:   2019-10-21 13:18:54 +0400
categories: jekyll update
---

In the last article we looked at how we can optimize application performance by utilizing connection pools. But like everything else in computing, it's no free lunch hence we don't always have a linear correlation between application performance and the size of the connection pool. At some point the application performance starts to deteriorate even with a large db pool. The root of this discrepancy can be attributed to **concurrency** where we have interleaved execution of tasks unlike in **parallelism** where we have simultaneous task execution.


The linux scheduler allocates tasks on the processor temporarily in units of **time slices**. In most cases we have a multi-processor setup and each processor has a dedicated queue from which it gets its tasks for execution. This means that unless a task is moved to another processor's queue it has to wait for the current executing task's time slice to end before it can be executed.


The image below illustrates these two forms of execution.The first image depicts a system which only has one processor and the second image shows a system which has 2 processors available. In both cases we assume that there are only two tasks to execute (**A** and **B**).



![Concurrency VS Parallelism](/assets/concurrencyVSparallelism.png){:class="img-responsive"}




From the first image we see that the processor is switching between the two tasks implying that as the switching is happening the processor has to store the state of **task A** before switching to **task B** so that it can be resumed later. This is known as context switchig and it has a cost . **Time**. It takes time to store and retrieve the state of a task. 

Apart from this scenario context switching can be triggered during an **I/O operation** eg task waiting for data to arrive from the network. We can categorize context switches into two classes. 

1. **Voluntary** - happens when the task willfully gives up the processor eg during I/O operation
2. **Involuntary** - happens when the task's allocated time slice by the scheduler is exhausted.



For the purpose of this discussion we are going to limit ourselves to a system which only has one processor and for us to simulate this we are going to use the **taskset** linux utility which will help us pin processes to one processor  We are also going to be using a connection pool from the postgresql database as it adopts a process per connection model. This means that if we create a DB connection pool of size 100 postgres will launch atleast 100 processes. You can check this by running 

{% highlight code %}
     sudo lsof -i :5432
{% endhighlight %}


For the DB connection pool code we will basically be using an extension of the connection pool golang code used earlier. As for the DB schema we will use a simple setup of just a single table of names. And fill it up with a bunch of random names. For my setup I used around 10000 entries.

{% highlight code %} 

    CREATE TABLE names (
        id integer NOT NULL,
        name character varying(255) NOT NULL
        );

{% endhighlight %}

Here are  brief snippets of the code. For the full code see [here](https://github.com/michaelmwangi/blog/tree/master/contextswitch)

Establishing connection pool 

{% highlight code %} 

    var wg sync.WaitGroup

    func dummySelect(db *sql.DB){
	    _, err := db.Exec("SELECT 1 FROM names")
    	if err != nil{
		    fmt.Println(err)
	    }
    }     

    for i := 0; i < poolSize ; i++ {
		wg.Add(1)
		go func(){
			defer wg.Done()
			dummySelect(db)
		}()
	}
	
    func fetchData(db *sql.DB){
	    //Using random ID for each select so as to avoid cache effects from DB 
	    id := rand.Intn(10000) 
	    var name string
	    if err := db.QueryRow("SELECT name FROM names WHERE id=$1", id).Scan(&name); err != nil {
		    fmt.Println(err)
	    }
    }


	wg.Wait()


	// the benchmark tests starts from here by launching num workerSize workers concurrently 
	start := time.Now()

	for i := 0; i < workerSize ; i++ {
		wg.Add(1)
		go func(){
			defer wg.Done()
			fetchData(db)
		}()
	}
	
	wg.Wait() // wait for all the workers to finish up

	end := time.Now()
	delta := end.Sub(start).Nanoseconds()
     
{% endhighlight %}   

As for the pinning of the Postgresql "master" process we are going to pin it to CPU-0

{% highlight code %}
        sudo taskset -pc 0 [process id of the postresql master process]
{% endhighlight %}

As for running the code we are also going to be using the [taskset](https://linux.die.net/man/1/taskset) utility but also counting the number of context switches occuring in CPU-0. To achieve this we are going to use a script

{% highlight code %}
    #!/bin/bash
    echo  "starting ...."
    taskset -c 1 go run main.go &
    sudo perf stat -C 0  -- sleep 1
 {% endhighlight %}    

 Noticeably we are using [perf](http://man7.org/linux/man-pages/man1/perf.1.html) to measure the number of context switches in the CPU. In a nutshell perf is a linux perfomance analysis tool that can do way more than count context switches.

 By changing the poolSize variable on the golang code in between runs of the script above we see that a high poolSize increases query time as well as an increase in number of contextswitches while the opposite happens for a small poolSize number.


Here are some sample results from a run on my machine.


{% highlight code %}
    starting ....
    Worker Size: 1000 
    Pool Size: 10 
    Time Taken: 103.932812 ms

    Performance counter stats for 'CPU(s) 0':

          1,001.19 msec cpu-clock                 #    1.000 CPUs utilized          
             2,643      context-switches          #    0.003 M/sec                  
    ......
{% endhighlight %}


{% highlight code %}
    starting ....
    Worker Size: 1000 
    Pool Size: 90 
    Time Taken: 126.070641 ms

    Performance counter stats for 'CPU(s) 0':

          1,001.06 msec cpu-clock                 #    1.000 CPUs utilized          
          2,964      context-switches          #    0.003 M/sec          
     .......     
{% endhighlight %}


As can be seen from the results a small pool size of 10 executes in less time than that of a size of 90. Also we see that context switches on the bigger pool size are more than when we have a small pool size. To verify this you can run the [runner.sh](https://github.com/michaelmwangi/blog/blob/master/contextswitch/runner.sh) script from the code multiple times to get consistent results.

**N:B** - When running this make sure you pin the Postgres master process to CPU-0

Ofcourse since am running the above test on my laptop which has other processes running the context switches observed are not all from the DB pool effects but also from other system processes. But since we are running the tests on the same CPU we can make the assumption that the "noise" from the other processes is fairly constant.

Due to different system design in terms of resources there isn't a magic connection pool number. The most optmimum size can only be determined by monitoring your application performance and also the system itself.


From the above experiment a follow up question would be what are the effects of a connection pool size on systems that don't use the same connection model as Postgres eg Redis which uses an event loop and is single threaded? 


