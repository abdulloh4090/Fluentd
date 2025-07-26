# Collecting Logs

:::note if you want to collect and route logs - use Fluentd agent
:::
## Overview

Fluentd is a fully free and fully open-source log collector that instantly enables you to have a Log Everything' architecture with [125+ types of systems.](https://www.fluentd.org/plugins)

You can visit [official site](https://www.fluentd.org/)


![alt text](../../static/img/fluentd-architecture.png)


Fluentd treats logs as JSON, a popular machine-readable format. It is written primarily in C with a thin-Ruby wrapper that gives users flexibility.


## Requirements 
To successfully install and run Fluentd on your system, the following software and hardware requirements must be met:

1. **CPU and RAM**
Minimum: 1 vCPU, 512MB RAM
Recommended: 2+ vCPUs, 2GB+ RAM (for handling large log volumes)
2. **Disk Space**
At least 1GB of free disk space (for logs and buffer files)
SSD is recommended for faster disk I/O performance

To use the td-agent (Fluentd’s stable production version), you can install it using the official installation script:

Step-by-step:

```
curl -fsSL https://toolbelt.treasuredata.com/sh/install-ubuntu-focal-td-agent4.sh | sh
```

Replace ```ubuntu-focal``` with your OS version (e.g., ```centos7```, ```debian-buster```, etc.)


After installation is complete, you can verify it with:

```
td-agent --version
```
### Tail plugin

>The ```in_tail``` Input plugin allows Fluentd to read events from the tail of text files. Its behavior is similar to the ```tail -F``` command. Each log is collected throughout ```<source> </source>```

### Eample Configuration

```
<source>
  @type tail
  path /var/log/json.log
  pos_file /fluentd/logs/json.pos
  tag json.log
  <parse>
    @type json
  </parse>
</source>
```

### How It Works

When Fluentd is first configured with ```in_tail```, it will start reading from the ***tail*** of that log, not the beginning. Once the log is rotated, Fluentd starts reading the new file from the beginning. It keeps track of the current inode number.

If ```td-agent``` restarts, it resumes reading from the last position before the restart. This position is recorded in the position file specified by the ```pos_file ```parameter.

### Reading JSON, Apache, Syslog Logs

To read logs from Json, Apache, Syslog etc. You need to follow below steps. In addition to this, these step will be similir to other sources. Once you learn these steps other steps will be easier.

1.  JSON / Apche / Syslog Logs Source:

```

<source>
  @type tail 
  path /var/log/json.log
  pos_file /fluentd/logs/json.pos
  tag json.log
  <parse>
    @type json
  </parse>
</source>

<source>
  @type tail
  path /var/log/apache.log
  pos_file /fluentd/logs/apache.pos
  tag apache.log
  <parse>
    @type apache2
  </parse>
</source>

<source>
  @type tail
  path /var/log/syslog.log
  pos_file /fluentd/logs/syslog.pos
  tag syslog.log
  <parse>
    @type syslog
  </parse>
</source>

<filter json.log>
  @type record_transformer
  enable_ruby true
  <record>
    Timestamp ${Time.at(time.to_i).utc.strftime('%Y-%m-%dT%H:%M:%SZ')}
    Level ${record["level"] || "Info"}
    Message ${record["message"]}
    Source ${record["source"]}
    SourceIp "#{Socket.ip_address_list.detect(&:ipv4_private?)&.ip_address}"
    LogPath /var/log/json
    tag json_logs
  </record>
</filter>

<filter apache.log>
  @type record_transformer 
  enable_ruby true
  <record>
    Timestamp ${Time.at(time.to_i).utc.strftime('%Y-%m-%dT%H:%M:%SZ')}
    Level ${record["level"] || "Info"}
    Message ${record["message"]}
    Source ${record["source"]}
    SourceIp "#{Socket.ip_address_list.detect(&:ipv4_private?)&.ip_address}"
    LogPath /var/log/apache
    tag apache_logs
  </record>
</filter>

<filter syslog.log>
  @type record_transformer
  enable_ruby true
  <record>
    Timestamp ${Time.at(time.to_i).utc.strftime('%Y-%m-%dT%H:%M:%SZ')}
    Level ${record["level"] || "Info"}
    Message ${record["message"]}
    Source ${record["source"]}
    SourceIp "#{Socket.ip_address_list.detect(&:ipv4_private?)&.ip_address}"
    LogPath /var/log/syslog
    tag syslog_logs
  </record>
</filter>

<match *.log>
  @type stdout
</match>
```
```@type``` tail Reads log entries from a file.

```path``` Location of the ```json.log``` file.

```pos_file``` Stores the position of the last read line to ensure Fluentd picks up from the same place after restart.

```tag``` Used to route events in the pipeline.

```parse``` in Fluentd is used to convert raw log lines into structured data using predefined or custom formats like ```json```, ```regexp```, or ```apache2```.

```enable_rubly``` allows you to write Ruby code inside ```<record>``` blocks. When set to ```true```, Ruby expressions like ```#{}``` are evaluated. ```<record> <record>``` block is used to **add custom fields** to each log event. These fields are **appended** to the output of the log pipeline (e.g., to file, stdout, Elasticsearch, etc.).

```Timestamp``` Converts Fluentd's event time to ISO 8601 UTC format.

```Level``` Defaults to ```"Inf"o"```if not specified in the log.

```Message```, ```Source``` Extracted directly from the log record.

```SourceIp``` Dynamically detects and embeds the host's internal IP address.

```LogPath``` Hardcoded to identify the source directory.

```tag``` Provides a secondary tag for downstream identification (not used for routing here). The same structure is applied to Apache and Syslog filters with minor differences in ```LogPath``` and tag.

```<match> ``` Captures all events with a tag ending in ```.log```.

```stdout``` Outputs the final log record to the standard output, useful for debugging or Docker logging.

### How to Run with Docker

You can run this Fluentd configuration in a Docker container using the following steps:

#### Project Structure:

```
Fluentd_Task/
├── fluent.conf
├── logs/
│   ├── json.log
│   ├── apache.log
│   └── syslog.log
```

#### Docker Run Command:

```
docker run -d \
  --name fluentd-log-aggregator \
  -v "$(pwd)/fluent.conf:/fluentd/etc/fluent.conf" \
  -v "$(pwd)/logs:/var/log" \
  -p 9880:9880 \
  fluent/fluentd:v1.16-
```

```-v``` mounts the local configuration and logs directory into the container.

- Logs written to ```logs/``` will be tailed by Fluentd in real-time.
- Use ```docker logs -f fluentd-log-aggregator``` to monitor the Fluentd output.

###  Log Samples

#### JSON Log (```json.log```):
```
{"level": "warn", "message": "User login failed", "source": "auth-service"}
```

#### Apache Log (```apache.log```):
```
127.0.0.1 - - [10/Jul/2025:09:00:00 +0000] "GET /index.html HTTP/1.1" 200 612
```

#### Syslog Log (```syslog.log```):
```
<34>Jul 10 09:02:30 server-name sshd[1234]: Failed password for invalid user
```

These logs will be parsed and transformed into structured events with unified fields like ```Timestamp```, ```Message```, ```Level```, etc.