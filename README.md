# custom-memcached-server

Memcached is a free, open source, high-performance, distributed memory object caching system. It is intended for use in speeding up dynamic web applications by reducing database load.

Memcached is an in-memory key-value store for small chunks of arbitrary data retrieved from back end systems with higher latency. It is simple yet powerful.

Its simple design promotes quick deployment, ease of development, and solves many problems facing large data caches. Its relatively simple API is available for most popular languages.

Memcached was originally developed by Brad Fitzpatrick for LiveJournal in 2003. It was originally written in Perl and later ported to C.

Memcached supports a client-server architecture with each client aware of all of the servers, but the servers are unaware of each other and do not communicate.

If a client wishes to set or read the value corresponding to a certain key, the client's library first computes a hash of the key to determine which server to use. This provides a simple form of sharding and creates a highly-scalable shared-nothing architecture across the Memcached servers.

When the server receives a request it computes a second hash of the key to determine where to store/read the corresponding value. Values are stored in RAM; if a server runs out of RAM, it discards the oldest values. Memcached is as a transitory cache; client cannot assume that the data in still in cache when they need it.

Step Zero

Like all good software engineering we’re zero indexed! In this step you’re going to set your environment up ready to begin developing and testing your solution.

I’ll leave you to setup your IDE / editor of choice and programming language of choice. After that I’d suggest ensuring you have a telnet client read so you can test your server. Memcached uses a very simple text protocol which we can test with telnet.

Step 1

In this step your goal is to create a server that can start up, listen for, and accept an incoming TCP connection on the specified port. Memcached uses 11211 by default, so if no port is specified your server should use that.

Otherwise you should accept an option on the command line that matches -p <port number>

To support this your server will need to create a socket, and bind it to the address of your server. For the purposes of this challenge that can be the IP address: 127.0.0.1 (the loopback address, also known as localhost) and the specified port.

Once that is done, your server will need to listen for requests, accept incoming requests and then receive the incoming data.

You can learn more about sockets in the Wikipedia article on Berkeley Sockets. Your programming language probably provides a wrapper around this API in its standard library for example Python has socket, Rust has std::net and node has node:net.

For an in-depth look at network programming check out Beej’s Guide to Network Programming.

Once you have implemented this step, you should be able to launch your server with and without a specified port. That should look like:

% ccmemcached # starting on the default port
% ccmemcached -p 9999 # starting on port 9999
And you should be able to connect to it with Telnet, just it won’t do much yet:

% telnet localhost 11211
Trying ::1...
Connected to localhost.
Escape character is '^]'.
Step 2

In this step your goal is to handle the set and get commands. Memcached uses a text based protocol with commands taking the form:

<command name> <key> <flags> <exptime> <byte count> [noreply]\\r\\n
<data block>\\r\\n
The field flags is a is an arbitrary 16-bit unsigned integer (written out in decimal) that the server stores along with the data and sends back when the item is retrieved.

The field byte count is the number of bytes is the number of bytes in the data block to follow, not including the delimiting \\r\\n. The byte count field may be zero (in which case it's followed by an empty data block).

The field noreply is an optional parameter that instructs the server to not send the reply.

So the command to set a key would look like this:

set test 0 0 4\\r\\n
1234\\r\\n
If the above command is successful the server will respond with STORED\\r\\n if the data is stored.

And the command to read back the key looks like this:

get test\\r\\n
The response to the get command will either be END if the key is not found, or VALUE <data block> <flags> <byte count> if the key is found.

You should now implement both of these commands in your server, for the moment we will ignore the exptime field. The most obvious data structure to use for Memcached is the hash table.

So to test out server handles set and get we can use telnet with a session like:

% telnet localhost 11211
Trying ::1...
Connected to localhost.
Escape character is '^]'.
set test 0 0 4
1234
STORED
get test
VALUE test 0 4
1234
END
set test2 1 0 7 noreply
testing
get test2
VALUE test2 1 7
testing
END
to test that we can set and get values and that the flags are set and returned as well as the noreply being respected.

Step 3

If you haven’t already done so, now is the time to make your Memcached server handle concurrent clients. You have two basic options here; have one thread per client or use the asynchronous programming support offered by your chosen programming language. If you have time, give both a go, both have pros and cons.

To test this use Telnet to connect from two or more different terminals at the same time.

Step 4

In this step your goal is to support expiry of keys. The field <exptime> in the set command is expiration time. If it is zero, the item never expires. If it's non-zero it is the number of seconds into the future in which the data expires. It is guaranteed that clients will not be able to retrieve this item after the expiration time arrives (measured by server time). If a negative value is given the item is immediately expired.

You can test this with the following simple test cases via telnet:

set test 0 1 4
test
STORED
get test
END
set test 0 100 4
test
STORED
get test
VALUE test 0 4
test
END
set test 0 -1 4
test
STORED
get test
END
When it comes to implementing the expiry you might like to know that Memcached uses lazy expiration to remove data. This means that the data isn't removed from the server immediately when it expires. Instead when a client tries to access an expired key, Memcached checks the key, identifies that the key is expired, and then removes it from memory.

Step 5

In this step your goal is to support the add and replace operations. The add command stores the data, but only if the server doesn't already hold data for the key.

The replace command stores the data, but only if the server does **already hold data for the key. As for set and get commands the general format is:

<command name> <key> <flags> <exptime> <byte cound> [noreply]\\r\\n
<data block>\\r\\n
When the preconditions of add and replace are not met the server returns NOT_STORED otherwise it returns STORED.

You can test these as so:

set test 0 0 4
data
STORED
get test
VALUE test 0 4
data
END
add test 0 0 4
test
NOT_STORED
replace test 0 0 4
john
STORED
get test
VALUE test 0 4
john
END
replace test2 0 0 4
data
NOT_STORED
Step 6

In this step your goal is to support the append and prepend commands. The append command results in the data being added to an existing key after existing data. Whereas the prepend command results in adding the data to an existing key before the existing data.

You can test those with the following examples:

append test 0 0 4
more
STORED
get test
VALUE test 0 8
johnmore
END
prepend test 0 0 4
send
STORED
get test
VALUE test 0 12
sendjohnmore
END
append foo 0 0 4
test
NOT_STORED
prepend foo 0 0 4
test
NOT_STORED
Once all that works, congratulations you’ve built the core of a Memcached server!

Going Further

If you want to take this project further you can look at adding support for the cas (Check and Set) operation and the delete, increment and decrement operations. Finally limit the cache size and then put more data than that into it.
