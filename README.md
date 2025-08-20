# MK Livestatus 1.5

MK Livestatus is a high-performance interface to monitoring data that provides real-time access to Nagios and Icinga monitoring information through a fast and efficient protocol. It was created by Mathias Kettner as part of the Check_MK monitoring solution.

## Overview

Livestatus acts as a broker module for Nagios/Icinga that opens a socket (Unix or TCP) which can be used to query the current status of all monitored objects. Unlike traditional monitoring interfaces that rely on flat files or CGI scripts, Livestatus provides:

- **Real-time data access** - No need to parse flat files or wait for updates
- **High performance** - Optimized C++ implementation with minimal overhead
- **Flexible query language** - SQL-like syntax for filtering and aggregating data
- **Multiple output formats** - CSV, JSON, Python-compatible formats
- **Multi-site support** - Query multiple monitoring instances simultaneously
- **Keep-alive connections** - Efficient connection reuse and pooling

## Key Features

### Core Capabilities
- **Live monitoring data access** - Real-time status information without file I/O
- **Efficient querying** - Filter, sort, and aggregate data server-side
- **Multi-threaded architecture** - Handle multiple concurrent connections
- **Zero configuration** - Works out-of-the-box with standard Nagios/Icinga setups
- **Minimal resource usage** - Low CPU and memory footprint

### Supported Data Tables
- **hosts** - Host status, configuration, and performance data
- **services** - Service status, configuration, and performance data
- **hostgroups/servicegroups** - Group memberships and statistics
- **contacts/contactgroups** - Contact information and notifications
- **commands** - Available check commands and their definitions
- **timeperiods** - Time period definitions and current states
- **downtimes** - Scheduled maintenance windows
- **comments** - User comments on hosts and services
- **log** - Historical log entries and events
- **status** - General Nagios/Icinga process information

### Output Formats
- **CSV** - Comma-separated values for easy parsing
- **JSON** - JavaScript Object Notation for web applications
- **Python** - Native Python data structures (lists and dictionaries)

### Language Bindings
- **Python** - Full-featured API with connection pooling and multi-site support
- **Perl** - Monitoring::Livestatus module for Perl applications
- **C++** - Native C++ client library for high-performance applications

## Quick Start

### Prerequisites
- Nagios >= 3.0 or Icinga >= 1.0
- C++17-compatible compiler (GCC 7+ or Clang 5.0+)
- Boost libraries (ASIO required)
- RRD library (librrd or librrd_th)

### Installation

1. **Download and extract** the source code:
   ```bash
   git clone https://github.com/xoner/mk-livestatus-1.5.git
   cd mk-livestatus-1.5
   ```

2. **Configure the build**:
   ```bash
   ./configure
   ```
   
   For Nagios 4, use:
   ```bash
   ./configure --with-nagios4
   ```

3. **Compile and install**:
   ```bash
   make
   sudo make install
   ```

4. **Configure Nagios/Icinga** to load the Livestatus module by adding to your main configuration file:
   ```
   # For standard Unix socket (recommended)
   broker_module=/usr/local/lib/mk-livestatus/livestatus.o /tmp/livestatus
   
   # For TCP socket (use with caution)
   broker_module=/usr/local/lib/mk-livestatus/livestatus.o inet:6557
   ```

5. **Restart Nagios/Icinga** to load the module.

### Basic Usage

#### Command Line (using unixcat)
```bash
# Get all hosts
echo "GET hosts" | /usr/local/bin/unixcat /tmp/livestatus

# Get hosts with specific columns
echo "GET hosts
Columns: name state plugin_output" | unixcat /tmp/livestatus

# Filter for down hosts
echo "GET hosts
Filter: state = 1" | unixcat /tmp/livestatus
```

#### Python API
```python
import livestatus

# Connect to livestatus socket
conn = livestatus.SingleSiteConnection("unix:/tmp/livestatus")

# Get all hosts
hosts = conn.query_table("GET hosts\nColumns: name state address")
for name, state, address in hosts:
    print(f"{name}: {state} ({address})")

# Get performance data
status = conn.query_row_assoc("GET status")
print(f"Livestatus version: {status['livestatus_version']}")

# Get service statistics
stats = conn.query_row(
    "GET services\n"
    "Stats: state = 0\n"
    "Stats: state = 1\n" 
    "Stats: state = 2\n"
    "Stats: state = 3"
)
ok, warning, critical, unknown = stats
print(f"Services: {ok} OK, {warning} Warning, {critical} Critical, {unknown} Unknown")
```

#### Perl API
```perl
use Monitoring::Livestatus;

my $ml = Monitoring::Livestatus->new(
    socket => '/tmp/livestatus'
);

# Get all hosts
my $hosts = $ml->selectall_arrayref(
    "GET hosts\nColumns: name state address"
);

for my $host (@$hosts) {
    my ($name, $state, $address) = @$host;
    print "$name: $state ($address)\n";
}
```

## Query Language

Livestatus uses a simple but powerful query language based on the Livestatus Query Language (LQL):

### Basic Query Structure
```
GET <table>
[Columns: <column1> <column2> ...]
[Filter: <column> <operator> <value>]
[Stats: <aggregation>]
[OutputFormat: <format>]
```

### Examples

#### Basic Queries
```bash
# Get all services with their current state
GET services
Columns: host_name description state plugin_output

# Get only critical services
GET services
Filter: state = 2
Columns: host_name description plugin_output

# Count services by state
GET services
Stats: state = 0
Stats: state = 1
Stats: state = 2
Stats: state = 3
```

#### Advanced Filtering
```bash
# Multiple filters (AND logic)
GET hosts
Filter: state = 1
Filter: scheduled_downtime_depth = 0
Columns: name state plugin_output

# OR logic using Or: directive
GET services
Filter: state = 1
Filter: state = 2
Or: 2
Columns: host_name description state
```

#### Aggregation and Statistics
```bash
# Group services by host
GET services
Columns: host_name
Stats: state = 0
Stats: state = 1
Stats: state = 2
StatsGroupBy: host_name
```

## Architecture

### Component Overview
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Client App    │◄──►│   Livestatus     │◄──►│ Nagios/Icinga   │
│                 │    │     Module       │    │    Process      │
│ - Python API    │    │                  │    │                 │
│ - Perl API      │    │ - Query Parser   │    │ - Host/Service  │
│ - C++ API       │    │ - Data Access    │    │   States        │
│ - unixcat       │    │ - Socket Server  │    │ - Configuration │
└─────────────────┘    │ - Multi-threading│    │ - Log Entries   │
                       └──────────────────┘    └─────────────────┘
```

### Key Components

#### Core Module (`src/module.cc`)
- **Nagios Event Broker Module** - Integrates with Nagios/Icinga core
- **Socket Management** - Handles Unix and TCP socket connections
- **Thread Management** - Manages client connection threads
- **Logging** - Comprehensive logging with configurable levels

#### Query Engine
- **Query Parser** - Parses LQL queries into internal representation
- **Table System** - Modular table implementation for different data types
- **Filter Engine** - Efficient filtering with multiple operators
- **Aggregation Engine** - Statistical functions and grouping

#### Data Access Layer
- **Live Data Access** - Direct access to Nagios/Icinga internal structures
- **Log File Access** - Historical data from Nagios log files
- **Performance Data** - Integration with RRD for performance metrics
- **External Data** - Support for external data sources (Event Console, etc.)

## Configuration

### Module Arguments
The Livestatus module accepts several configuration parameters:

```bash
# Basic Unix socket
broker_module=/path/to/livestatus.o /tmp/livestatus

# TCP socket with custom options
broker_module=/path/to/livestatus.o inet:6557 debug=1 max_cached_messages=1000

# Multiple options
broker_module=/path/to/livestatus.o /tmp/livestatus debug=2 max_lines_per_logfile=100000
```

### Available Options
- **Socket path/port** - Unix socket path or `inet:PORT` for TCP
- **debug=LEVEL** - Debug level (0=off, 1=info, 2=debug)
- **max_cached_messages=NUM** - Maximum log messages to cache
- **max_lines_per_logfile=NUM** - Maximum lines to read from log files
- **thread_stack_size=BYTES** - Stack size for client threads

### Performance Tuning
- **Connection pooling** - Reuse connections when possible
- **Efficient filtering** - Apply filters server-side to reduce data transfer
- **Column selection** - Only request needed columns
- **Appropriate timeouts** - Set reasonable query timeouts

## API Reference

### Python API

The Python API provides both single-site and multi-site connection classes:

#### SingleSiteConnection
```python
import livestatus

conn = livestatus.SingleSiteConnection("unix:/tmp/livestatus")

# Query methods
hosts = conn.query_table(query)           # Returns list of lists
host_dict = conn.query_row_assoc(query)   # Returns dictionary
count = conn.query_value(query)           # Returns single value
column = conn.query_column(query)         # Returns list of values
```

#### MultiSiteConnection
```python
sites = {
    "site1": {"socket": "unix:/omd/site1/tmp/run/live"},
    "site2": {"socket": "unix:/omd/site2/tmp/run/live"}
}

conn = livestatus.MultiSiteConnection(sites)
# Same query methods as SingleSiteConnection
```

### Perl API

```perl
use Monitoring::Livestatus;

my $ml = Monitoring::Livestatus->new(
    socket => '/tmp/livestatus',
    timeout => 30
);

# Query methods
my $data = $ml->selectall_arrayref($query);
my $data = $ml->selectall_hashref($query);
my $row = $ml->selectrow_arrayref($query);
my $row = $ml->selectrow_hashref($query);
```

### C++ API

```cpp
#include "Livestatus.h"

Livestatus live;
live.connectUNIX("/tmp/livestatus");

live.sendQuery("GET hosts\nColumns: name state");
std::vector<std::string> *row;
while ((row = live.nextRow()) != nullptr) {
    // Process row data
    delete row;
}
```

## Examples

For more detailed examples, see the following files:
- `api/python/example.py` - Basic Python usage
- `api/python/example_multisite.py` - Multi-site Python usage
- `api/c++/demo.cc` - C++ example
- See `EXAMPLES.md` for additional use cases

## Building from Source

### Dependencies
- **Build tools**: autoconf, automake, make, C++17 compiler
- **Libraries**: Boost (ASIO), RRD library
- **Optional**: RE2 library for enhanced regex support

### Configure Options
```bash
./configure --help

# Common options:
--with-nagios4              # Build for Nagios 4
--with-re2[=PATH]          # Enable RE2 regex library
--disable-rrd-is-thread-safe # Use librrd_th instead of librrd
```

### Development Build
```bash
git clone https://github.com/xoner/mk-livestatus-1.5.git
cd mk-livestatus-1.5
./configure --with-nagios4
make
make check  # Run tests (if available)
```

## Troubleshooting

### Common Issues

1. **Socket permission errors**
   - Ensure the web server/application user can access the socket
   - Check socket path and permissions

2. **Connection timeouts**
   - Verify Nagios/Icinga is running
   - Check that the module is loaded correctly
   - Review Nagios/Icinga logs for errors

3. **Performance issues**
   - Use column selection to reduce data transfer
   - Apply filters server-side rather than client-side
   - Consider connection pooling for high-frequency queries

4. **Build failures**
   - Ensure all dependencies are installed
   - Check C++17 compiler support
   - Verify Boost library versions

### Log Analysis
Check Nagios/Icinga logs for Livestatus messages:
```bash
# Look for Livestatus initialization
grep "livestatus" /var/log/nagios/nagios.log

# Check for socket creation
grep "socket" /var/log/nagios/nagios.log
```

## Contributing

We welcome contributions to MK Livestatus! Please see our contribution guidelines:

### Development Environment
1. Fork the repository
2. Set up development environment with required dependencies
3. Make changes with appropriate tests
4. Submit pull request with clear description

### Coding Standards
- Follow existing C++ style conventions
- Add appropriate documentation for new features
- Include unit tests for new functionality
- Update API documentation as needed

### Reporting Issues
- Use GitHub Issues for bug reports and feature requests
- Provide detailed reproduction steps
- Include relevant log files and configuration
- Specify Nagios/Icinga version and operating system

## License

MK Livestatus is licensed under the GNU General Public License version 2 (GPL-2.0).

```
Copyright (C) 2014 Mathias Kettner

This program is free software; you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation; either version 2 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
GNU General Public License for more details.
```

## Change History

For detailed change history, see:
- `api/perl/Changes` - Perl API change history
- Git commit history for core changes

## Links and Resources

- **Official Check_MK Website**: http://mathias-kettner.de/check_mk.html
- **OMD (Open Monitoring Distribution)**: http://omdistro.org
- **Nagios**: https://www.nagios.org
- **Icinga**: https://icinga.com
- **GitHub Repository**: https://github.com/xoner/mk-livestatus-1.5

## Support

For support and questions:
- GitHub Issues for bug reports and feature requests
- Check_MK community forums for general usage questions
- Professional support available through Check_MK commercial offerings

---

*Created by Mathias Kettner (mk@mathias-kettner.de)*  
*Part of the Check_MK monitoring solution*