## Moving Riak Configuration Forward ##

I started using Riak in 2010. My first Riak powered proof of concept application was built on Riak 0.10.1. Here's the config file I used to start that application:

```erlang
%% -*- tab-width: 4;erlang-indent-level: 4;indent-tabs-mode: nil -*-
%% ex: ts=4 sw=4 et
[
 %% Riak Core config
 {riak_core, [
              %% Default location of ringstate
              {ring_state_dir, "data/ring"},

              %% riak_web_ip is the IP address that Riak's HTTP interface will
              %%  bind to.  If this is undefined, the HTTP interface will not run.
              {web_ip, "127.0.0.1"},

              %% riak_web_port is the TCP port that Riak's HTTP interface will
              %% bind to.
              {web_port, 8098}
             ]},

 %% Riak KV config
 {riak_kv, [
            %% Storage_backend specifies the Erlang module defining the storage
            %% mechanism that will be used on this node.
            {storage_backend, riak_kv_dets_backend},

            %% Different storage backends can use other configuration variables.
            %%  For instance, riak_dets_backend_root determines the directory
            %%  under which dets files will be placed.
            {riak_kv_dets_backend_root, "data/dets"},

            %% riak_handoff_port is the TCP port that Riak uses for
            %% intra-cluster data handoff.
            {handoff_port, 8099},

            %% pb_ip is the IP address that Riak's Protocol Buffers interface
            %% will bid to.  If this is undefined, the interface will not run.
            {pb_ip,   "0.0.0.0"},

            %% pb_port is the TCP port that Riak's Protocol Buffers interface
            %% will bind to
            {pb_port, 8087},

            %% raw_name is the first part of all URLS used by Riak's raw HTTP
            %% interface.  See riak_web.erl and raw_http_resource.erl for
            %% details.
            %{raw_name, "riak"},

            %% mapred_name is URL used to submit map/reduce requests to Riak.
            {mapred_name, "mapred"},

            %% js_vm_count is the number of Javascript VMs to start per Riak
            %% node.  8 is a good default for smaller installations. A larger
            %% number like 12 or 16 is appropriate for installations handling
            %% lots of map/reduce processing.
            {js_vm_count, 8},

            %% js_source_dir should point to a directory containing Javascript
            %% source files which will be loaded by Riak when it initializes
            %% Javascript VMs.
            %{js_source_dir, "/tmp/js_source"}
         
            %% riak_stat enables the use of the "riak-admin status" command to 
            %% retrieve information the Riak node for performance and debugging needs
            {riak_kv_stat, true}
           ]},

 %% SASL config
 {sasl, [
         {sasl_error_logger, {file, "log/sasl-error.log"}},
         {errlog_type, error},
         {error_logger_mf_dir, "log/sasl"},      % Log directory
         {error_logger_mf_maxbytes, 10485760},   % 10 MB max file size
         {error_logger_mf_maxfiles, 5}           % 5 files max
         ]}
].
```

And of course, we cannot forget vm.args:

```
## Name of the riak node
-name dev1@127.0.0.1

## Cookie for distributed erlang
-setcookie riak

## Heartbeat management; auto-restarts VM if it dies or becomes unresponsive
## (Disabled by default..use with caution!)
##-heart

## Enable kernel poll and a few async threads
+K true
+A 5

## Increase number of concurrent ports/sockets
-env ERL_MAX_PORTS 4096

## Tweak GC to run more often 
-env ERL_FULLSWEEP_AFTER 10
```

Guess how many times I broke this configuration? A lot.

Forget a comma? Your configuration broke. 

Didn't close that bracket? Too bad. 

Not an Erlang developer? Well, you should be. 

So that's what I did. Now I'm an Erlang developer, but I never really shook that feeling I had before that when it seemed to me that this configuration system was not just too hard, but harder than it needed to be.

But why was it like that? It's because that's how Erlang applications were deployed and there were hard database problems to solve, so we used the system that we got for free with Erlang.

I've already said 'Erlang' too many times for this post. Please don't think you need to know Erlang, what I'm saying here is that a user shouldn't need to know anything about Erlang to configure Riak, but you should probably know it exists for the historic context of Riak configuration.

Back to my story.

Erlang gets a bad rap for it's syntax. I don't mean to get into that debate. But I certainly agree that this is a bad syntax for the user facing configuration of any application. But I lived with it, just like Basho lived with it. Why not? There were only a few values to be set, and then I can get back to development. So we all lived with it. As time went on, the product and feature set grew. As new features showed up, so did new configuration knobs.

A developer adds a feature, and that feature needs a configuration setting. That developer is writing the feature in Erlang, and chances are, they're writing a function that takes this configuration setting as an argument. I'm going to pick on anti_entropy here for a minute, because it's a very clear and simple case of this.

Here's the function that extracts anti_entropy from the riak_kv config:

<a href="https://github.com/basho/riak_kv/blob/1.4.2/src/riak_kv_entropy_manager.erl#L309-320">riak_kv_entropy_manager</a>

```erlang
settings() ->
    case app_helper:get_env(riak_kv, anti_entropy, {off, []}) of
        {on, Opts} ->
            {true, Opts};
        {off, Opts} ->
            {false, Opts};
        X ->
            lager:warning("Invalid setting for riak_kv/anti_entropy: ~p", [X]),
            application:set_env(riak_kv, anti_entropy, {off, []}),
            {false, []}
    end.
```

It's built to recieve one of the following three values:

* `{on, []}`
* `{off, []}`
* `{on, [debug]}`

If you wanted to configure just this, it would look like this:

```erlang
%% cool syntax, bro
[{riak_kv, [
    {anti_entropy, {on, [debug]}}
    ]}
].
```

<a href="https://github.com/basho/riak_kv/blob/1.4.2/src/riak_kv_entropy_manager.erl#L185-L195">init/0</a>

Why? Because Erlang likes to read in options as a list ([debug]), so we give it to erlang in a syntax it likes. There's nothing wrong with that! It's how Erlang is done. But it got me thinking: What does a user really need to know about all this?

### NOTHING!

What I want to know from the user is: "Hey, do you want your anti_entropy on, off or in debug mode?"

So, I got to thinking about all the things that have changed since I started using Riak. I've learned more about the different kinds of users for configuration of systems. Not just the developer configuring a few nodes locally, but also automated deployment tools. I came up with a few things I wanted out of a better configuration system.

**Everything you need to know about a single configuration setting should be on a single line**
I didn't want to see something that wasn't easy to read, so eliminate the clutter. You could technically put an entire Riak app.config on a single line, which would make it even harder to read. Riak's app.config also has a nested structure, so you can't really tell what a setting is doing by reading a single line (or using a tool like grep). You need to have a concept of which Erlang library within Riak that a setting belongs to, and look in that block of config.

**That line can be anywhere in the file**
One of my least favorite parts of configuring Riak was that the settings were in an Erlang proplist. That means that if you want comment out the last setting, you also need to comment out the comma on the previous line. If you don't, you break Erlang's list syntax, which I already think that you shouldn't have to know. You should be able to organize the file how you want.

**If you set something more than once, the last one wins**
Sometimes you want to change some settings to test, just to see how it feels. You should be able to put those all at the end of your configuration file for testing. For an automated deployment tool, this means, you can append to configuration file to change a value

**Key/Value is a pretty good idea**
A setting has a key and a value. This started to remind me of sysctl.

So, what did I want to give the user? A configuration experience that looks like this:

```properties
## enable active anti-entropy subsystem
## possible values: on, off, debug
anti_entropy = on
```

No trailing commas, no Erlang lists. No nested settings.

**And so I give you, Cuttlefish**
<a href="http://github.com/basho/cuttlefish">Cuttlefish Github Repository</a>

What is it? It's another thing that you don't need to know about. What it does is give you a sysctl like syntax to write configuration files that are converted into files that Erlang loves (i.e. app.config and vm.args). It's an Erlang library, for which you can write custom schemas. All that means is that it can be used for any Erlang application, not just Riak.

Riak 2.0 will be the first release of Riak to ship with a riak.conf file, in stead of app.config and vm.args. In this file, new deployments of Riak will immediately be using the new configuration syntax from the get go. If you're upgrading to 2.0, fear not! If you have an existing app.config and vm.args, they will be detected and used, bypassing the riak.conf file until you choose to migrate.

Back to anti_entropy... that's not all, because I wanted a user to able to do some commandline fu like this:

```
➜  riak git:(jd-cuttlefish) ✗ cat riak.conf| grep anti_entropy
anti_entropy = on
anti_entropy.build_limit.number = 1
anti_entropy.build_limit.per_timespan = 1h
anti_entropy.expire = 1w
anti_entropy.concurrency = 2
anti_entropy.tick = 15s
anti_entropy.data_dir = ./data/anti_entropy
anti_entropy.write_buffer_size = 4MB
anti_entropy.max_open_files = 20
```

Wow! All the anti_entropy settings for a node, right there for you to look at without the clutter of the rest of the file.

Here's the default riak.conf file that will ship with 2.0. I think it's a lot cleaner and a lot easier to configure by both humans and deployment tools.

```properties
## The enabled Yokozuna set this 'on'.
yokozuna = off

## The port number which Solr binds to.
yokozuna.solr_port = 8093

## The port number which Solr JMX binds to.
yokozuna.solr_jmx_port = 8985

## The arguments to pass to the Solr JVM.  Non-standard
## arguments, i.e. -XX, may not be portable across JVM
## implementations.  E.g. -XX:+UseCompressedStrings.
yokozuna.solr_jvm_args = -Xms1g -Xmx1g -XX:+UseStringCache -XX:+UseCompressedOops

## The data under which to store all Yokozuna related data.
## Including the Solr index data.
yokozuna.data_dir = ./data/yz

## leveldb data_root
leveldb.data_root = ./data/leveldb

## This parameter defines the percentage, 1 to 100, of total
## server memory to assign to leveldb.  leveldb will dynamically
## adjust it internal cache sizes as Riak activates / inactivates
## vnodes on this server to stay within this size.  The memory size
## can alternately be assigned as a byte count via total_leveldb_mem instead.
leveldb.total_leveldb_mem_percent = 90

## Each database .sst table file can include an optional "bloom filter"
## that is highly effective in shortcutting data queries that are destined
## to not find the requested key. The bloom_filter typically increases the
## size of an .sst table file by about 2%. This option must be set to true
## in the riak.conf to take effect.
leveldb.bloomfilter = on

## Set to 'off' to disable the admin panel.
riak_control = off

## Authentication style used for access to the admin
## panel. Valid styles are: off, userlist
riak_control.auth = userlist

## If auth is set to 'userlist' then this is the
## list of usernames and passwords for access to the
## admin panel.
riak_control.user.user.password = pass

## bitcask data root
bitcask.data_root = ./data/bitcask

## Configure how Bitcask writes data to disk.
## erlang: Erlang's built-in file API
## nif: Direct calls to the POSIX C API
## The NIF mode provides higher throughput for certain
## workloads, but has the potential to negatively impact
## the Erlang VM, leading to higher worst-case latencies
## and possible throughput collapse.
bitcask.io_mode = erlang

## enable active anti-entropy subsystem
anti_entropy = on

## Storage_backend specifies the Erlang module defining the storage
## mechanism that will be used on this node.
storage_backend = bitcask

## raw_name is the first part of all URLS used by the Riak raw HTTP
## interface.  See riak_web.erl and raw_http_resource.erl for
## details.
## raw_name = riak

## Restrict how fast AAE can build hash trees. Building the tree
## for a given partition requires a full scan over that partition's
## data. Once built, trees stay built until they are expired.
## Config is of the form:
## {num-builds, per-timespan}
## Default is 1 build per hour.
anti_entropy.build_limit.number = 1

anti_entropy.build_limit.per_timespan = 1h

## Determine how often hash trees are expired after being built.
## Periodically expiring a hash tree ensures the on-disk hash tree
## data stays consistent with the actual k/v backend data. It also
## helps Riak identify silent disk failures and bit rot. However,
## expiration is not needed for normal AAE operation and should be
## infrequent for performance reasons. The time is specified in
## milliseconds. The default is 1 week.
anti_entropy.expire = 1w

## Limit how many AAE exchanges/builds can happen concurrently.
anti_entropy.concurrency = 2

## The tick determines how often the AAE manager looks for work
## to do (building/expiring trees, triggering exchanges, etc).
## The default is every 15 seconds. Lowering this value will
## speedup the rate that all replicas are synced across the cluster.
## Increasing the value is not recommended.
anti_entropy.tick = 15s

## The directory where AAE hash trees are stored.
anti_entropy.data_dir = ./data/anti_entropy

## The LevelDB options used by AAE to generate the LevelDB-backed
## on-disk hashtrees.
anti_entropy.write_buffer_size = 4MB

anti_entropy.max_open_files = 20

## mapred_name is URL used to submit map/reduce requests to Riak.
mapred_name = mapred

## mapred_2i_pipe indicates whether secondary-index
## MapReduce inputs are queued in parallel via their own
## pipe ('true'), or serially via a helper process
## ('false' or undefined).  Set to 'false' or leave
## undefined during a rolling upgrade from 1.0.
mapred_2i_pipe = on

## Each of the following entries control how many Javascript
## virtual machines are available for executing map, reduce,
## pre- and post-commit hook functions.
javascript_vm.map_count = 8

javascript_vm.reduce_count = 6

javascript_vm.hook_count = 2

## js_max_vm_mem is the maximum amount of memory, in megabytes,
## allocated to the Javascript VMs. If unset, the default is
## 8MB.
javascript_vm.max_vm_mem = 8

## js_thread_stack is the maximum amount of thread stack, in megabyes,
## allocate to the Javascript VMs. If unset, the default is 16MB.
## NOTE: This is not the same as the C thread stack.
javascript_vm.thread_stack = 16

## js_source_dir should point to a directory containing Javascript
## source files which will be loaded by Riak when it initializes
## Javascript VMs.
## javascript_vm.source_dir = /tmp/js_source

## http_url_encoding determines how Riak treats URL encoded
## buckets, keys, and links over the REST API. When set to 'on'
## Riak always decodes encoded values sent as URLs and Headers.
## Otherwise, Riak defaults to compatibility mode where links
## are decoded, but buckets and keys are not. The compatibility
## mode will be removed in a future release.
http_url_encoding = on

## Switch to vnode-based vclocks rather than client ids.  This
## significantly reduces the number of vclock entries.
## Only set on if *all* nodes in the cluster are upgraded to 1.0
vnode_vclocks = on

## This option toggles compatibility of keylisting with 1.0
## and earlier versions.  Once a rolling upgrade to a version
## > 1.0 is completed for a cluster, this should be set to
## true for better control of memory usage during key listing
## operations
listkeys_backpressure = on

## This option specifies how many of each type of fsm may exist
## concurrently.  This is for overload protection and is a new
## mechanism that obsoletes 1.3's health checks. Note that this number
## represents two potential processes, so +P in vm.args should be at
## least 3X the fsm_limit.
fsm_limit = 50000

## retry_put_coordinator_failure will enable/disable the
## 'retry_put_coordinator_failure' per-operation option of the
## put FSM.
## on = Riak 2.0 behavior (strongly recommended)
## off = Riak 1.x behavior
retry_put_coordinator_failure = on

## object_format controls which binary representation of a riak_object
## is stored on disk.
## Current options are: v0, v1.
## v0: Original erlang:term_to_binary format. Higher space overhead.
## v1: New format for more compact storage of small values.
object_format = v1

## listener.http.<name> is an IP address and TCP port that the Riak
## HTTP interface will bind.
listener.http.internal = 127.0.0.1:8098

## listener.protobuf.<name> is an IP address and TCP port that the Riak
## Protocol Buffers interface will bind.
listener.protobuf.internal = 127.0.0.1:8087

## pb_backlog is the maximum length to which the queue of pending
## connections may grow. If set, it must be an integer >= 0.
## By default the value is 5. If you anticipate a huge number of
## connections being initialised *simultaneously*, set this number
## higher.
## protobuf.backlog = 64

## listener.https.<name> is an IP address and TCP port that the Riak
## HTTPS interface will bind.
## listener.https.internal = 127.0.0.1:8098

## the number of replicas stored. Note: See CAP Controls for further discussion.
## http://docs.basho.com/riak/latest/dev/advanced/cap-controls/
buckets.default.n_val = 3

buckets.default.pr = 0

buckets.default.r = quorum

buckets.default.w = quorum

buckets.default.pw = 0

buckets.default.dw = quorum

buckets.default.rw = quorum

## whether or not siblings are allowed.
## Note: See Vector Clocks for a discussion of sibling resolution.
buckets.default.siblings = on

buckets.default.last_write_wins = false

## Default ring creation size.  Make sure it is a power of 2,
## e.g. 16, 32, 64, 128, 256, 512 etc
## ring_size = 64

## Default location of ringstate
ring.state_dir = ./data/ring

## Default cert location for https can be overridden
## with the ssl config variable, for example:
## ssl.certfile = ./etc/cert.pem

## Default key location for https can be overridden
## with the ssl config variable, for example:
## ssl.keyfile = ./etc/key.pem

## Default signing authority location for https can be overridden
## with the ssl config variable, for example:
## ssl.cacertfile = ./etc/cacertfile.pem

## handoff.port is the TCP port that Riak uses for
## intra-cluster data handoff.
handoff.port = 8099

## To encrypt riak_core intra-cluster data handoff traffic,
## uncomment the following line and edit its path to an
## appropriate certfile and keyfile.  (This example uses a
## single file with both items concatenated together.)
## handoff.ssl.certfile = /tmp/erlserver.pem

## DTrace support
## Do not enable 'dtrace' unless your Erlang/OTP
## runtime is compiled to support DTrace.  DTrace is
## available in R15B01 (supported by the Erlang/OTP
## official source package) and in R14B04 via a custom
## source repository & branch.
dtrace = off

platform_bin_dir = ./bin

platform_data_dir = ./data

platform_etc_dir = ./etc

platform_lib_dir = ./lib

platform_log_dir = ./log

## where do you want the console.log output:
## off : nowhere
## file: the file specified by log.console.file
## console : standard out
## both : log.console.file and standard out.
log.console = file

## the log level of the console log
log.console.level = info

## location of the console log
log.console.file = ./log/console.log

## location of the error log
log.error.file = ./log/error.log

## turn on syslog
log.syslog = off

## Whether to write a crash log, and where.
## Commented/omitted/undefined means no crash logger.
log.crash.file = ./log/crash.log

## Maximum size in bytes of events in the crash log - defaults to 65536
log.crash.msg_size = 64KB

## Maximum size of the crash log in bytes, before its rotated, set
## to 0 to disable rotation - default is 0
log.crash.size = 10MB

## What time to rotate the crash log - default is no time
## rotation. See the lager README for a description of this format:
## https://github.com/basho/lager/blob/master/README.org
log.crash.date = $D0

## Number of rotated crash logs to keep, 0 means keep only the
## current one - default is 0
log.crash.count = 5

## Whether to redirect error_logger messages into lager - defaults to true
log.error.redirect = on

## maximum number of error_logger messages to handle in a second
## lager 2.0.0 shipped with a limit of 50, which is a little low for riak's startup
log.error.messages_per_second = 100

## Name of the riak node
nodename = riak@127.0.0.1

## Cookie for distributed node communication.  All nodes in the same cluster
## should use the same cookie or they will not be able to communicate.
distributed_cookie = riak

erlang.async_threads = 64

## Increase number of concurrent ports/sockets
erlang.max_ports = 64000

## Set the location of crash dumps
erlang.crash_dump = ./log/erl_crash.dump

## Raise the ETS table limit
erlang.max_ets_tables = 256000

## Raise the default erlang process limit
erlang.process_limit = 256000

## For nodes with many busy_dist_port events, Basho recommends
## raising the sender-side network distribution buffer size.
## 32MB may not be sufficient for some workloads and is a suggested
## starting point.
## The Erlang/OTP default is 1024 (1 megabyte).
## See: http://www.erlang.org/doc/man/erl.html#%2bzdbbl
erlang.zdbbl = 32MB

## Erlang VM scheduler tuning.
## Prerequisite: a patched VM from Basho, or a VM compiled separately
## with this patch applied:
## https://gist.github.com/evanmcc/a599f4c6374338ed672e
## erlang.sfwi = 500

## Reading or writing objects bigger than this size will write
## a warning in the logs.
warn_object_size = 5MB

## Writing an object bigger than this will fail.
max_object_size = 50MB

## Writing an object with more than this number of siblings
## will generate a warning in the logs.
warn_siblings = 25

## Writing an object with more than this number of siblings
## will fail.
max_siblings = 100
```