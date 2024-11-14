Here’s the documentation for `fluentbit-netdp` translated into English, emphasizing that `fluent.conf` is the global configuration file, while the actual configurations are located in the `conf.d` directory.

# Fluent Bit NetDP Documentation

## Directory Structure

```
fluentbit-netdp/
├── bin/
│   └── fluentbit              # Fluent Bit binary file
├── conf/
│   ├── fluent.conf            # Global configuration file
│   ├── fluent.env             # Environment variable configuration file
│   ├── parser.conf            # Log parser configuration file
│   └── conf.d/                # Directory for actual configuration files
│       └── ...                # Additional included configuration files
└── data/
    └── db/                    # Directory for database file storage
```

## Directory Content Overview

### bin Directory

- **fluentbit**: This is the Fluent Bit binary executable that is responsible for starting and running the log collector.

### conf Directory

- **fluent.conf**: The global configuration file primarily used to define the basic settings of the service, such as the flush interval and log level. This file typically includes actual configuration files from the `conf.d` directory.

- **fluent.env**: The environment variable configuration file used to set runtime environment variables, such as API keys and connection strings.

- **parser.conf**: The log parser configuration file that defines how to parse different formats of log data for further processing.

- **conf.d/**: This directory contains the actual configuration files that define specific input, filter, and output pipeline configurations. These files can be included in `fluent.conf` using the `@INCLUDE` directive for modular management.

### data Directory

- **db/**: This directory stores database files used to record the state of processed logs to avoid duplicate processing. Fluent Bit automatically manages the files in this directory.

## Configuration File Overview

### fluent.conf

This is the global configuration file, typically including:

- **[SERVICE]**: Service configuration, such as the flush interval and log level.

```ini
[SERVICE]
    Flush         5
    Log_Level     info
```

- **@INCLUDE**: Directive to include actual configuration files from the `conf.d` directory.

```ini
@INCLUDE conf.d/*.conf
```

### fluent.env

This file is used to set environment variables, typically including:

- `FLUENT_ELASTICSEARCH_HOST`: The host address for Elasticsearch.
- `FLUENT_ELASTICSEARCH_PORT`: The port for Elasticsearch.
- Other environment variables related to the service.

### parser.conf

This file is used to define parsers, specifying parsing rules for different formats such as JSON, CSV, etc. Example content:

```ini
[PARSER]
    Name   json
    Format json
```

### conf.d/ Directory

You can place multiple configuration files in this directory that define specific input, filter, and output configurations, supporting modular management. For example:

```ini
[INPUT]
    Name   tail
    Path   /var/log/myapp/*.log
    Parser json
    Tag    myapp_logs

[OUTPUT]
    Name   forward
    Match  myapp_logs
    Host   127.0.0.1
    Port   24224
```

## Starting and Running

To start Fluent Bit, you can run the following command from the `bin` directory:

```bash
./fluentbit -c ../conf/fluent.conf
```

Make sure all configuration files are correctly set up before running.

## Frequently Asked Questions

1. **How do I add a new input source in `conf.d`?**
   - Create a new configuration file in the `conf.d` directory and define a new `[INPUT]` block.

2. **How do I modify the log output target?**
   - Find the corresponding output configuration file in `conf.d` and update the `[OUTPUT]` block.

3. **How can I view Fluent Bit's logs?**
   - Logs from Fluent Bit are typically displayed in the console, and you can also configure output to files or other log management systems.

## References

- [Fluent Bit Official Documentation]
- [Fluent Bit GitHub Page]
