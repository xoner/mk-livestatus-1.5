# Contributing to MK Livestatus

We welcome contributions to MK Livestatus! This document provides guidelines for contributing to the project.

## Table of Contents

1. [Getting Started](#getting-started)
2. [Development Environment](#development-environment)
3. [Coding Standards](#coding-standards)
4. [Submitting Changes](#submitting-changes)
5. [Bug Reports](#bug-reports)
6. [Feature Requests](#feature-requests)
7. [Testing](#testing)
8. [Documentation](#documentation)

## Getting Started

### Prerequisites

Before contributing, please ensure you have:

- Familiarity with C++17 and modern C++ practices
- Understanding of Nagios/Icinga monitoring systems
- Knowledge of network programming and socket interfaces
- Experience with autotools build system

### Areas for Contribution

We particularly welcome contributions in:

- **Core Engine**: Query processing, performance optimizations
- **API Bindings**: Python, Perl, C++ client libraries
- **Documentation**: Examples, tutorials, API documentation
- **Testing**: Unit tests, integration tests, performance tests
- **Platform Support**: Additional operating systems and architectures
- **Features**: New query capabilities, output formats, data sources

## Development Environment

### Setting Up Development Environment

1. **Clone the repository**:
   ```bash
   git clone https://github.com/xoner/mk-livestatus-1.5.git
   cd mk-livestatus-1.5
   ```

2. **Install dependencies** (see INSTALL.md for detailed instructions):
   ```bash
   # Ubuntu/Debian
   sudo apt-get install build-essential autoconf automake libtool \
       libboost-dev librrd-dev libre2-dev
   
   # CentOS/RHEL
   sudo yum install gcc-c++ autoconf automake libtool \
       boost-devel rrdtool-devel re2-devel
   ```

3. **Configure for development**:
   ```bash
   ./configure --with-nagios4 --with-re2 --enable-dependency-tracking
   ```

4. **Build**:
   ```bash
   make -j$(nproc)
   ```

5. **Test the build**:
   ```bash
   # Check if library builds correctly
   ls -la src/.libs/livestatus.so
   
   # Test unixcat utility
   src/unixcat --help
   ```

### Development Build Options

For development builds, consider these configure options:

```bash
# Debug build with all warnings
./configure --with-nagios4 --with-re2 \
    CXXFLAGS="-g -O0 -Wall -Wextra -Werror" \
    --enable-dependency-tracking

# Release build with optimizations
./configure --with-nagios4 --with-re2 \
    CXXFLAGS="-O3 -DNDEBUG -march=native" \
    --disable-dependency-tracking
```

### Testing Environment

Set up a test Nagios instance for development:

```bash
# Using OMD (recommended)
# Download from https://omdistro.org/
sudo dpkg -i omd-*.deb
sudo omd create testsite
sudo omd start testsite

# Configure test environment
echo "broker_module=$PWD/src/.libs/livestatus.so /opt/omd/sites/testsite/tmp/livestatus" \
    >> /opt/omd/sites/testsite/etc/nagios/nagios.cfg
```

## Coding Standards

### C++ Coding Standards

#### Code Style
- **C++17 standard**: Use modern C++ features where appropriate
- **Indentation**: 4 spaces, no tabs
- **Line length**: Maximum 100 characters
- **Braces**: Opening brace on same line for functions, control structures

#### Example Code Style
```cpp
class QueryProcessor {
public:
    explicit QueryProcessor(const std::string& socket_path)
        : socket_path_(socket_path), connected_(false) {}

    bool connect() {
        if (socket_path_.empty()) {
            return false;
        }
        
        // Implementation here...
        return true;
    }

private:
    std::string socket_path_;
    bool connected_;
};
```

#### Naming Conventions
- **Classes**: PascalCase (`QueryProcessor`, `ColumnFilter`)
- **Functions/Methods**: camelCase (`processQuery`, `getColumnValue`)
- **Variables**: snake_case (`socket_path`, `column_name`)
- **Constants**: UPPER_SNAKE_CASE (`MAX_QUERY_LENGTH`)
- **Private members**: trailing underscore (`socket_path_`, `connected_`)

#### Memory Management
- Use RAII principles
- Prefer smart pointers over raw pointers
- Use `std::unique_ptr` for single ownership
- Use `std::shared_ptr` for shared ownership
- Always delete allocated memory in C-style APIs

```cpp
// Good
auto query = std::make_unique<Query>(query_string);

// Avoid
Query* query = new Query(query_string);
// ... (easy to forget delete)
```

#### Error Handling
- Use exceptions for error conditions
- Provide meaningful error messages
- Log errors appropriately

```cpp
void QueryProcessor::processQuery(const std::string& query) {
    if (query.empty()) {
        throw std::invalid_argument("Query cannot be empty");
    }
    
    try {
        // Process query...
    } catch (const std::exception& e) {
        Error(logger_) << "Query processing failed: " << e.what();
        throw;
    }
}
```

### API Design Guidelines

#### Consistency
- Maintain consistent naming across all APIs
- Use similar patterns for similar operations
- Provide analogous functionality in all language bindings

#### Error Handling
- Always provide meaningful error messages
- Use appropriate exception types for different languages
- Document all possible error conditions

#### Documentation
- Document all public APIs
- Provide usage examples
- Include performance considerations

## Submitting Changes

### Git Workflow

1. **Create a feature branch**:
   ```bash
   git checkout -b feature/new-query-operator
   ```

2. **Make your changes**:
   - Follow coding standards
   - Add appropriate tests
   - Update documentation

3. **Commit changes**:
   ```bash
   git add .
   git commit -m "Add new query operator for regex filtering
   
   - Implements ~= operator for case-insensitive regex matching
   - Adds unit tests for new operator
   - Updates API documentation"
   ```

4. **Test your changes**:
   ```bash
   make clean
   make -j$(nproc)
   # Run any available tests
   ```

5. **Push and create pull request**:
   ```bash
   git push origin feature/new-query-operator
   ```

### Commit Message Guidelines

- Use present tense ("Add feature" not "Added feature")
- Use imperative mood ("Move cursor to..." not "Moves cursor to...")
- Limit first line to 72 characters
- Reference issues and pull requests when appropriate
- Include detailed description for complex changes

#### Commit Message Format
```
Short (50 chars or less) summary of changes

More detailed explanatory text, if necessary. Wrap it to about 72
characters or so. In some contexts, the first line is treated as the
subject of an email and the rest of the text as the body.

- Bullet points are okay too
- Use a hanging indent for longer bullet points

Fixes #123
Closes #456
```

### Pull Request Guidelines

#### Before Submitting
- [ ] Code follows project coding standards
- [ ] All tests pass
- [ ] Documentation is updated
- [ ] Commit messages are clear and descriptive
- [ ] No merge conflicts with main branch

#### Pull Request Description
Include:
- **Summary**: What does this change do?
- **Motivation**: Why is this change needed?
- **Testing**: How was this tested?
- **Impact**: Any breaking changes or performance implications?
- **Related Issues**: Link to relevant issues

#### Example Pull Request Template
```markdown
## Summary
Adds support for case-insensitive regex filtering with new ~= operator.

## Motivation
Users frequently need to perform case-insensitive searches on log messages
and service output. The existing ~ operator is case-sensitive, making it
difficult to find entries regardless of case.

## Changes
- Added `~=` operator to filter engine
- Updated query parser to recognize new operator
- Added unit tests for case-insensitive matching
- Updated API documentation and examples

## Testing
- All existing tests pass
- Added 15 new unit tests covering edge cases
- Tested with real Nagios data

## Breaking Changes
None - this is a backward-compatible addition.

Fixes #234
```

## Bug Reports

### Before Reporting a Bug

1. **Search existing issues** to avoid duplicates
2. **Update to latest version** if possible
3. **Test with minimal configuration** to isolate the issue
4. **Gather relevant information** (see template below)

### Bug Report Template

```markdown
## Bug Description
A clear and concise description of what the bug is.

## Steps to Reproduce
1. Configure Livestatus with...
2. Execute query...
3. Observe error...

## Expected Behavior
What you expected to happen.

## Actual Behavior
What actually happened.

## Environment
- OS: Ubuntu 20.04 LTS
- Nagios Version: 4.4.6
- Livestatus Version: 1.5.0
- Compiler: GCC 9.4.0
- Configure options: --with-nagios4 --with-re2

## Query/Configuration
```
GET services
Filter: name ~ test.*
Columns: host_name description state
```

## Log Output
Include relevant log entries from Nagios and Livestatus.

## Additional Context
Any other information that might be helpful.
```

### Critical Bug Reports

For security issues or critical bugs:

1. **Do not create public issue** for security vulnerabilities
2. **Email directly** to maintainers
3. **Include "SECURITY" or "CRITICAL"** in subject line
4. **Provide detailed reproduction steps**

## Feature Requests

### Before Requesting

1. **Check existing issues** for similar requests
2. **Consider scope** - is this appropriate for core Livestatus?
3. **Think about alternatives** - can this be achieved differently?

### Feature Request Template

```markdown
## Feature Summary
One-line summary of the requested feature.

## Motivation
Why is this feature needed? What problem does it solve?

## Detailed Description
Detailed description of the proposed feature.

## Proposed API
How would this feature be exposed to users?

## Example Usage
```python
# Example of how the feature would be used
conn.query_table("GET services\nNewFeature: example")
```

## Alternatives Considered
What other approaches did you consider?

## Implementation Ideas
Any thoughts on how this could be implemented?
```

## Testing

### Types of Tests

#### Unit Tests
- Test individual components in isolation
- Focus on edge cases and error conditions
- Mock external dependencies

#### Integration Tests
- Test interaction between components
- Use real Nagios/Icinga instances when possible
- Verify end-to-end functionality

#### Performance Tests
- Measure query response times
- Test with large datasets
- Profile memory usage

### Writing Tests

#### C++ Unit Tests
```cpp
#include <gtest/gtest.h>
#include "QueryProcessor.h"

class QueryProcessorTest : public ::testing::Test {
protected:
    void SetUp() override {
        processor_ = std::make_unique<QueryProcessor>("/tmp/test_socket");
    }
    
    std::unique_ptr<QueryProcessor> processor_;
};

TEST_F(QueryProcessorTest, ProcessValidQuery) {
    std::string query = "GET hosts\nColumns: name state";
    EXPECT_NO_THROW(processor_->processQuery(query));
}

TEST_F(QueryProcessorTest, ProcessInvalidQuery) {
    std::string query = "INVALID QUERY";
    EXPECT_THROW(processor_->processQuery(query), std::invalid_argument);
}
```

#### Python Tests
```python
import unittest
import livestatus

class LivestatusTest(unittest.TestCase):
    def setUp(self):
        self.conn = livestatus.SingleSiteConnection("unix:/tmp/test_livestatus")
    
    def test_basic_query(self):
        # Test basic functionality
        result = self.conn.query_table("GET status")
        self.assertIsInstance(result, list)
    
    def test_invalid_table(self):
        # Test error handling
        with self.assertRaises(livestatus.MKLivestatusException):
            self.conn.query_table("GET invalid_table")

if __name__ == '__main__':
    unittest.main()
```

### Running Tests

```bash
# Run C++ tests (if available)
make check

# Run Python tests
cd api/python
python -m pytest tests/

# Run Perl tests
cd api/perl
perl Makefile.PL
make test
```

## Documentation

### Documentation Standards

#### Code Documentation
- Document all public APIs
- Include usage examples
- Explain complex algorithms
- Document performance characteristics

#### User Documentation
- Provide clear installation instructions
- Include working examples
- Cover common use cases
- Document troubleshooting steps

#### API Documentation
- Document all parameters and return values
- Include error conditions
- Provide code examples
- Keep examples up to date

### Documentation Format

Use clear, concise language:

```cpp
/**
 * Processes a Livestatus query and returns results.
 * 
 * @param query The LQL query string to execute
 * @param timeout Query timeout in seconds (0 = no timeout)
 * @return Vector of result rows, empty if no results
 * @throws QueryException if query is malformed
 * @throws SocketException if connection fails
 * 
 * Example:
 * ```cpp
 * auto results = processor.processQuery("GET hosts\nColumns: name state", 30);
 * for (const auto& row : results) {
 *     std::cout << "Host: " << row[0] << ", State: " << row[1] << std::endl;
 * }
 * ```
 */
std::vector<std::vector<std::string>> processQuery(
    const std::string& query, 
    int timeout = 0
);
```

## Code Review Process

### For Contributors

1. **Self-review** your changes before submitting
2. **Test thoroughly** with different configurations
3. **Update documentation** as needed
4. **Respond promptly** to review feedback

### Review Criteria

Reviewers will check for:

- **Correctness**: Does the code work as intended?
- **Performance**: Any performance regressions?
- **Security**: Any security implications?
- **Maintainability**: Is the code clean and well-structured?
- **Testing**: Are changes adequately tested?
- **Documentation**: Is documentation updated?

## Getting Help

### Communication Channels

- **GitHub Issues**: For bug reports and feature requests
- **GitHub Discussions**: For questions and general discussion
- **Pull Request Comments**: For code-specific discussions

### Questions

When asking questions:

1. **Search existing issues** first
2. **Provide context** about what you're trying to achieve
3. **Include relevant code** and configuration
4. **Be specific** about the problem

## License

By contributing to MK Livestatus, you agree that your contributions will be licensed under the same GPL-2.0 license that covers the project.

---

Thank you for contributing to MK Livestatus! Your efforts help improve monitoring capabilities for the entire community.