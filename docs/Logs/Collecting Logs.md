# Fluentd Documentation

## Overview

Fluentd is an open-source data collector for unified logging layer. It allows you to collect logs from various sources, process them, and forward them to different destinations.

## Installation

### Using RubyGems

```sh
gem install fluentd
```

### Using Docker

```sh
docker run -it --rm fluent/fluentd:v1.16-1
```

## Basic Configuration

Create a configuration file `fluent.conf`:

```conf
<source>
  @type tail
  path /var/log/app.log
  pos_file /var/log/fluentd-app.pos
  tag app.log
  format none
</source>

<match app.log>
  @type stdout
</match>
```

## Running Fluentd

```sh
fluentd -c fluent.conf
```

## Plugins

Fluentd supports many plugins for input, output, and filtering. Install plugins using:

```sh
gem install fluent-plugin-<plugin-name>
```

## Useful Links

- [Official Documentation](https://docs.fluentd.org/)
- [Plugin List](https://www.fluentd.org/plugins)

## Example Use Cases

- Centralized log collection
- Log forwarding to Elasticsearch, S3, or other destinations
- Log filtering and transformation

## Troubleshooting

- Check logs for errors: `/var/log/fluentd.log`
- Validate configuration: `fluentd --dry-run -c fluent.conf`
