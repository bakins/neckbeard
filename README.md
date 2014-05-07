Neckbeard
==========

Neckbeard is log forwarded intended to be used for logstash. It can take a "log stream" from a variety of inputs, transform it through external "pipes", wrap each event in the logstash json format, and send it to redis servers.

Motivation
----------
We could do the various transforms on the logstash servers, but we have issues with large numbers of servers.  Neckbeard allows us to push the task of getting the log events into proper json format to the servers themselves.

Status
------
Neckbeard is currently under heavy development.


# Usage #

Neckbeard has *inputs*, *pipes*, and *outputs*.  Log events are taken from an input, optionally ran through multiple pipes, wrapped in the logstash json envelope, then forwarded to redis.

## Inputs ##

Neckbeard can have one and only one input at this time. Run multiple instances for multiple inputs. Each event must be on a single line.

* `-file <path/to/file>` - read from a file in a manner like `tail -F`
* `-port <port>` - listen on a tcp port and read log events line-by-line
* `-stdin` - read log events from STDIN (This is the default if no option is given)
* `-exec "<path/to/program/with/args>"` - execute a program and read from its STDOUT. Any messages from STDERR will simply be written to the STDERR of neckbeard.  Neckbeard will manage the process and restart it if it fails, etc.

One, and only one, input may be used at this time. Neckbeard will exit with an error if this is not the case.

## Pipes ##

Neckbeard can optionally pass the output of the input through a series of "pipes."  These pipes are simply external programs. Neckbeard will write to the program's STDIN and read from it's STDOUT. . Any messages from STDERR will simple be written to the STDERR of neckbeard.  Neckbeard will manage the process and restart it if it fails, etc.

* `-pipe "<path/to/program/with/args>"` - pipe to use.  Pipes are ran in order given on the command line.

The last pipe should always generate valid json.  If no pipes are used then the input should be valid json. Json must be one message per line and a message can only be one line.

Neckeard inserts its own "internal pipe" at the end that wraps the json in the logstash json event envelope, the event from the input/pipes itself becomes `@fields.`

## Ouputs ##

Currently, Neckbeard can only output to a redis list. It will loadbalance between the servers, retry failed servers, etc.

* `-redis <ip>:<port>` - add this option multiple times for multiple servers
* `-redis-key <key>` - redis key use. Neckbeard uses [LPUSH](http://redis.io/commands/lpush). defaults to `logstash:neckbeard`
* `-redis-timeout <time>` - timeout for redis operations in milliseconds. Defaults to 250ms.
* `-redis-timer <time>` - if given, neckbeard will collect logs and attempt to send to redis every `<time>` seconds in a batch form. If not given neckbeard sends messages one by one. **Not yet implemented**
* `-redis-batch-size <num> - if `-redis-timer` is used, this is the maximum number of messages to send in a redis batch. Defaults to 64. **Not yet implemented**

* `-stdout` - This is default.  This sends json events to standard out.  This is useful for debugging or piping to another tool.

## Sampling ##

* `-sample <samplepercent>` - Only sample X percentage of the logs (can be percentage) [default: 100]

## Tags ##

You can add tags that will be added to the `@tags` field of the output.

* `-tag <tag>` - can be used multiple times

Additionally, if the field `tags` is an array in the json event, these will also be added to `@tags` before forwarding to logstash.

## Source ##

The logstash source can be set.

* `-source <source>` - sets `@source`. defaults to `neckbeard:<inputtype>:<inputarg>`

## Type ##

The logstash type can be set.

* `-type <type>` - sets `@type`. defaults to `neckbeard`


## Time ##

Logstash requires a specific timestamp format (iso8601).  Most programs log in their own format.  Neckbeard has options to transform into this format.

* `-time-field <field>` - field in the event to treat as the timestamp. If this option is not used, the fiedld is not found, or the value of the field is invalid, neckbeard will use the current time.
* `-time-layout "<layout>"` - layout of the timestamp in the event. This uses the standard go [time layout](http://golang.org/pkg/time/#pkg-constants).

## Stats ##

Neckbeard can provide statistics in json via HTTP.

* `-http-stats <port>` - listen on this port and provide statistics including buffer lengths, process health, events processed, latency, etc.

## Tuning ##
Various tuning options

* `-buffer-size <number>` - number of lines to keep in buffers. Note: this is per step, ie, this size buffer is used for each pipe, etc.

## Examples ##

`neckbeard -file /var/log/nginx/access.log -time-field request_field -time-layout "Mon, 2 Jan 2006 15:04:05 -0700 -t my_site -redis 127.0.0.1:6379` - this will read from `/var/log/nginx/access.log`, which is assumed to already be in json. The timestamp is in the field `request_field`. The tag `my_site` will be added to the output.

The event in the file will look like (presented on multiple lines for clarity)

    {
        "request_time": "01/May/2014:17:50:50 +0000",
        "remote_addr": "192.168.0.1",
        "remote_user": "-",
        "body_bytes_sent": "13988",
        "request_time": "0.122",
        "status": "200",
        "request": "GET /some/url HTTP/1.1",
        "request_method": "GET",
        "http_referrer": "http://www.example.org/some/url",
        "http_user_agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.1 (KHTML, like Gecko) Chrome/21.0.1180.79 Safari/537.1"
    }

(example from http://blog.pkhamre.com/2012/08/23/logging-to-logstash-json-format-in-nginx/ )

The transformed event sent to redis for logstash will look like (presented on multiple lines for clarity)

    {
        "@source": "neckbeard:file:/var/log/nginx/access.log",
        "@type": "neckbeard",
        "@tags": [ "my_site" ],
        "@fields": {
            "request_time": "01/May/2014:17:50:50 +0000",
            "remote_addr": "192.168.0.1",
            "remote_user": "-",
            "body_bytes_sent": "13988",
            "request_time": "0.122",
            "status": "200",
            "request": "GET /some/url HTTP/1.1",
            "request_method": "GET",
            "http_referrer": "http://www.example.org/some/url",
            "http_user_agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.1 (KHTML, like Gecko) Chrome/21.0.1180.79 Safari/537.1"
        },
        "@timestamp": "2012-08-23T10:49:14+02:00"
    }

Similar Projects
----------------
* [beaver](https://github.com/josegonzalez/beaver)
* [logstash forwarder](https://github.com/elasticsearch/logstash-forwarder)

Implementation
--------------
Neckbeard is written in go. 


LICENSE
-------
Copyright (c) 2014, Brian Akins, Wilson Wise
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:
    * Redistributions of source code must retain the above copyright
      notice, this list of conditions and the following disclaimer.
    * Redistributions in binary form must reproduce the above copyright
      notice, this list of conditions and the following disclaimer in the
      documentation and/or other materials provided with the distribution.
    * Neither the name of the <organization> nor the
      names of its contributors may be used to endorse or promote products
      derived from this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL <COPYRIGHT HOLDER> BE LIABLE FOR ANY
DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
(INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND
ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
