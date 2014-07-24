One of the tools you can use in your “Performance Toolbox” with Drupal is Memcached. 
Memcached is a general-purpose distributed memory caching system. It is used to speed up dynamic database-driven websites (such as Drupal) by caching data and objects in memory to reduce the number of times an external data source (such as a database or API) is called.

In other words, it’s faster to pull something from memory than a database or the filesystem. Think of it like handing a friend a piece of paper with a number written on it. He pops it into his pocket and forgets about it. When you ask him for the number, he has to pull the paper out of his pocket, read the number and then tell you what’s written on the paper. If he had memorised the number, it would take him less time and effort, he can just tell you what you need to know directly from memory. This is effectively how caching works with drupal. Instead of bootstrapping drupal which involves a number of database queries, memcache can supply the data if it has it in memory, saving drupal some time and effort.

Combining this with APC to cache php opcodes and Varnish as a reverse proxy, you can build a very robust Drupal implementation that can handle large volumes of traffic with ease. Offloading different parts of the application to different tools designed to handle that work load. Making your website “distribute” it’s workload across multiple machines.

John Allspaw was the data operations manager for Flickr. He provides some interesting insights in his book; “The Art of Capacity Planning” with regard to leveraging this approach. He discovered that thumbnail generation was very resource intensive, so he split this job out onto it’s own dedicated resources so that the server wouldn’t slow down the website each time it needed to generate thumbnails. Effectively handing the task to another machine to do the work, while the rest of the architecture got on with the business of delivering your photos on Flickr.

We can apply this approach in how we use Memcached, by splitting up our caches and assigning them different memory sizes, enable logging per cache to help debug problems and potentially offloading them on to their own boxes. This will provide horizontal scaling to our architecture. Instead of putting in another web head to your cluster, you can deploy a dedicated memcache box and it will require less resources than an additional web server and cost a lot less.
Sounds fun, let’s do it! … erm … now what?

Here’s what we want to do.

Create a memcached instance for each drupal cache
Allocate different memory allotments for each one
Teach Drupal that memcached is running in multiple places
Provide a simple setup using a config file
Add logging to Memcache
Identify ways to see what memcache is doing

Lets start by looking at how memcache is started. You can manage it through a service using:

sudo service memcached {start | stop | restart | status}

This calls an init script  /etc/init.d/memcached opening the script, we can see it calls memcached as a daemon and passes some simple parameters to the daemon.

eg. This part defines our parameters to pass to the daemon on start: 

PORT=11211
USER=memcached
MAXCONN=1024
CACHESIZE=64
OPTIONS=""

which is then called from this line:

daemon --pidfile ${pidfile} memcached -d -p $PORT -u $USER  -m $CACHESIZE -c $MAXCONN -P ${pidfile} $OPTIONS

Explanation of what it does: this starts memcache listening on port 11211 and allows for a maximum connection limit of 1024 and allocates 64m of memory for the cache. 

But how about if we decide to spawn multiple versions of Memcached in memory all assigned to different tasks and different memory allocations. eg: 

daemon memcached -d -p 11211 -u $USER  -m 256 -c $MAXCONN  $OPTIONS  2>&1
daemon memcached -d -p 11212 -u $USER  -m 512 -c $MAXCONN  $OPTIONS  2>&1
daemon memcached -d -p 11213 -u $USER  -m 128 -c $MAXCONN  $OPTIONS  2>&1
daemon memcached -d -p 11214 -u $USER  -m 512 -c $MAXCONN  $OPTIONS  2>&1

This will spawn memcached a number of times and listen on different ports (11211-11215) using different memory allocations for each instance. We can now assign drupal to assign each instance to use a different cache. In our settings.php we can do this using the following snippet.

/**
 * Memcached configuration
 */
$conf['cache_backends'][] = 'modules/memcache/memcache.inc';
$conf['memcache_key_prefix'] = 'drupal';
$conf['memcache_persistent'] = TRUE;
$conf['memcache_stampede_protection'] = TRUE;
$conf['cache_default_class'] = 'MemCacheDrupal';
$conf['lock_inc'] = 'modules/memcache/memcache-lock.inc';

$conf['memcache_servers'] = array(
  'my.webserver.number.1.com:11211' => 'default',
  'my.webserver.number.1.com:11212' => 'form',
  'my.webserver.number.1.com:11213' => 'semaphore',
  'my.webserver.number.1.com:11214' => 'page',
  'my.webserver.number.2.com:11211' => 'default',
  'my.webserver.number.2.com:11212' => 'form',
  'my.webserver.number.2.com:11213' => 'semaphore',
  'my.webserver.number.2.com:11214' => 'page',
);
$conf['memcache_bins'] = array(
  'cache' => 'default',
  'cache_form' => 'form',
  'semaphore' => 'semaphore',
  'cache_page' => 'page',
);

This tells drupal that memcached is running multiple instances on 2 different web servers. The cache is then divided according to it’s function. Form cache needs more memory than semaphore, so we have allocated 512m for that service on port 11212 and semaphore needs less so we set it to 128m and it runs on 11213.

If we want to provide logging, we can set the verbosity level in the options section and then pipe the output from memcache to a log file.

So in the config section, we can add verbosity to the OPTIONS variable.

OPTIONS="vvv"
LOGPATH="/var/log/memcache_"

and add a pipe to the daemon command:

daemon memcached -d -p 11211 -u $USER  -m 256 -c $MAXCONN  $OPTIONS >> ${LOGPATH}default.log  2>&1

This would start the default cache on port 11211, with 256m of memory and produce a log file under /var/log/memcache_default.log

So how do we put this all together?

It would be good if we can use this distributed memcache setup as a replacement for the existing memcache setup. This can be done by replicating the memcache init scripts and producing a new one specifically for this setup, for our usage, we can call our script as memcached_drupal, so to start it we would use:

sudo service memcached_drupal {start | stop | restart | status}

To make it a little more fun, we can split out the configuration into a new file, /etc/sysconfig/memcached_drupal and define the run levels we want it to run in the comments at the top. So our header on our version of the script looks like this.

#! /bin/sh
# chkconfig: 345 55 45
# description:Custom memcache init script - Spawns multiple  memcache instances
# processname: memcached_drupal
# config: /etc/sysconfig/memcache_drupal

The line with chkconfig tells us which run levels we want this service to run in, in this case 3,4 and 5. Our config file has been moved out to /etc/sysconfig/memcache_drupal and I have added a variable to turn logging on / off.




