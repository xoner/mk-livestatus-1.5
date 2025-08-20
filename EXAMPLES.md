# Examples and Use Cases

This document provides practical examples of using MK Livestatus in various scenarios, from simple queries to complex monitoring dashboards.

## Table of Contents

1. [Basic Query Examples](#basic-query-examples)
2. [Python API Examples](#python-api-examples)
3. [Perl API Examples](#perl-api-examples)
4. [C++ API Examples](#cpp-api-examples)
5. [Command Line Examples](#command-line-examples)
6. [Monitoring Dashboard Examples](#monitoring-dashboard-examples)
7. [Performance Monitoring](#performance-monitoring)
8. [Custom Applications](#custom-applications)
9. [Troubleshooting Queries](#troubleshooting-queries)
10. [Advanced Techniques](#advanced-techniques)

## Basic Query Examples

### Simple Status Queries

#### Get All Hosts
```
GET hosts
Columns: name state address alias
```

#### Get Services with Problems
```
GET services
Filter: state > 0
Filter: scheduled_downtime_depth = 0
Filter: acknowledged = 0
Columns: host_name description state plugin_output
```

#### Count Hosts by State
```
GET hosts
Stats: state = 0
Stats: state = 1
Stats: state = 2
```

#### Get Recent Log Entries
```
GET log
Filter: time > 1609459200
Filter: class = 1
Columns: time type message
Limit: 100
```

### Filtering and Sorting

#### Complex Filter Logic
```
GET services
Filter: state = 1
Filter: state = 2
Or: 2
Filter: acknowledged = 0
Columns: host_name description state plugin_output last_check
```

#### Regular Expression Filtering
```
GET hosts
Filter: name ~ ^web.*
Columns: name address state
```

#### Time-based Filtering
```
GET services
Filter: last_state_change > 1609459200
Filter: state > 0
Columns: host_name description last_state_change state
```

## Python API Examples

### Basic Connection and Queries

```python
#!/usr/bin/env python3
import sys
import livestatus
from datetime import datetime, timedelta

# Single site connection
def connect_livestatus(socket_path="/tmp/livestatus"):
    """Create a connection to Livestatus"""
    try:
        return livestatus.SingleSiteConnection(f"unix:{socket_path}")
    except Exception as e:
        print(f"Failed to connect to Livestatus: {e}")
        sys.exit(1)

# Basic status overview
def get_status_overview(conn):
    """Get basic monitoring status overview"""
    try:
        # Get general status
        status = conn.query_row_assoc("GET status")
        print(f"Nagios version: {status.get('nagios_version', 'Unknown')}")
        print(f"Livestatus version: {status.get('livestatus_version', 'Unknown')}")
        print(f"Uptime: {status.get('program_start', 0)} seconds")
        
        # Host statistics
        host_stats = conn.query_row(
            "GET hosts\n"
            "Stats: state = 0\n"
            "Stats: state = 1\n"
            "Stats: state = 2\n"
        )
        up, down, unreachable = host_stats
        print(f"\nHost Status:")
        print(f"  Up: {up}, Down: {down}, Unreachable: {unreachable}")
        
        # Service statistics
        service_stats = conn.query_row(
            "GET services\n"
            "Stats: state = 0\n"
            "Stats: state = 1\n"
            "Stats: state = 2\n"
            "Stats: state = 3\n"
        )
        ok, warning, critical, unknown = service_stats
        print(f"\nService Status:")
        print(f"  OK: {ok}, Warning: {warning}, Critical: {critical}, Unknown: {unknown}")
        
    except Exception as e:
        print(f"Error getting status: {e}")

# Get problem services
def get_problem_services(conn, limit=10):
    """Get services with problems"""
    query = f"""GET services
Filter: state > 0
Filter: scheduled_downtime_depth = 0
Filter: acknowledged = 0
Columns: host_name description state plugin_output last_check
Limit: {limit}"""
    
    try:
        services = conn.query_table(query)
        print(f"\nTop {limit} Problem Services:")
        print("-" * 80)
        for host, service, state, output, last_check in services:
            state_names = ["OK", "WARNING", "CRITICAL", "UNKNOWN"]
            state_name = state_names[state] if state < len(state_names) else "UNKNOWN"
            timestamp = datetime.fromtimestamp(last_check).strftime("%Y-%m-%d %H:%M:%S")
            print(f"{host:20} {service:30} {state_name:8} {timestamp}")
            print(f"    Output: {output[:60]}...")
            
    except Exception as e:
        print(f"Error getting problem services: {e}")

# Performance data collection
def collect_performance_data(conn):
    """Collect performance metrics"""
    try:
        # Get service performance data
        query = """GET services
Filter: perf_data !=
Columns: host_name description perf_data
Limit: 5"""
        
        services = conn.query_table(query)
        print("\nPerformance Data Sample:")
        print("-" * 60)
        for host, service, perf_data in services:
            print(f"{host}/{service}: {perf_data}")
            
    except Exception as e:
        print(f"Error collecting performance data: {e}")

# Main execution
if __name__ == "__main__":
    conn = connect_livestatus()
    get_status_overview(conn)
    get_problem_services(conn, 5)
    collect_performance_data()
```

### Multi-Site Monitoring

```python
#!/usr/bin/env python3
import livestatus
import json
from collections import defaultdict

def setup_multisite_connection():
    """Setup connection to multiple monitoring sites"""
    sites = {
        "production": {
            "socket": "unix:/omd/production/tmp/run/live",
            "alias": "Production Environment"
        },
        "staging": {
            "socket": "unix:/omd/staging/tmp/run/live", 
            "alias": "Staging Environment"
        },
        "development": {
            "socket": "unix:/omd/development/tmp/run/live",
            "alias": "Development Environment"
        }
    }
    
    return livestatus.MultiSiteConnection(sites)

def get_multisite_overview(conn):
    """Get overview across all sites"""
    try:
        # Get status from all sites
        query = "GET status\nColumns: livestatus_version nagios_version program_start"
        sites_status = conn.query_table_assoc(query)
        
        print("Multi-Site Status Overview:")
        print("=" * 50)
        for site_data in sites_status:
            site_name = site_data.get('site', 'Unknown')
            version = site_data.get('livestatus_version', 'Unknown')
            nagios_ver = site_data.get('nagios_version', 'Unknown')
            uptime = site_data.get('program_start', 0)
            
            print(f"Site: {site_name}")
            print(f"  Livestatus: {version}")
            print(f"  Nagios: {nagios_ver}")
            print(f"  Uptime: {uptime} seconds")
            print()
            
    except Exception as e:
        print(f"Error getting multi-site status: {e}")

def aggregate_problems_by_site(conn):
    """Aggregate problems across all sites"""
    try:
        query = """GET services
Filter: state > 0
Filter: scheduled_downtime_depth = 0
Filter: acknowledged = 0
Columns: host_name description state"""
        
        problems = conn.query_table_assoc(query)
        
        # Group by site
        site_problems = defaultdict(list)
        for problem in problems:
            site_name = problem.get('site', 'Unknown')
            site_problems[site_name].append(problem)
        
        print("Problems by Site:")
        print("=" * 40)
        for site, problems_list in site_problems.items():
            print(f"\n{site}: {len(problems_list)} problems")
            for problem in problems_list[:5]:  # Show first 5
                state_names = ["OK", "WARNING", "CRITICAL", "UNKNOWN"]
                state = problem['state']
                state_name = state_names[state] if state < len(state_names) else "UNKNOWN"
                print(f"  {problem['host_name']}/{problem['description']}: {state_name}")
            
            if len(problems_list) > 5:
                print(f"  ... and {len(problems_list) - 5} more")
                
    except Exception as e:
        print(f"Error aggregating problems: {e}")

# Usage
if __name__ == "__main__":
    conn = setup_multisite_connection()
    get_multisite_overview(conn)
    aggregate_problems_by_site(conn)
```

### Custom Monitoring Dashboard

```python
#!/usr/bin/env python3
import livestatus
import time
import json
from datetime import datetime, timedelta

class MonitoringDashboard:
    def __init__(self, socket_path="/tmp/livestatus"):
        self.conn = livestatus.SingleSiteConnection(f"unix:{socket_path}")
        
    def get_critical_services(self):
        """Get critical services requiring immediate attention"""
        query = """GET services
Filter: state = 2
Filter: scheduled_downtime_depth = 0
Filter: acknowledged = 0
Columns: host_name description plugin_output last_state_change"""
        
        return self.conn.query_table(query)
    
    def get_host_summary(self):
        """Get summary of host states"""
        query = """GET hosts
Stats: state = 0
Stats: state = 1  
Stats: state = 2"""
        
        return self.conn.query_row(query)
    
    def get_service_summary(self):
        """Get summary of service states"""
        query = """GET services
Stats: state = 0
Stats: state = 1
Stats: state = 2
Stats: state = 3"""
        
        return self.conn.query_row(query)
    
    def get_recent_events(self, hours=1):
        """Get recent events from log"""
        since = int(time.time()) - (hours * 3600)
        query = f"""GET log
Filter: time > {since}
Filter: class = 1
Columns: time type message
Limit: 20"""
        
        return self.conn.query_table(query)
    
    def get_top_flapping_services(self, limit=10):
        """Get services that are flapping"""
        query = f"""GET services
Filter: is_flapping = 1
Columns: host_name description percent_state_change
Limit: {limit}"""
        
        return self.conn.query_table(query)
    
    def generate_dashboard_data(self):
        """Generate complete dashboard data"""
        dashboard = {
            "timestamp": datetime.now().isoformat(),
            "critical_services": self.get_critical_services(),
            "host_summary": {
                "up": self.get_host_summary()[0],
                "down": self.get_host_summary()[1], 
                "unreachable": self.get_host_summary()[2]
            },
            "service_summary": {
                "ok": self.get_service_summary()[0],
                "warning": self.get_service_summary()[1],
                "critical": self.get_service_summary()[2],
                "unknown": self.get_service_summary()[3]
            },
            "recent_events": self.get_recent_events(),
            "flapping_services": self.get_top_flapping_services()
        }
        
        return dashboard
    
    def print_dashboard(self):
        """Print formatted dashboard"""
        data = self.generate_dashboard_data()
        
        print("=" * 60)
        print(f"Monitoring Dashboard - {data['timestamp']}")
        print("=" * 60)
        
        # Host summary
        hs = data['host_summary']
        print(f"\nHosts: {hs['up']} UP, {hs['down']} DOWN, {hs['unreachable']} UNREACHABLE")
        
        # Service summary
        ss = data['service_summary']
        print(f"Services: {ss['ok']} OK, {ss['warning']} WARN, {ss['critical']} CRIT, {ss['unknown']} UNK")
        
        # Critical services
        critical = data['critical_services']
        if critical:
            print(f"\nCRITICAL SERVICES ({len(critical)}):")
            for host, service, output, last_change in critical[:10]:
                age = int(time.time()) - last_change
                print(f"  {host}/{service} ({age}s ago)")
                print(f"    {output[:60]}...")
        
        # Recent events
        events = data['recent_events']
        if events:
            print(f"\nRECENT EVENTS ({len(events)}):")
            for timestamp, event_type, message in events[:5]:
                dt = datetime.fromtimestamp(timestamp)
                print(f"  {dt.strftime('%H:%M:%S')}: {message[:50]}...")

# Usage
if __name__ == "__main__":
    dashboard = MonitoringDashboard()
    dashboard.print_dashboard()
```

## Perl API Examples

### Basic Status Check

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Monitoring::Livestatus;

# Create connection
my $ml = Monitoring::Livestatus->new(
    socket => '/tmp/livestatus',
    timeout => 30
);

# Get status overview
sub get_status_overview {
    my $ml = shift;
    
    # Get general status
    my $status = $ml->selectrow_hashref("GET status");
    print "Nagios Version: " . ($status->{nagios_version} // 'Unknown') . "\n";
    print "Livestatus Version: " . ($status->{livestatus_version} // 'Unknown') . "\n";
    
    # Host statistics
    my $host_stats = $ml->selectrow_arrayref(
        "GET hosts\n" .
        "Stats: state = 0\n" .
        "Stats: state = 1\n" .
        "Stats: state = 2"
    );
    
    my ($up, $down, $unreachable) = @$host_stats;
    print "\nHost Status:\n";
    print "  Up: $up, Down: $down, Unreachable: $unreachable\n";
    
    # Service statistics  
    my $service_stats = $ml->selectrow_arrayref(
        "GET services\n" .
        "Stats: state = 0\n" .
        "Stats: state = 1\n" .
        "Stats: state = 2\n" .
        "Stats: state = 3"
    );
    
    my ($ok, $warning, $critical, $unknown) = @$service_stats;
    print "\nService Status:\n";
    print "  OK: $ok, Warning: $warning, Critical: $critical, Unknown: $unknown\n";
}

# Get problem services
sub get_problem_services {
    my ($ml, $limit) = @_;
    $limit //= 10;
    
    my $services = $ml->selectall_arrayref(
        "GET services\n" .
        "Filter: state > 0\n" .
        "Filter: scheduled_downtime_depth = 0\n" .
        "Filter: acknowledged = 0\n" .
        "Columns: host_name description state plugin_output\n" .
        "Limit: $limit"
    );
    
    print "\nTop $limit Problem Services:\n";
    print "-" x 80 . "\n";
    
    my @state_names = qw(OK WARNING CRITICAL UNKNOWN);
    
    for my $service (@$services) {
        my ($host, $desc, $state, $output) = @$service;
        my $state_name = $state_names[$state] // 'UNKNOWN';
        
        printf "%-20s %-30s %-8s\n", $host, $desc, $state_name;
        printf "    Output: %.60s...\n", $output;
    }
}

# Host availability report
sub host_availability_report {
    my ($ml, $days) = @_;
    $days //= 7;
    
    my $since = time() - ($days * 24 * 3600);
    
    my $hosts = $ml->selectall_arrayref(
        "GET hosts\n" .
        "Columns: name state total_services services_ok services_warn services_crit services_unknown\n" .
        "Filter: last_check > $since"
    );
    
    print "\nHost Availability Report (Last $days days):\n";
    print "-" x 80 . "\n";
    printf "%-20s %-8s %-15s %-15s\n", "Host", "State", "Total Services", "Problems";
    print "-" x 80 . "\n";
    
    for my $host (@$hosts) {
        my ($name, $state, $total, $ok, $warn, $crit, $unknown) = @$host;
        my $problems = $warn + $crit + $unknown;
        my @state_names = qw(UP DOWN UNREACHABLE);
        my $state_name = $state_names[$state] // 'UNKNOWN';
        
        printf "%-20s %-8s %-15d %-15d\n", $name, $state_name, $total, $problems;
    }
}

# Main execution
get_status_overview($ml);
get_problem_services($ml, 5);
host_availability_report($ml, 7);
```

### Log Analysis

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Monitoring::Livestatus;
use POSIX qw(strftime);

my $ml = Monitoring::Livestatus->new(socket => '/tmp/livestatus');

# Analyze log entries for patterns
sub analyze_log_patterns {
    my ($ml, $hours) = @_;
    $hours //= 24;
    
    my $since = time() - ($hours * 3600);
    
    my $logs = $ml->selectall_arrayref(
        "GET log\n" .
        "Filter: time > $since\n" .
        "Columns: time type message state_type\n" .
        "Limit: 1000"
    );
    
    my %event_counts;
    my %hourly_counts;
    
    for my $log (@$logs) {
        my ($time, $type, $message, $state_type) = @$log;
        
        # Count by event type
        $event_counts{$type}++;
        
        # Count by hour
        my $hour = strftime("%Y-%m-%d %H:00", localtime($time));
        $hourly_counts{$hour}++;
    }
    
    print "\nLog Analysis (Last $hours hours):\n";
    print "=" x 50 . "\n";
    
    print "\nEvent Type Counts:\n";
    for my $type (sort keys %event_counts) {
        printf "  Type %d: %d events\n", $type, $event_counts{$type};
    }
    
    print "\nHourly Event Distribution:\n";
    for my $hour (sort keys %hourly_counts) {
        printf "  %s: %d events\n", $hour, $hourly_counts{$hour};
    }
}

# Find recurring problems
sub find_recurring_problems {
    my ($ml, $days) = @_;
    $days //= 7;
    
    my $since = time() - ($days * 24 * 3600);
    
    my $services = $ml->selectall_arrayref(
        "GET services\n" .
        "Columns: host_name description check_interval max_check_attempts current_attempt\n" .
        "Filter: last_hard_state_change > $since\n" .
        "Filter: state > 0"
    );
    
    my %problem_counts;
    
    for my $service (@$services) {
        my ($host, $desc, $interval, $max_attempts, $current) = @$service;
        my $key = "$host/$desc";
        $problem_counts{$key}++;
    }
    
    print "\nRecurring Problems (Last $days days):\n";
    print "=" x 60 . "\n";
    
    my @sorted = sort { $problem_counts{$b} <=> $problem_counts{$a} } keys %problem_counts;
    
    for my $service (@sorted[0..9]) {  # Top 10
        printf "%-40s: %d occurrences\n", $service, $problem_counts{$service};
    }
}

# Main execution
analyze_log_patterns($ml, 24);
find_recurring_problems($ml, 7);
```

## C++ API Examples

### Basic Connection and Query

```cpp
#include <iostream>
#include <string>
#include <vector>
#include "Livestatus.h"

class LivestatusMonitor {
private:
    Livestatus live;
    
public:
    bool connect(const std::string& socket_path) {
        live.connectUNIX(socket_path.c_str());
        return live.isConnected();
    }
    
    void getStatusOverview() {
        std::string query = 
            "GET status\n"
            "Columns: livestatus_version nagios_version program_start\n";
        
        live.sendQuery(query.c_str());
        
        std::vector<std::string>* row = live.nextRow();
        if (row) {
            std::cout << "Livestatus Version: " << (*row)[0] << std::endl;
            std::cout << "Nagios Version: " << (*row)[1] << std::endl;
            std::cout << "Program Start: " << (*row)[2] << std::endl;
            delete row;
        }
    }
    
    void getHostStatistics() {
        std::string query = 
            "GET hosts\n"
            "Stats: state = 0\n"
            "Stats: state = 1\n"
            "Stats: state = 2\n";
        
        live.sendQuery(query.c_str());
        
        std::vector<std::string>* row = live.nextRow();
        if (row) {
            std::cout << "\nHost Statistics:" << std::endl;
            std::cout << "  Up: " << (*row)[0] << std::endl;
            std::cout << "  Down: " << (*row)[1] << std::endl;
            std::cout << "  Unreachable: " << (*row)[2] << std::endl;
            delete row;
        }
    }
    
    void getProblemServices(int limit = 10) {
        std::string query = 
            "GET services\n"
            "Filter: state > 0\n"
            "Filter: scheduled_downtime_depth = 0\n"
            "Filter: acknowledged = 0\n"
            "Columns: host_name description state plugin_output\n"
            "Limit: " + std::to_string(limit) + "\n";
        
        live.sendQuery(query.c_str());
        
        std::cout << "\nProblem Services:" << std::endl;
        std::cout << std::string(60, '-') << std::endl;
        
        std::vector<std::string>* row;
        while ((row = live.nextRow()) != nullptr) {
            std::string state_name;
            int state = std::stoi((*row)[2]);
            switch (state) {
                case 0: state_name = "OK"; break;
                case 1: state_name = "WARNING"; break;
                case 2: state_name = "CRITICAL"; break;
                case 3: state_name = "UNKNOWN"; break;
                default: state_name = "UNKNOWN"; break;
            }
            
            std::cout << (*row)[0] << "/" << (*row)[1] 
                      << " [" << state_name << "]" << std::endl;
            std::cout << "  " << (*row)[3].substr(0, 60) << "..." << std::endl;
            
            delete row;
        }
    }
    
    void getPerformanceData() {
        std::string query = 
            "GET services\n"
            "Filter: perf_data !=\n"
            "Columns: host_name description perf_data\n"
            "Limit: 5\n";
        
        live.sendQuery(query.c_str());
        
        std::cout << "\nPerformance Data Sample:" << std::endl;
        std::cout << std::string(60, '-') << std::endl;
        
        std::vector<std::string>* row;
        while ((row = live.nextRow()) != nullptr) {
            std::cout << (*row)[0] << "/" << (*row)[1] << ": " 
                      << (*row)[2] << std::endl;
            delete row;
        }
    }
};

int main(int argc, char** argv) {
    if (argc != 2) {
        std::cerr << "Usage: " << argv[0] << " <socket_path>" << std::endl;
        return 1;
    }
    
    LivestatusMonitor monitor;
    
    if (!monitor.connect(argv[1])) {
        std::cerr << "Failed to connect to Livestatus socket: " << argv[1] << std::endl;
        return 1;
    }
    
    monitor.getStatusOverview();
    monitor.getHostStatistics();
    monitor.getProblemServices(5);
    monitor.getPerformanceData();
    
    return 0;
}
```

### Compile the C++ example:
```bash
g++ -std=c++17 -I/usr/local/include -L/usr/local/lib \
    -o monitor_example monitor_example.cpp -llivestatus
```

## Command Line Examples

### Using unixcat for Quick Queries

#### System Status Check
```bash
#!/bin/bash
SOCKET="/tmp/livestatus"

echo "=== Livestatus Status Check ==="

# Get basic status
echo "GET status" | unixcat $SOCKET | head -2

# Host counts
echo "--- Host Summary ---"
echo "GET hosts
Stats: state = 0
Stats: state = 1  
Stats: state = 2" | unixcat $SOCKET

# Service counts
echo "--- Service Summary ---"
echo "GET services
Stats: state = 0
Stats: state = 1
Stats: state = 2
Stats: state = 3" | unixcat $SOCKET

# Critical services
echo "--- Critical Services ---"
echo "GET services
Filter: state = 2
Filter: scheduled_downtime_depth = 0
Filter: acknowledged = 0
Columns: host_name description plugin_output
Limit: 5" | unixcat $SOCKET
```

#### Performance Monitoring Script
```bash
#!/bin/bash
SOCKET="/tmp/livestatus"
LOGFILE="/var/log/monitoring_stats.log"

# Function to log with timestamp
log_with_time() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> $LOGFILE
}

# Collect host statistics
HOST_STATS=$(echo "GET hosts
Stats: state = 0
Stats: state = 1
Stats: state = 2" | unixcat $SOCKET)

UP=$(echo $HOST_STATS | cut -d';' -f1)
DOWN=$(echo $HOST_STATS | cut -d';' -f2)
UNREACHABLE=$(echo $HOST_STATS | cut -d';' -f3)

log_with_time "HOSTS up=$UP down=$DOWN unreachable=$UNREACHABLE"

# Collect service statistics
SERVICE_STATS=$(echo "GET services
Stats: state = 0
Stats: state = 1
Stats: state = 2
Stats: state = 3" | unixcat $SOCKET)

OK=$(echo $SERVICE_STATS | cut -d';' -f1)
WARNING=$(echo $SERVICE_STATS | cut -d';' -f2)
CRITICAL=$(echo $SERVICE_STATS | cut -d';' -f3)
UNKNOWN=$(echo $SERVICE_STATS | cut -d';' -f4)

log_with_time "SERVICES ok=$OK warning=$WARNING critical=$CRITICAL unknown=$UNKNOWN"

# Alert on critical services
if [ $CRITICAL -gt 0 ]; then
    log_with_time "ALERT: $CRITICAL critical services detected"
fi
```

## Monitoring Dashboard Examples

### Web Dashboard (PHP)
```php
<?php
// Simple PHP dashboard using Livestatus
class LivestatusDashboard {
    private $socket_path;
    
    public function __construct($socket_path = '/tmp/livestatus') {
        $this->socket_path = $socket_path;
    }
    
    private function query($query) {
        $socket = socket_create(AF_UNIX, SOCK_STREAM, 0);
        if (!socket_connect($socket, $this->socket_path)) {
            throw new Exception("Cannot connect to Livestatus socket");
        }
        
        socket_write($socket, $query . "\n");
        socket_shutdown($socket, 1);
        
        $result = '';
        while ($line = socket_read($socket, 1024)) {
            $result .= $line;
        }
        
        socket_close($socket);
        return trim($result);
    }
    
    public function getStatusSummary() {
        $host_query = "GET hosts\nStats: state = 0\nStats: state = 1\nStats: state = 2";
        $service_query = "GET services\nStats: state = 0\nStats: state = 1\nStats: state = 2\nStats: state = 3";
        
        $host_stats = explode(';', $this->query($host_query));
        $service_stats = explode(';', $this->query($service_query));
        
        return [
            'hosts' => [
                'up' => $host_stats[0],
                'down' => $host_stats[1],
                'unreachable' => $host_stats[2]
            ],
            'services' => [
                'ok' => $service_stats[0],
                'warning' => $service_stats[1], 
                'critical' => $service_stats[2],
                'unknown' => $service_stats[3]
            ]
        ];
    }
    
    public function getCriticalServices($limit = 10) {
        $query = "GET services\n" .
                "Filter: state = 2\n" .
                "Filter: scheduled_downtime_depth = 0\n" .
                "Filter: acknowledged = 0\n" .
                "Columns: host_name description plugin_output\n" .
                "Limit: $limit";
        
        $result = $this->query($query);
        $services = [];
        
        foreach (explode("\n", $result) as $line) {
            if (trim($line)) {
                $parts = explode(';', $line);
                $services[] = [
                    'host' => $parts[0],
                    'service' => $parts[1],
                    'output' => $parts[2]
                ];
            }
        }
        
        return $services;
    }
}

// Usage in HTML dashboard
$dashboard = new LivestatusDashboard();
$summary = $dashboard->getStatusSummary();
$critical = $dashboard->getCriticalServices(5);
?>

<!DOCTYPE html>
<html>
<head>
    <title>Monitoring Dashboard</title>
    <style>
        .status-box { display: inline-block; margin: 10px; padding: 20px; border: 1px solid #ccc; }
        .critical { background-color: #ffcccc; }
        .warning { background-color: #fff3cd; }
        .ok { background-color: #d4edda; }
    </style>
</head>
<body>
    <h1>Monitoring Dashboard</h1>
    
    <div class="status-summary">
        <div class="status-box ok">
            <h3>Hosts Up</h3>
            <p><?php echo $summary['hosts']['up']; ?></p>
        </div>
        
        <div class="status-box <?php echo $summary['hosts']['down'] > 0 ? 'critical' : 'ok'; ?>">
            <h3>Hosts Down</h3>
            <p><?php echo $summary['hosts']['down']; ?></p>
        </div>
        
        <div class="status-box <?php echo $summary['services']['critical'] > 0 ? 'critical' : 'ok'; ?>">
            <h3>Critical Services</h3>
            <p><?php echo $summary['services']['critical']; ?></p>
        </div>
    </div>
    
    <?php if (!empty($critical)): ?>
    <h2>Critical Services</h2>
    <table border="1">
        <tr>
            <th>Host</th>
            <th>Service</th>
            <th>Output</th>
        </tr>
        <?php foreach ($critical as $service): ?>
        <tr>
            <td><?php echo htmlspecialchars($service['host']); ?></td>
            <td><?php echo htmlspecialchars($service['service']); ?></td>
            <td><?php echo htmlspecialchars(substr($service['output'], 0, 60)); ?>...</td>
        </tr>
        <?php endforeach; ?>
    </table>
    <?php endif; ?>
</body>
</html>
```

## Advanced Techniques

### Custom Metrics Collection

```python
#!/usr/bin/env python3
import livestatus
import json
import time
from collections import defaultdict

class MetricsCollector:
    def __init__(self, socket_path="/tmp/livestatus"):
        self.conn = livestatus.SingleSiteConnection(f"unix:{socket_path}")
        
    def collect_response_times(self):
        """Collect service response times"""
        query = """GET services
Filter: perf_data ~ time=
Columns: host_name description perf_data execution_time
Limit: 100"""
        
        services = self.conn.query_table(query)
        metrics = []
        
        for host, service, perf_data, exec_time in services:
            # Parse performance data for response time
            if 'time=' in perf_data:
                try:
                    # Extract time value from performance data
                    time_part = [p for p in perf_data.split() if 'time=' in p][0]
                    time_value = float(time_part.split('=')[1].split('s')[0])
                    
                    metrics.append({
                        'host': host,
                        'service': service,
                        'response_time': time_value,
                        'execution_time': exec_time,
                        'timestamp': int(time.time())
                    })
                except (ValueError, IndexError):
                    continue
                    
        return metrics
    
    def collect_availability_stats(self, days=30):
        """Collect availability statistics"""
        since = int(time.time()) - (days * 24 * 3600)
        
        query = f"""GET services
Filter: last_check > {since}
Columns: host_name description state last_hard_state_change percent_state_change
"""
        
        services = self.conn.query_table(query)
        stats = defaultdict(list)
        
        for host, service, state, last_change, state_change in services:
            uptime_pct = 100.0 - float(state_change)
            stats[host].append({
                'service': service,
                'current_state': state,
                'last_change': last_change,
                'uptime_percent': uptime_pct
            })
            
        return dict(stats)
    
    def export_metrics_json(self, filename):
        """Export metrics to JSON file"""
        metrics = {
            'timestamp': int(time.time()),
            'response_times': self.collect_response_times(),
            'availability': self.collect_availability_stats()
        }
        
        with open(filename, 'w') as f:
            json.dump(metrics, f, indent=2)
            
        print(f"Metrics exported to {filename}")

# Usage
collector = MetricsCollector()
collector.export_metrics_json('/tmp/monitoring_metrics.json')
```

### Real-time Monitoring

```python
#!/usr/bin/env python3
import livestatus
import time
import threading
from datetime import datetime

class RealTimeMonitor:
    def __init__(self, socket_path="/tmp/livestatus"):
        self.conn = livestatus.SingleSiteConnection(f"unix:{socket_path}")
        self.running = False
        self.callbacks = []
        
    def add_callback(self, callback):
        """Add callback function for events"""
        self.callbacks.append(callback)
        
    def check_for_changes(self):
        """Check for state changes"""
        last_check = int(time.time()) - 300  # Last 5 minutes
        
        query = f"""GET log
Filter: time > {last_check}
Filter: class = 1
Columns: time type message"""
        
        try:
            events = self.conn.query_table(query)
            for event in events:
                timestamp, event_type, message = event
                for callback in self.callbacks:
                    callback(timestamp, event_type, message)
        except Exception as e:
            print(f"Error checking for changes: {e}")
    
    def start_monitoring(self, interval=60):
        """Start real-time monitoring"""
        self.running = True
        
        def monitor_loop():
            while self.running:
                self.check_for_changes()
                time.sleep(interval)
                
        self.monitor_thread = threading.Thread(target=monitor_loop)
        self.monitor_thread.daemon = True
        self.monitor_thread.start()
        
    def stop_monitoring(self):
        """Stop monitoring"""
        self.running = False
        if hasattr(self, 'monitor_thread'):
            self.monitor_thread.join()

# Event handlers
def alert_handler(timestamp, event_type, message):
    """Handle critical alerts"""
    if 'CRITICAL' in message or 'DOWN' in message:
        dt = datetime.fromtimestamp(timestamp)
        print(f"ALERT [{dt}]: {message}")

def log_handler(timestamp, event_type, message):
    """Log all events"""
    dt = datetime.fromtimestamp(timestamp)
    with open('/var/log/livestatus_events.log', 'a') as f:
        f.write(f"[{dt}] Type {event_type}: {message}\n")

# Usage
monitor = RealTimeMonitor()
monitor.add_callback(alert_handler)
monitor.add_callback(log_handler)
monitor.start_monitoring(interval=30)

# Keep running
try:
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    monitor.stop_monitoring()
    print("Monitoring stopped")
```

These examples demonstrate the versatility and power of MK Livestatus for building custom monitoring solutions, from simple status checks to complex real-time dashboards and automated alerting systems.