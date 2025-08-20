# API Reference

This document provides comprehensive reference documentation for all MK Livestatus APIs and query language features.

## Table of Contents

1. [Query Language Reference](#query-language-reference)
2. [Data Tables](#data-tables)
3. [Python API Reference](#python-api-reference)
4. [Perl API Reference](#perl-api-reference)
5. [C++ API Reference](#cpp-api-reference)
6. [Output Formats](#output-formats)
7. [Error Handling](#error-handling)
8. [Performance Considerations](#performance-considerations)

## Query Language Reference

The Livestatus Query Language (LQL) is the foundation for all data access in Livestatus.

### Basic Syntax

```
GET <table>
[Columns: <column1> <column2> ...]
[Filter: <column> <operator> <value>]
[And: <number>]
[Or: <number>]
[Negate:]
[Stats: <aggregation>]
[StatsGroupBy: <column>]
[Limit: <number>]
[OutputFormat: <format>]
[ColumnHeaders: on|off]
[Separators: <record> <field> <list> <host_service>]
[ResponseHeader: fixed16]
[KeepAlive: on|off]
[AuthUser: <username>]
[Localtime: <timestamp>]
```

### Commands

#### GET
Retrieves data from a specific table.
```
GET hosts
GET services
GET log
```

#### Columns
Specifies which columns to return. If omitted, all columns are returned.
```
Columns: name state address
Columns: host_name description state plugin_output
```

#### Filter
Applies filtering conditions to limit results.
```
Filter: state = 1
Filter: name ~ web.*
Filter: last_check > 1609459200
```

### Filter Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `=` | Equals | `Filter: state = 0` |
| `!=` or `<>` | Not equals | `Filter: state != 0` |
| `~` | Regular expression match | `Filter: name ~ ^web.*` |
| `!~` | Regular expression not match | `Filter: name !~ test.*` |
| `=~` | Case-insensitive regex | `Filter: plugin_output =~ error` |
| `!=~` | Case-insensitive regex not match | `Filter: plugin_output !=~ ok` |
| `<` | Less than | `Filter: last_check < 1609459200` |
| `<=` | Less than or equal | `Filter: current_attempt <= 1` |
| `>` | Greater than | `Filter: execution_time > 5.0` |
| `>=` | Greater than or equal | `Filter: state >= 1` |

### Logical Operators

#### And
Combines the specified number of preceding filters with AND logic.
```
Filter: state = 1
Filter: acknowledged = 0
And: 2
```

#### Or
Combines the specified number of preceding filters with OR logic.
```
Filter: state = 1
Filter: state = 2
Or: 2
```

#### Negate
Negates the following filter.
```
Negate:
Filter: state = 0
```

### Statistics and Aggregation

#### Stats
Performs aggregation operations.
```
Stats: sum execution_time
Stats: avg execution_time
Stats: min last_check
Stats: max last_check
Stats: std execution_time
Stats: suminv execution_time
Stats: avginv execution_time
```

#### State Counting
```
Stats: state = 0
Stats: state = 1
Stats: state = 2
Stats: state = 3
```

#### StatsGroupBy
Groups statistics by specified columns.
```
Stats: state = 0
Stats: state = 1
StatsGroupBy: host_name
```

### Output Control

#### Limit
Limits the number of returned rows.
```
Limit: 100
```

#### OutputFormat
Specifies the output format.
```
OutputFormat: csv
OutputFormat: json
OutputFormat: python
```

#### ColumnHeaders
Controls whether column headers are included.
```
ColumnHeaders: on
ColumnHeaders: off
```

#### Separators
Defines custom separators for CSV output.
```
Separators: 10 59 44 124
# 10=\n, 59=;, 44=,, 124=|
```

## Data Tables

### hosts

Contains information about monitored hosts.

#### Key Columns
| Column | Type | Description |
|--------|------|-------------|
| `name` | string | Host name |
| `address` | string | IP address |
| `alias` | string | Host alias |
| `state` | int | Current state (0=UP, 1=DOWN, 2=UNREACHABLE) |
| `hard_state` | int | Hard state |
| `plugin_output` | string | Output from host check |
| `perf_data` | string | Performance data |
| `last_check` | time | Last check timestamp |
| `next_check` | time | Next scheduled check |
| `acknowledged` | int | Acknowledgment status |
| `scheduled_downtime_depth` | int | Downtime depth |
| `notifications_enabled` | int | Notifications enabled flag |
| `active_checks_enabled` | int | Active checks enabled flag |
| `accept_passive_checks` | int | Passive checks accepted flag |
| `execution_time` | float | Check execution time |
| `latency` | float | Check latency |
| `percent_state_change` | float | State change percentage |
| `is_flapping` | int | Flapping status |
| `total_services` | int | Total number of services |
| `services_ok` | int | Number of OK services |
| `services_warn` | int | Number of WARNING services |
| `services_crit` | int | Number of CRITICAL services |
| `services_unknown` | int | Number of UNKNOWN services |

#### Example Queries
```bash
# Get all UP hosts
GET hosts
Filter: state = 0
Columns: name address alias

# Get hosts with problems
GET hosts
Filter: state > 0
Filter: scheduled_downtime_depth = 0
Columns: name state plugin_output

# Host statistics by state
GET hosts
Stats: state = 0
Stats: state = 1
Stats: state = 2
```

### services

Contains information about monitored services.

#### Key Columns
| Column | Type | Description |
|--------|------|-------------|
| `host_name` | string | Name of the host |
| `description` | string | Service description |
| `display_name` | string | Service display name |
| `state` | int | Current state (0=OK, 1=WARN, 2=CRIT, 3=UNKNOWN) |
| `hard_state` | int | Hard state |
| `plugin_output` | string | Output from service check |
| `long_plugin_output` | string | Long output from service check |
| `perf_data` | string | Performance data |
| `check_command` | string | Check command |
| `last_check` | time | Last check timestamp |
| `next_check` | time | Next scheduled check |
| `last_state_change` | time | Last state change |
| `last_hard_state_change` | time | Last hard state change |
| `current_attempt` | int | Current check attempt |
| `max_check_attempts` | int | Maximum check attempts |
| `acknowledged` | int | Acknowledgment status |
| `scheduled_downtime_depth` | int | Downtime depth |
| `notifications_enabled` | int | Notifications enabled flag |
| `active_checks_enabled` | int | Active checks enabled flag |
| `accept_passive_checks` | int | Passive checks accepted flag |
| `execution_time` | float | Check execution time |
| `latency` | float | Check latency |
| `percent_state_change` | float | State change percentage |
| `is_flapping` | int | Flapping status |

#### Example Queries
```bash
# Get all CRITICAL services
GET services
Filter: state = 2
Columns: host_name description plugin_output

# Get services with problems not in downtime
GET services
Filter: state > 0
Filter: scheduled_downtime_depth = 0
Filter: acknowledged = 0
Columns: host_name description state plugin_output

# Service statistics by state
GET services
Stats: state = 0
Stats: state = 1
Stats: state = 2
Stats: state = 3

# Services grouped by host
GET services
Stats: state = 0
Stats: state = 1
Stats: state = 2
Stats: state = 3
StatsGroupBy: host_name
```

### log

Contains historical log entries.

#### Key Columns
| Column | Type | Description |
|--------|------|-------------|
| `time` | time | Log entry timestamp |
| `lineno` | int | Line number in log file |
| `class` | int | Log entry class |
| `type` | string | Log entry type |
| `state` | int | State information |
| `state_type` | string | State type (HARD/SOFT) |
| `host_name` | string | Host name |
| `service_description` | string | Service description |
| `plugin_output` | string | Plugin output |
| `message` | string | Complete log message |
| `options` | string | Additional options |
| `comment` | string | Comment text |
| `contact_name` | string | Contact name |
| `command_name` | string | Command name |

#### Log Classes
- `0` = Info
- `1` = State alerts (host/service state changes)
- `2` = Program messages
- `3` = Notifications
- `4` = Passive checks
- `5` = External commands
- `6` = Host/service flapping

#### Example Queries
```bash
# Get recent alerts
GET log
Filter: time > 1609459200
Filter: class = 1
Columns: time host_name service_description message
Limit: 50

# Get notifications in the last hour
GET log
Filter: time > 1609459200
Filter: class = 3
Columns: time contact_name host_name service_description

# Count log entries by class
GET log
Stats: class = 0
Stats: class = 1
Stats: class = 2
Stats: class = 3
```

### status

Contains general Nagios/Icinga process information.

#### Key Columns
| Column | Type | Description |
|--------|------|-------------|
| `livestatus_version` | string | Livestatus version |
| `nagios_version` | string | Nagios/Icinga version |
| `program_start` | time | Program start time |
| `program_version` | string | Program version |
| `last_command_check` | time | Last command check |
| `last_log_rotation` | time | Last log rotation |
| `enable_notifications` | int | Global notifications enabled |
| `execute_service_checks` | int | Service checks enabled |
| `execute_host_checks` | int | Host checks enabled |
| `enable_event_handlers` | int | Event handlers enabled |
| `enable_flap_detection` | int | Flap detection enabled |
| `process_performance_data` | int | Performance data processing |
| `num_hosts` | int | Total number of hosts |
| `num_services` | int | Total number of services |

### hostgroups

Contains host group information.

#### Key Columns
| Column | Type | Description |
|--------|------|-------------|
| `name` | string | Host group name |
| `alias` | string | Host group alias |
| `members` | list | List of member hosts |
| `num_hosts` | int | Number of hosts |
| `num_hosts_up` | int | Number of UP hosts |
| `num_hosts_down` | int | Number of DOWN hosts |
| `num_hosts_unreach` | int | Number of UNREACHABLE hosts |
| `num_services` | int | Total services in group |
| `num_services_ok` | int | Number of OK services |
| `num_services_warn` | int | Number of WARNING services |
| `num_services_crit` | int | Number of CRITICAL services |
| `num_services_unknown` | int | Number of UNKNOWN services |
| `worst_host_state` | int | Worst host state in group |
| `worst_service_state` | int | Worst service state in group |

### servicegroups

Contains service group information.

#### Key Columns
| Column | Type | Description |
|--------|------|-------------|
| `name` | string | Service group name |
| `alias` | string | Service group alias |
| `members` | list | List of member services |
| `num_services` | int | Number of services |
| `num_services_ok` | int | Number of OK services |
| `num_services_warn` | int | Number of WARNING services |
| `num_services_crit` | int | Number of CRITICAL services |
| `num_services_unknown` | int | Number of UNKNOWN services |
| `worst_service_state` | int | Worst service state in group |

### contacts

Contains contact information.

#### Key Columns
| Column | Type | Description |
|--------|------|-------------|
| `name` | string | Contact name |
| `alias` | string | Contact alias |
| `email` | string | Email address |
| `pager` | string | Pager number |
| `host_notification_period` | string | Host notification period |
| `service_notification_period` | string | Service notification period |
| `host_notifications_enabled` | int | Host notifications enabled |
| `service_notifications_enabled` | int | Service notifications enabled |
| `can_submit_commands` | int | Can submit commands |

### contactgroups

Contains contact group information.

#### Key Columns
| Column | Type | Description |
|--------|------|-------------|
| `name` | string | Contact group name |
| `alias` | string | Contact group alias |
| `members` | list | List of member contacts |

### downtimes

Contains scheduled downtime information.

#### Key Columns
| Column | Type | Description |
|--------|------|-------------|
| `id` | int | Downtime ID |
| `author` | string | Downtime author |
| `comment` | string | Downtime comment |
| `start_time` | time | Downtime start time |
| `end_time` | time | Downtime end time |
| `fixed` | int | Fixed downtime flag |
| `duration` | int | Downtime duration |
| `host_name` | string | Host name |
| `service_description` | string | Service description |
| `is_service` | int | Service downtime flag |

### comments

Contains comment information.

#### Key Columns
| Column | Type | Description |
|--------|------|-------------|
| `id` | int | Comment ID |
| `author` | string | Comment author |
| `comment` | string | Comment text |
| `entry_time` | time | Entry timestamp |
| `type` | int | Comment type |
| `is_service` | int | Service comment flag |
| `persistent` | int | Persistent comment flag |
| `source` | int | Comment source |
| `expires` | int | Comment expires flag |
| `expire_time` | time | Expiration time |
| `host_name` | string | Host name |
| `service_description` | string | Service description |

### commands

Contains command definitions.

#### Key Columns
| Column | Type | Description |
|--------|------|-------------|
| `name` | string | Command name |
| `line` | string | Command line |

### timeperiods

Contains time period definitions.

#### Key Columns
| Column | Type | Description |
|--------|------|-------------|
| `name` | string | Time period name |
| `alias` | string | Time period alias |
| `in` | int | Currently in time period |
| `monday` | string | Monday time ranges |
| `tuesday` | string | Tuesday time ranges |
| `wednesday` | string | Wednesday time ranges |
| `thursday` | string | Thursday time ranges |
| `friday` | string | Friday time ranges |
| `saturday` | string | Saturday time ranges |
| `sunday` | string | Sunday time ranges |

## Python API Reference

### Module: livestatus

#### SingleSiteConnection

```python
class SingleSiteConnection:
    def __init__(self, socket_uri, timeout=None)
```

**Parameters:**
- `socket_uri` (str): Socket URI (e.g., "unix:/tmp/livestatus" or "tcp:localhost:6557")
- `timeout` (int, optional): Connection timeout in seconds

**Methods:**

##### query_table(query)
Execute query and return list of lists.
```python
hosts = conn.query_table("GET hosts\nColumns: name state address")
# Returns: [['host1', 0, '192.168.1.1'], ['host2', 1, '192.168.1.2']]
```

##### query_table_assoc(query)
Execute query and return list of dictionaries.
```python
hosts = conn.query_table_assoc("GET hosts\nColumns: name state address")
# Returns: [{'name': 'host1', 'state': 0, 'address': '192.168.1.1'}, ...]
```

##### query_row(query)
Execute query and return single row as list.
```python
stats = conn.query_row("GET hosts\nStats: state = 0\nStats: state = 1")
# Returns: [10, 2]
```

##### query_row_assoc(query)
Execute query and return single row as dictionary.
```python
status = conn.query_row_assoc("GET status")
# Returns: {'livestatus_version': '1.5.0', 'nagios_version': '4.4.5', ...}
```

##### query_value(query)
Execute query and return single value.
```python
count = conn.query_value("GET hosts\nStats: state = 0")
# Returns: 10
```

##### query_column(query)
Execute query and return single column as list.
```python
names = conn.query_column("GET hosts\nColumns: name")
# Returns: ['host1', 'host2', 'host3']
```

##### query_summed_stats(query)
Execute query on multiple sites and sum statistics.
```python
stats = conn.query_summed_stats("GET hosts\nStats: state = 0\nStats: state = 1")
# Returns: [25, 5]  # Summed across all sites
```

#### MultiSiteConnection

```python
class MultiSiteConnection:
    def __init__(self, sites, timeout=None)
```

**Parameters:**
- `sites` (dict): Dictionary of site configurations
- `timeout` (int, optional): Connection timeout in seconds

**Site Configuration:**
```python
sites = {
    "site1": {
        "socket": "unix:/omd/site1/tmp/run/live",
        "alias": "Site 1"
    },
    "site2": {
        "socket": "tcp:monitoring2:6557",
        "alias": "Site 2"
    }
}
```

**Methods:**
Same as SingleSiteConnection, but queries are executed across all configured sites.

#### Exceptions

##### MKLivestatusException
Base exception for all Livestatus errors.
```python
try:
    result = conn.query_value("GET hosts\nColumns: invalid_column")
except livestatus.MKLivestatusException as e:
    print(f"Livestatus error: {e}")
```

##### MKLivestatusSocketError
Socket connection errors.
```python
try:
    conn = livestatus.SingleSiteConnection("unix:/invalid/socket")
except livestatus.MKLivestatusSocketError as e:
    print(f"Socket error: {e}")
```

##### MKLivestatusConfigError
Configuration errors.

### Example Usage

```python
import livestatus

# Basic connection
conn = livestatus.SingleSiteConnection("unix:/tmp/livestatus")

# Error handling
try:
    hosts = conn.query_table_assoc("GET hosts\nColumns: name state address")
    for host in hosts:
        print(f"{host['name']}: {host['state']}")
except livestatus.MKLivestatusException as e:
    print(f"Error: {e}")

# Multi-site setup
sites = {
    "prod": {"socket": "unix:/omd/prod/tmp/run/live"},
    "test": {"socket": "unix:/omd/test/tmp/run/live"}
}
multi_conn = livestatus.MultiSiteConnection(sites)
all_hosts = multi_conn.query_table_assoc("GET hosts\nColumns: name state")
```

## Perl API Reference

### Module: Monitoring::Livestatus

#### Constructor

```perl
my $ml = Monitoring::Livestatus->new(%options);
```

**Options:**
- `socket` (string): Socket path or URI
- `server` (string): Server hostname (for TCP)
- `port` (int): Port number (for TCP)
- `timeout` (int): Query timeout in seconds
- `keepalive` (bool): Enable keep-alive connections
- `errors_are_fatal` (bool): Die on errors vs. return undef
- `warnings` (bool): Enable warning messages

#### Methods

##### selectall_arrayref($query)
Execute query and return array reference of array references.
```perl
my $hosts = $ml->selectall_arrayref("GET hosts\nColumns: name state address");
# Returns: [['host1', 0, '192.168.1.1'], ['host2', 1, '192.168.1.2']]
```

##### selectall_hashref($query [, $key_field])
Execute query and return hash reference with specified key field.
```perl
my $hosts = $ml->selectall_hashref("GET hosts\nColumns: name state address", "name");
# Returns: { 'host1' => ['host1', 0, '192.168.1.1'], ... }
```

##### selectrow_array($query)
Execute query and return first row as array.
```perl
my @stats = $ml->selectrow_array("GET hosts\nStats: state = 0\nStats: state = 1");
# Returns: (10, 2)
```

##### selectrow_arrayref($query)
Execute query and return first row as array reference.
```perl
my $stats = $ml->selectrow_arrayref("GET hosts\nStats: state = 0\nStats: state = 1");
# Returns: [10, 2]
```

##### selectrow_hashref($query)
Execute query and return first row as hash reference.
```perl
my $status = $ml->selectrow_hashref("GET status");
# Returns: { livestatus_version => '1.5.0', nagios_version => '4.4.5', ... }
```

##### selectcol_arrayref($query)
Execute query and return first column as array reference.
```perl
my $names = $ml->selectcol_arrayref("GET hosts\nColumns: name");
# Returns: ['host1', 'host2', 'host3']
```

##### do($query)
Execute query without returning results (for commands).
```perl
$ml->do("COMMAND [" . time() . "] DISABLE_HOST_CHECK;hostname");
```

#### Options Methods

```perl
# Set socket
$ml->socket('/tmp/livestatus');

# Set timeout
$ml->timeout(30);

# Enable/disable warnings
$ml->warnings(1);

# Enable/disable fatal errors
$ml->errors_are_fatal(1);
```

### Example Usage

```perl
use Monitoring::Livestatus;

# Basic connection
my $ml = Monitoring::Livestatus->new(
    socket => '/tmp/livestatus',
    timeout => 30
);

# Error handling
eval {
    my $hosts = $ml->selectall_arrayref("GET hosts\nColumns: name state");
    for my $host (@$hosts) {
        my ($name, $state) = @$host;
        print "$name: $state\n";
    }
};
if ($@) {
    print "Error: $@\n";
}

# TCP connection
my $tcp_ml = Monitoring::Livestatus->new(
    server => 'monitoring.example.com',
    port => 6557,
    timeout => 10
);
```

## C++ API Reference

### Class: Livestatus

#### Constructor/Destructor

```cpp
Livestatus();
~Livestatus();
```

#### Connection Methods

##### connectUNIX(const char* socket_path)
Connect to Unix socket.
```cpp
Livestatus live;
live.connectUNIX("/tmp/livestatus");
```

##### connectTCP(const char* host, int port)
Connect to TCP socket.
```cpp
Livestatus live;
live.connectTCP("localhost", 6557);
```

##### disconnect()
Disconnect from socket.
```cpp
live.disconnect();
```

##### isConnected()
Check connection status.
```cpp
if (live.isConnected()) {
    // Connection is active
}
```

#### Query Methods

##### sendQuery(const char* query)
Send query to Livestatus.
```cpp
live.sendQuery("GET hosts\nColumns: name state address");
```

##### nextRow()
Get next row from query result.
```cpp
std::vector<std::string>* row;
while ((row = live.nextRow()) != nullptr) {
    // Process row data
    for (const auto& field : *row) {
        std::cout << field << " ";
    }
    std::cout << std::endl;
    delete row;  // Important: free memory
}
```

#### Configuration Methods

##### setTimeout(int seconds)
Set query timeout.
```cpp
live.setTimeout(30);
```

##### setOutputFormat(const std::string& format)
Set output format.
```cpp
live.setOutputFormat("json");
live.setOutputFormat("csv");
```

### Example Usage

```cpp
#include "Livestatus.h"
#include <iostream>
#include <vector>
#include <string>

int main() {
    Livestatus live;
    
    // Connect
    if (!live.connectUNIX("/tmp/livestatus")) {
        std::cerr << "Failed to connect" << std::endl;
        return 1;
    }
    
    // Query
    live.sendQuery("GET hosts\nColumns: name state address");
    
    // Process results
    std::vector<std::string>* row;
    while ((row = live.nextRow()) != nullptr) {
        if (row->size() >= 3) {
            std::cout << "Host: " << (*row)[0] 
                      << ", State: " << (*row)[1]
                      << ", Address: " << (*row)[2] << std::endl;
        }
        delete row;
    }
    
    return 0;
}
```

## Output Formats

### CSV Format (Default)

Field separator: `;` (semicolon)  
Record separator: `\n` (newline)  
List separator: `,` (comma)  
Host/Service separator: `|` (pipe)

```
name;state;address
host1;0;192.168.1.1
host2;1;192.168.1.2
```

### JSON Format

```json
[
  ["name", "state", "address"],
  ["host1", 0, "192.168.1.1"],
  ["host2", 1, "192.168.1.2"]
]
```

With `OutputFormat: json` and `ColumnHeaders: on`:
```json
[
  {"name": "host1", "state": 0, "address": "192.168.1.1"},
  {"name": "host2", "state": 1, "address": "192.168.1.2"}
]
```

### Python Format

Native Python data structures:
```python
[
    ['host1', 0, '192.168.1.1'],
    ['host2', 1, '192.168.1.2']
]
```

### Custom Separators

Use the `Separators` directive to define custom separators:
```
Separators: 10 59 44 124
# 10 = \n (record)
# 59 = ; (field)  
# 44 = , (list)
# 124 = | (host/service)
```

## Error Handling

### Error Response Format

Livestatus returns error responses in the following format:
```
400: Bad Request - <error message>
404: Table 'invalid_table' does not exist
```

### Common Error Codes

| Code | Description | Example |
|------|-------------|---------|
| 400 | Bad Request | Invalid query syntax |
| 404 | Not Found | Invalid table or column name |
| 452 | Invalid Header | Invalid header directive |

### Python Error Handling

```python
import livestatus

try:
    conn = livestatus.SingleSiteConnection("unix:/tmp/livestatus")
    result = conn.query_table("GET invalid_table")
except livestatus.MKLivestatusSocketError:
    print("Cannot connect to socket")
except livestatus.MKLivestatusException as e:
    print(f"Query error: {e}")
```

### Perl Error Handling

```perl
eval {
    my $result = $ml->selectall_arrayref("GET invalid_table");
};
if ($@) {
    print "Error: $@\n";
}

# Or with errors_are_fatal => 0
$ml->errors_are_fatal(0);
my $result = $ml->selectall_arrayref("GET invalid_table");
if (!defined $result) {
    print "Query failed\n";
}
```

### C++ Error Handling

```cpp
try {
    live.sendQuery("GET invalid_table");
    // Process results...
} catch (const std::exception& e) {
    std::cerr << "Error: " << e.what() << std::endl;
}
```

## Performance Considerations

### Query Optimization

#### Column Selection
Always specify only needed columns:
```bash
# Good
GET hosts
Columns: name state

# Avoid
GET hosts  # Returns all columns
```

#### Server-side Filtering
Apply filters server-side rather than client-side:
```bash
# Good
GET services
Filter: state > 0
Filter: acknowledged = 0

# Avoid downloading all data and filtering client-side
```

#### Limit Results
Use `Limit` for large datasets:
```bash
GET log
Filter: time > 1609459200
Limit: 1000
```

#### Statistics vs. Individual Records
Use statistics when possible:
```bash
# Good for counts
GET services
Stats: state = 0
Stats: state = 1
Stats: state = 2

# Avoid for simple counting
GET services
Filter: state = 0
# Then counting client-side
```

### Connection Management

#### Keep-Alive Connections
Use keep-alive for multiple queries:
```bash
GET hosts
KeepAlive: on

GET services
KeepAlive: on

# Final query without keep-alive
GET status
```

#### Connection Pooling
Implement connection pooling for high-frequency applications:

```python
import threading
import queue
import livestatus

class ConnectionPool:
    def __init__(self, socket_path, pool_size=5):
        self.socket_path = socket_path
        self.pool = queue.Queue(maxsize=pool_size)
        for _ in range(pool_size):
            conn = livestatus.SingleSiteConnection(socket_path)
            self.pool.put(conn)
    
    def get_connection(self):
        return self.pool.get()
    
    def return_connection(self, conn):
        self.pool.put(conn)
    
    def execute_query(self, query):
        conn = self.get_connection()
        try:
            return conn.query_table(query)
        finally:
            self.return_connection(conn)
```

### Memory Management

#### C++ Memory Management
Always delete row pointers:
```cpp
std::vector<std::string>* row;
while ((row = live.nextRow()) != nullptr) {
    // Process row
    delete row;  // Critical!
}
```

#### Large Result Sets
For large result sets, consider streaming processing:
```python
def process_large_query(conn, query, chunk_size=1000):
    offset = 0
    while True:
        chunk_query = f"{query}\nLimit: {chunk_size}\nOffset: {offset}"
        rows = conn.query_table(chunk_query)
        if not rows:
            break
        
        for row in rows:
            yield row
        
        offset += chunk_size
```

### Monitoring Performance

#### Query Timing
Monitor query execution times:
```python
import time

start_time = time.time()
result = conn.query_table(query)
execution_time = time.time() - start_time
print(f"Query took {execution_time:.2f} seconds")
```

#### Resource Usage
Monitor Livestatus resource usage:
```bash
# Check socket statistics
echo "GET status
Columns: num_queued_connections num_cached_log_messages" | unixcat /tmp/livestatus
```

This API reference provides comprehensive documentation for all aspects of MK Livestatus APIs and query capabilities. Use it as a reference while developing applications that interact with Livestatus.