# Installation Guide

This guide provides detailed instructions for building and installing MK Livestatus from source.

## System Requirements

### Operating Systems
- Linux (Ubuntu 16.04+, CentOS 7+, RHEL 7+, SUSE 12+)
- macOS 10.14+ (with Homebrew)
- FreeBSD 11+

### Monitoring Systems
- **Nagios**: Version 3.0 or later
- **Icinga**: Version 1.0 or later (use `--with-nagios4` for Icinga 2.x)

### Compiler Requirements
- **GCC**: Version 7 or later
- **Clang**: Version 5.0 or later
- **C++17 standard support** required

## Dependencies

### Required Dependencies

#### Build Tools
```bash
# Ubuntu/Debian
sudo apt-get install build-essential autoconf automake libtool pkg-config

# CentOS/RHEL
sudo yum install gcc-c++ autoconf automake libtool pkgconfig
# or for newer versions:
sudo dnf install gcc-c++ autoconf automake libtool pkgconfig

# macOS (with Homebrew)
brew install autoconf automake libtool pkg-config
```

#### C++ Compiler (if not available)
```bash
# Ubuntu/Debian (for GCC 7+)
sudo apt-get install gcc-7 g++-7

# CentOS 7 (enable devtoolset)
sudo yum install centos-release-scl
sudo yum install devtoolset-7-gcc-c++
scl enable devtoolset-7 bash

# macOS
# Clang is included with Xcode Command Line Tools
xcode-select --install
```

#### Boost Libraries
```bash
# Ubuntu/Debian
sudo apt-get install libboost-dev libboost-system-dev

# CentOS/RHEL
sudo yum install boost-devel
# or for newer versions:
sudo dnf install boost-devel

# macOS
brew install boost
```

#### RRD Library
```bash
# Ubuntu/Debian
sudo apt-get install librrd-dev

# CentOS/RHEL
sudo yum install rrdtool-devel
# or for newer versions:
sudo dnf install rrdtool-devel

# macOS
brew install rrdtool
```

### Optional Dependencies

#### RE2 Library (for enhanced regex support)
```bash
# Ubuntu/Debian
sudo apt-get install libre2-dev

# CentOS/RHEL (from EPEL)
sudo yum install epel-release
sudo yum install re2-devel

# macOS
brew install re2
```

#### Development Headers for Nagios/Icinga
```bash
# Ubuntu/Debian (Nagios)
sudo apt-get install nagios4-dev
# or for Icinga
sudo apt-get install icinga2-dev

# CentOS/RHEL
# Usually installed with nagios-devel or icinga2-devel packages
```

## Building from Source

### 1. Download Source Code

#### From Git Repository
```bash
git clone https://github.com/xoner/mk-livestatus-1.5.git
cd mk-livestatus-1.5
```

#### From Tarball
```bash
wget https://github.com/xoner/mk-livestatus-1.5/archive/master.tar.gz
tar -xzf master.tar.gz
cd mk-livestatus-1.5-master
```

### 2. Configure Build

#### Basic Configuration
```bash
./configure
```

#### Nagios 4/Icinga 2 Configuration
```bash
./configure --with-nagios4
```

#### Custom Installation Prefix
```bash
./configure --prefix=/opt/livestatus
```

#### With RE2 Support
```bash
./configure --with-re2
# or with custom path:
./configure --with-re2=/usr/local
```

#### Complete Example
```bash
./configure \
    --prefix=/usr/local \
    --with-nagios4 \
    --with-re2 \
    --enable-silent-rules
```

### 3. Common Configure Options

| Option | Description |
|--------|-------------|
| `--prefix=PATH` | Installation prefix (default: `/usr/local`) |
| `--with-nagios4` | Build for Nagios 4/Icinga 2 compatibility |
| `--with-re2[=PATH]` | Enable RE2 library support |
| `--disable-rrd-is-thread-safe` | Use librrd_th instead of librrd |
| `--enable-silent-rules` | Less verbose build output |
| `--disable-dependency-tracking` | Speed up one-time builds |

### 4. Build and Install

```bash
# Build (parallel build for faster compilation)
make -j$(nproc)

# Install (requires appropriate permissions)
sudo make install
```

### 5. Verify Installation

```bash
# Check if files are installed
ls -la /usr/local/lib/mk-livestatus/
ls -la /usr/local/bin/unixcat

# Check library dependencies
ldd /usr/local/lib/mk-livestatus/livestatus.o
```

## Platform-Specific Instructions

### Ubuntu/Debian

#### Complete Installation Script
```bash
#!/bin/bash
# Install dependencies
sudo apt-get update
sudo apt-get install -y \
    build-essential \
    autoconf \
    automake \
    libtool \
    pkg-config \
    libboost-dev \
    libboost-system-dev \
    librrd-dev \
    libre2-dev \
    git

# Clone and build
git clone https://github.com/xoner/mk-livestatus-1.5.git
cd mk-livestatus-1.5
./configure --with-nagios4 --with-re2
make -j$(nproc)
sudo make install
```

### CentOS/RHEL 7

#### With DevToolSet for Modern GCC
```bash
#!/bin/bash
# Enable EPEL and SCL repositories
sudo yum install -y epel-release centos-release-scl

# Install dependencies
sudo yum install -y \
    devtoolset-7-gcc-c++ \
    autoconf \
    automake \
    libtool \
    pkgconfig \
    boost-devel \
    rrdtool-devel \
    re2-devel \
    git

# Enable modern GCC
scl enable devtoolset-7 bash

# Clone and build
git clone https://github.com/xoner/mk-livestatus-1.5.git
cd mk-livestatus-1.5
./configure --with-nagios4 --with-re2
make -j$(nproc)
sudo make install
```

### macOS

#### Using Homebrew
```bash
#!/bin/bash
# Install dependencies
brew install \
    autoconf \
    automake \
    libtool \
    pkg-config \
    boost \
    rrdtool \
    re2 \
    git

# Clone and build
git clone https://github.com/xoner/mk-livestatus-1.5.git
cd mk-livestatus-1.5
./configure --with-nagios4 --with-re2
make -j$(sysctl -n hw.ncpu)
sudo make install
```

## Nagios/Icinga Configuration

### 1. Module Configuration

Add the following line to your main Nagios/Icinga configuration file:

#### For Unix Socket (Recommended)
```
# /etc/nagios/nagios.cfg or /etc/icinga/icinga.cfg
broker_module=/usr/local/lib/mk-livestatus/livestatus.o /tmp/livestatus
```

#### For TCP Socket
```
# Use with caution - no authentication by default
broker_module=/usr/local/lib/mk-livestatus/livestatus.o inet:6557
```

#### With Debug Options
```
broker_module=/usr/local/lib/mk-livestatus/livestatus.o /tmp/livestatus debug=1
```

### 2. Event Broker Options

Ensure your Nagios/Icinga configuration has appropriate event broker options:

```
# Minimum required options
event_broker_options=-1

# Or specific options if needed
event_broker_options=1048575
```

### 3. Socket Permissions

For Unix sockets, ensure proper permissions:

```bash
# Make socket accessible to web server user
sudo chmod 666 /tmp/livestatus
# or
sudo chgrp www-data /tmp/livestatus
sudo chmod 660 /tmp/livestatus
```

### 4. Restart Monitoring System

```bash
# Nagios
sudo systemctl restart nagios
# or
sudo service nagios restart

# Icinga
sudo systemctl restart icinga
# or
sudo service icinga restart

# Icinga 2
sudo systemctl restart icinga2
```

## Verification and Testing

### 1. Check Module Loading

Look for Livestatus messages in your monitoring system logs:

```bash
# Nagios
tail -f /var/log/nagios/nagios.log | grep -i livestatus

# Icinga
tail -f /var/log/icinga/icinga.log | grep -i livestatus

# Icinga 2
tail -f /var/log/icinga2/icinga2.log | grep -i livestatus
```

Expected output:
```
[1234567890] livestatus: Livestatus 1.5.0 by Mathias Kettner. Socket: '/tmp/livestatus'
[1234567890] livestatus: finished initialization
```

### 2. Test Socket Connectivity

#### Using unixcat
```bash
echo "GET status" | /usr/local/bin/unixcat /tmp/livestatus
```

#### Using netcat (for TCP sockets)
```bash
echo "GET status" | nc localhost 6557
```

#### Expected Response
```
accept_passive_host_checks;accept_passive_service_checks;cached_log_messages;...
1;1;0;...
```

### 3. Test with Python API

```python
#!/usr/bin/env python
import sys
sys.path.insert(0, '/usr/local/lib/python')

try:
    import livestatus
    conn = livestatus.SingleSiteConnection("unix:/tmp/livestatus")
    status = conn.query_row_assoc("GET status")
    print(f"Livestatus version: {status['livestatus_version']}")
    print("Connection successful!")
except Exception as e:
    print(f"Error: {e}")
```

## Troubleshooting

### Common Build Issues

#### 1. Missing Dependencies
```
Error: configure: error: Boost library not found or too old
```
**Solution**: Install Boost development packages
```bash
# Ubuntu/Debian
sudo apt-get install libboost-dev

# CentOS/RHEL
sudo yum install boost-devel
```

#### 2. C++17 Compiler Issues
```
Error: C++ headers are too old. Please install a newer g++/clang/libc++-dev package.
```
**Solution**: Update compiler or enable newer toolset
```bash
# CentOS 7
sudo yum install centos-release-scl devtoolset-7-gcc-c++
scl enable devtoolset-7 bash
```

#### 3. RRD Library Issues
```
Error: unable to find the rrd_xport function
```
**Solution**: Install RRD development packages
```bash
# Ubuntu/Debian
sudo apt-get install librrd-dev

# CentOS/RHEL
sudo yum install rrdtool-devel
```

### Common Runtime Issues

#### 1. Module Not Loading
**Check**: Nagios/Icinga logs for error messages
**Solutions**:
- Verify file path in broker_module directive
- Check file permissions
- Ensure event_broker_options is set correctly

#### 2. Socket Permission Errors
**Symptoms**: Cannot connect to socket
**Solutions**:
```bash
# Check socket permissions
ls -la /tmp/livestatus

# Fix permissions
sudo chmod 666 /tmp/livestatus
# or add user to appropriate group
sudo usermod -a -G nagios www-data
```

#### 3. Connection Timeouts
**Symptoms**: Connections hang or timeout
**Solutions**:
- Check firewall settings (for TCP sockets)
- Verify Nagios/Icinga is running
- Check system resource usage
- Review debug logs with `debug=1` option

### Debug Mode

Enable debug logging for troubleshooting:

```
broker_module=/usr/local/lib/mk-livestatus/livestatus.o /tmp/livestatus debug=2
```

Debug levels:
- `0`: No debug output (default)
- `1`: Informational messages
- `2`: Detailed debug information

## Performance Optimization

### Socket Configuration
- **Unix sockets**: Better performance than TCP
- **Socket location**: Use RAM disk for high-frequency access
  ```bash
  # Create tmpfs mount
  sudo mount -t tmpfs tmpfs /var/lib/livestatus
  broker_module=.../livestatus.o /var/lib/livestatus/livestatus
  ```

### Build Optimizations
```bash
# Optimized build
./configure --with-nagios4 --with-re2 CXXFLAGS="-O3 -march=native"
make -j$(nproc)
```

### Memory Settings
```
# Increase cached messages for busy systems
broker_module=.../livestatus.o /tmp/livestatus max_cached_messages=10000
```

## Next Steps

After successful installation:

1. **Read the main README.md** for usage examples
2. **Check EXAMPLES.md** for practical use cases
3. **Review API_REFERENCE.md** for detailed API documentation
4. **Set up monitoring dashboards** using the APIs
5. **Configure performance monitoring** with appropriate tools

## Getting Help

If you encounter issues:

1. **Check logs** in your monitoring system
2. **Review documentation** in this repository
3. **Search existing issues** on GitHub
4. **Create detailed bug reports** with:
   - Operating system and version
   - Nagios/Icinga version
   - Build configuration used
   - Complete error messages
   - Relevant log entries