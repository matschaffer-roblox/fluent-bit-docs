# Syslog

_Syslog_ input plugins allows to collect Syslog messages through a Unix socket server (UDP or TCP) or over the network using TCP mode.

Content:

- [Configuration Parameters](#config)
- [Getting Started](#getting_started)
- [Recipes](#recipes)
 - [Rsyslog to Fluent Bit / Network TCP](#rsyslog_to_fluentbit_network)
 - [Rsyslog to Fluent Bit / Unix Socket UDP](#rsyslog_to_fluentbit_unix_udp)

## Configuration Parameters {#config}

The plugin supports the following configuration parameters:

| Key         | Description       | Default |
| ------------|-------------------|---------|
| Mode        | Defines transport protocol mode: unix\_udp (UDP over Unix socket), unix\_tcp (TCP over Unix socket) or common tcp | unix_udp |
| Listen      | If _Mode_ is set to _tcp_, specify the network interface to bind. | 0.0.0.0 |
| Port        | If _Mode_ is set to _tcp_, specify the TCP port to listen for incoming connections. | 5140 |
| Path        | If _Mode_ is set to _unix\_tcp_ or _unix\_udp_, set the absolute path to the Unix socket file. | |
| Parser      | Specify an alternative parser for the message. By default, the plugin uses the parser _syslog-rfc3164_. If your syslog messages have fractional seconds set this Parser value to _syslog-rfc5424_ instead. | |
| Buffer\_Size| Specify the maximum buffer size in KB to receive a Syslog message. If not set, the default size will be the value of _Chunk\_Size_. |
| Chunk\_Size  | By default the buffer to store the incoming Syslog messages, do not allocate the maximum memory allowed, instead it allocate memory when is required. The rounds of allocations are set by _Chunk\_Size_ in KB. If not set, _Chunk\_Size_ is equal to 32 (32KB). | |

Note that Fluent Bit requires access to the _parsers.conf_ file, the path to this file can be specified with the option _-R_ or through the _Parsers\_File_ key on the [SERVER] section (more details below).

## Getting Started {#getting_started}

In order to receive Syslog messages, you can run the plugin from the command line or through the configuration file:

### Command Line

From the command line you can let Fluent Bit listen for _Forward_ messages with the following options:

```bash
$ fluent-bit -R /path/to/parsers.conf -i syslog -p path=/tmp/in_syslog -o stdout
```

By default the service will create and listen for Syslog messages on the unix socket _/tmp/in\_syslog_

### Configuration File

In your main configuration file append the following _Input_ & _Output_ sections:

```python
[SERVICE]
    Flush        1
    Log_Level    info
    Parsers_File parsers.conf

[INPUT]
    Name         syslog
    Path         /tmp/in_syslog
    Chunk_Size   32
    Buffer_Size  64

[OUTPUT]
    Name   stdout
    Match  *
```

### Testing

Once Fluent Bit is running, you can send some messages using the _logger_ tool:

```bash
$ logger -u /tmp/in_syslog my_ident my_message
```

In [Fluent Bit](http://fluentbit.io) we should see the following output:

```bash
$ bin/fluent-bit -R ../conf/parsers.conf -i syslog -p path=/tmp/in_syslog -o stdout
Fluent-Bit v0.11.0
Copyright (C) Treasure Data

[2017/03/09 02:23:27] [ info] [engine] started
[0] syslog.0: [1489047822, {"pri"=>"13", "host"=>"edsiper:", "ident"=>"my_ident", "pid"=>"", "message"=>"my_message"}]
```

## Recipes {#recipes}

The following content aims to provide configuration examples for different use cases to integrate Fluent Bit and make it listen for Syslog messages from your systems.

### Rsyslog to Fluent Bit: Network mode over TCP {#rsyslog_to_fluentbit_network}

##### Fluent Bit Configuration

Put the following content in your fluent-bit.conf file:

```
[SERVICE]
    Flush        1
    Parsers_File parsers.conf

[INPUT]
    Name     syslog
    Parser   syslog-rfc3164
    Listen   0.0.0.0
    Port     5140
    Mode     tcp

[OUTPUT]
    Name     stdout
    Match    *
```

then start Fluent Bit.

##### RSyslog Configuration

Add a new file to your rsyslog config rules called _60-fluent-bit.conf_ inside the directory _/etc/rsyslog.d/_ and add the following content:

```
action(type="omfwd" Target="127.0.0.1" Port="5140" Protocol="tcp")
```

then make sure to restart your rsyslog daemon:

```bash
$ sudo service rsyslog restart
```

### Rsyslog to Fluent Bit: Unix socket mode over UDP {#rsyslog_to_fluentbit_unix_udp}

##### Fluent Bit Configuration

Put the following content in your fluent-bit.conf file:

```
[SERVICE]
    Flush        1
    Parsers_File parsers.conf

[INPUT]
    Name     syslog
    Parser   syslog-rfc3164
    Path     /tmp/fluent-bit.sock
    Mode     unix_udp

[OUTPUT]
    Name     stdout
    Match    *
```

then start Fluent Bit.

##### RSyslog Configuration

Add a new file to your rsyslog config rules called _60-fluent-bit.conf_ inside the directory _/etc/rsyslog.d/_ and place the following content:

```
$ModLoad omuxsock
$OMUxSockSocket /tmp/fluent-bit.sock
*.* :omuxsock:
```

then make sure to set proper permissions to the socket and restart your rsyslog daemon:

```bash
$ sudo chmod 666 /tmp/fluent-bit.sock
$ sudo service rsyslog restart
```