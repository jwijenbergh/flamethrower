Flamethrower [![Build Status](https://travis-ci.org/DNS-OARC/flamethrower.svg?branch=master)](https://travis-ci.org/DNS-OARC/flamethrower)
============
> This project is in [active development](https://github.com/ns1/community/blob/master/project_status/ACTIVE_DEVELOPMENT.md).

A DNS performance and functional testing utility.

2017-2020© NSONE, Inc.

License
-------
This code is released under Apache License 2.0. You can find terms and conditions in the LICENSE file.


Overview
--------

Flamethrower is a small, fast, configurable tool for functional testing, benchmarking, and stress testing DNS servers and networks. It supports IPv4, IPv6, UDP, TCP, and DoT and has a modular system for generating queries used in the tests.

Originally built as an alternative to dnsperf (https://github.com/DNS-OARC/dnsperf), many of the command line options are compatible.

Dependencies
------------

* CMake >= 3.8
* Linux or OSX
* libuv >= 1.23.0
* libldns >= 1.7.0
* gnutls >= 3.3
* C++ compiler supporting C++17

Build
-----

CMake based, requires libuv and ldns.
```
mkdir build; cd build
cmake ..
make
```

Docker based, requires a recent version of docker.
```
org="myorg"
image="myflame"
tag="latest"
docker build --network host -t ${org}/${image}:${tag} -f Dockerfile .
docker run --rm -it --net host ${org}/${image}:${tag} --help
```

Usage
-----

Current command line options are described with:

```
flame --help
```

Quick Examples
--------

Flame localhost port 53, UDP, maximum speed:
```
flame localhost
```

Flame target, port 5300, TCP:
```
flame -p 5300 -P tcp target.test.com
```

Flame target, port 443, DoT:
```
flame -p 443 -P dot target.test.com
```

Flame target, DNS over HTTPS GET:
```
flame -P https target.test.com/dns-query
```

Flame target, DNS over HTTPS POST:
```
flame -P https -M POST target.test.com/dns-query
```

Flame target with random labels:
```
flame target.test.com -g randomlabel lblsize=10 lblcount=4 count=1000
```

Flame multiple target at once, reading the list from a file:
```
flame file --targets myresolvers.txt
```

Detailed Features
-----------------

### Query Generators

 Flamethrower uses a modular system for generating queries. Most modules generate all queries before sending begins, for performance reasons.
 Each module may include its own list of configuration options which can be set via key/value pairs on the command line.
 See full `--help` for the current list of generators and their options.

### Rate Limiting

 By default, Flamethrower will send traffic as fast as possible. To limit to a specific overall queries per second, use `-Q`

### Dynamic QPS Flow

 Flamethrower can adjust its QPS flow over time. This is useful for generating a "signal" of traffic (e.g. a square wave) for calibrating metrics collection. For example, to send 10 QPS for 120000ms, then 80 QPS for 120000ms, etc use `--qps-flow "10,120000;80,120000;10,120000;"`. Flow change will not loop, you should list as many changes as necessary. Once the flow reaches the final QPS number, it will hold it until program termination.

### Output Metrics

 Flamethrower can generate detailed metrics for each of its concurrent senders. Metrics include send and receive counts, timeouts, min, max and average latency, errors, and the like. The output format is JSON, and is suitable for ingestion into databases such as Elastic for further processing or visualization. See the `-o` flag.

### Concurrency

 Flamethrower is single threaded, async i/o. You specify the amount of concurrent senders with the `-c` option. Each of these senders will send a configurable number of consecutive queries (see `-q`), then enter a configurable delay period (see `-d`) before looping.

 Each concurrent sender will pull the next query from the total queries generated by the Query Generator, looping once it reaches the end of the query list (if the program is configured to continue).

 There is currently no built-in support for multiprocess sending, so the maximum throughput will be reached once a single CPU is saturated. However, you may manually start several concurrent `flame` processes, including up to 1 per CPU available. There is future planned support for builtin multiprocess sending.

Contributions
---
Pull Requests and issues are welcome. See the [NS1 Contribution Guidelines](https://github.com/ns1/community) for more information.
