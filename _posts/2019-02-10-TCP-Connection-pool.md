---
layout: post
title:  "TCP Connection Pool"
date:   2019-02-10 13:18:54 +0400
categories: jekyll update
---

Inorder for a tcp connection to be established it undergoes the famous 3-way handshake process

{% highlight code %}
                         SYN->SYN-ACK->ACK
{% endhighlight %}

This moves the TCP connection state to established . In linux to check the metrics of an established tcp connection we can use.

{% highlight code %}
                     ss  -neopt  state established 
{% endhighlight %}

We are going to be exploring the cost of establishing a new TCP connection vs using an already established one. I will be connecting to a locally running redis (port 6379) sending ping messages.

Below are sample functions for connecting using a pool and without. The full can be viewed [here](https://github.com/michaelmwangi/blog/tree/master/connectionpool)

**Not using a connection pool**

{% highlight golang %}
    func noPool(){
    	startTime := time.Now()
	    conn, err := createConnection("127.0.0.1:6379")
    	if err != nil{
		    fmt.Println(err)
	    }
	    defer conn.Close()
    	if _, err := send(conn, []byte("ping\r\n")); err != nil{
		    fmt.Println(err)
		    return
	    }

	    _, err = read(conn, 32)
	    if err != nil{
		    fmt.Println(err)
		    return 
	    }

	    endTime := time.Now()
	    diff := endTime.Sub(startTime)
	    fmt.Println("no pool ",diff)
    }
{% endhighlight %}

**Using a connection pool**

{% highlight golang %}
    func withPool(pool *TCPPool){
	    start := time.Now()
    	conn := pool.getConnection()
	    if conn == nil{
    		fmt.Println("Could not get a free connection")
		    return 
	    }

    	defer pool.putConnection(conn)
	    if _, err := send(conn, []byte("ping\r\n")); err != nil{
    		fmt.Println(err)
		    return
	    }

	    _, err := read(conn, 32)
    	if err != nil{
		    fmt.Println(err)
		    return 
	    }
	    end := time.Now()
    	diff := end.Sub(start)
	    fmt.Println("pool ",diff)
    }
{% endhighlight %}

On average I get shorter response times when using a pool than when creating a new connection everytime. Although I do get on a few occassions when the **_with pool time is greater than that of the with no pool_**. I still don't have an explanation for this.

Example response 

{% highlight code%}
    Starting
    pool   120.571µs
    no pool  148.335µs
{% endhighlight %}


The explanation for the lower times when using the TCPPool is simple we only pay the connection price (time) once when establishing the connection and reuse the established connections to send data. When not using a pool we have to establish a connection everytime we send data. So does it mean that we can initialize a large connection pool and expect linear performance gain from our application? Well no . In another post I will be exploring the cost of establishing many connections in a thread vs process model in relation to the number of CPUs available.
