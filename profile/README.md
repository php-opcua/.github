<h1 align="center"><strong>PHP OPC UA</strong></h1>

<div align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="../assets/logo-dark.svg">
    <source media="(prefers-color-scheme: light)" srcset="../assets/logo-light.svg">
    <img alt="PHP OPC UA" src="../assets/logo-light.svg" width="435">
  </picture>
</div>

<p align="center">
  Pure PHP implementation of the OPC UA protocol stack.<br>
  Connect your PHP applications to PLCs, SCADA systems, sensors, historians, and IoT devices &mdash; no C/C++ extensions, no HTTP gateways, no middleware.
</p>

<p align="center">
  <a href="#packages">Packages</a> &bull;
  <a href="#quick-start">Quick Start</a> &bull;
  <a href="#framework-integrations">Integrations</a> &bull;
  <a href="#why-php-opcua">Why php-opcua</a>
</p>

---

## What is OPC UA?

**OPC UA** (Open Platform Communications Unified Architecture) is the industry standard protocol for secure, reliable communication in industrial automation and IoT. It's used worldwide to connect PLCs, SCADA systems, sensors, robots, CNC machines, building automation, and more.

**php-opcua** brings this protocol to the PHP ecosystem with a complete, native binary implementation.

---

## Packages

### Core

| Package | Description | |
|---------|-------------|---|
| [**opcua-client**](https://github.com/php-opcua/opcua-client) | The core OPC UA client library. Pure PHP binary protocol implementation with support for browse, read, write, method calls, subscriptions, history read, events, alarms, and type discovery. 6 security policies, 3 authentication modes, auto-retry, fluent builders, PSR-3/PSR-14/PSR-16 integration. | [![GitHub](https://img.shields.io/badge/GitHub-opcua--client-blue)](https://github.com/php-opcua/opcua-client) |
| [**opcua-cli**](https://github.com/php-opcua/opcua-cli) | Command-line tool for interacting with OPC UA servers. Browse, read, write, watch values in real-time, discover endpoints, manage certificates, export address spaces, and generate PHP code from NodeSet2.xml files. | [![GitHub](https://img.shields.io/badge/GitHub-opcua--cli-blue)](https://github.com/php-opcua/opcua-cli) |
| [**opcua-client-nodeset**](https://github.com/php-opcua/opcua-client-nodeset) | Pre-generated PHP types from 51 OPC Foundation companion specifications. 338 enums, 191 typed DTOs, 191 binary codecs covering DI, Robotics, MachineTool, BACnet, MTConnect, ISA-95, PackML, PROFINET, and more. | [![GitHub](https://img.shields.io/badge/GitHub-opcua--client--nodeset-blue)](https://github.com/php-opcua/opcua-client-nodeset) |
| [**opcua-test-suite**](https://github.com/php-opcua/opcua-test-suite) | Docker-based OPC UA test infrastructure. 8 pre-configured servers covering all security policies, authentication methods, and ~270 test nodes. Includes a GitHub Action for CI integration. | [![GitHub](https://img.shields.io/badge/GitHub-opcua--test--suite-blue)](https://github.com/php-opcua/opcua-test-suite) |

### Session Management & Framework Integrations

| Package | Description | Status |
|---------|-------------|--------|
| [**opcua-session-manager**](https://github.com/php-opcua/opcua-session-manager) | ReactPHP daemon that keeps OPC UA sessions alive across PHP requests via Unix socket IPC. Eliminates the per-request connection overhead (~150ms &rarr; ~5ms). | In development |
| [**laravel-opcua**](https://github.com/php-opcua/laravel-opcua) | Laravel service provider, facade, named connections, Artisan commands, and automatic PSR-3/PSR-16 injection. | In development |
| **symfony-opcua** | Symfony bundle with equivalent integration for the Symfony ecosystem. | Planned |

---

## Quick Start

### Installation

```bash
composer require php-opcua/opcua-client
```

**Requirements:** PHP 8.2+ and `ext-openssl`. No other extensions required.

### Connect and Read

```php
use PhpOpcua\Client\ClientBuilder;

$client = ClientBuilder::create()
    ->connect('opc.tcp://localhost:4840');

// Read server status
$status = $client->read('i=2259');
echo $status->getValue(); // 0 = Running

$client->disconnect();
```

### Browse the Address Space

```php
$refs = $client->browse('i=85'); // Objects folder

foreach ($refs as $ref) {
    echo "{$ref->displayName} ({$ref->nodeId})\n";
}
```

### Read Multiple Values

```php
$results = $client->readMulti()
    ->node('i=2259')->value()
    ->node('ns=2;i=1001')->displayName()
    ->node('ns=2;s=Temperature')->value()
    ->execute();
```

### Write to a Device

```php
use PhpOpcua\Client\Types\BuiltinType;

$client->write('ns=2;i=1001', 42);                      // auto-detect type
$client->write('ns=2;i=1001', 42, BuiltinType::Int32);  // explicit type
```

### Subscribe to Data Changes

```php
$sub = $client->createSubscription(publishingInterval: 500.0);

$client->createMonitoredItems($sub->subscriptionId, [
    ['nodeId' => 'ns=2;i=1001'],
]);

$response = $client->publish();
foreach ($response->notifications as $notif) {
    echo $notif['dataValue']->getValue() . "\n";
}
```

### Secure Connection

```php
use PhpOpcua\Client\Security\SecurityPolicy;
use PhpOpcua\Client\Security\SecurityMode;

$client = ClientBuilder::create()
    ->setSecurityPolicy(SecurityPolicy::Basic256Sha256)
    ->setSecurityMode(SecurityMode::SignAndEncrypt)
    ->setClientCertificate('/certs/client.pem', '/certs/client.key', '/certs/ca.pem')
    ->setUserCredentials('operator', 'secret')
    ->connect('opc.tcp://192.168.1.100:4840');
```

### History Read

```php
$values = $client->historyReadRaw(
    'ns=2;i=1001',
    startTime: new DateTimeImmutable('-1 hour'),
    endTime: new DateTimeImmutable(),
);

foreach ($values as $dv) {
    echo "[{$dv->sourceTimestamp->format('H:i:s')}] {$dv->getValue()}\n";
}
```

---

## CLI Tool

Install the CLI globally or per-project:

```bash
composer require php-opcua/opcua-cli
```

```bash
# Browse a server
opcua-cli browse opc.tcp://localhost:4840 /Objects --recursive

# Read a value
opcua-cli read opc.tcp://localhost:4840 ns=2;i=1001

# Watch values in real-time
opcua-cli watch opc.tcp://localhost:4840 ns=2;i=1001

# Write a value
opcua-cli write opc.tcp://localhost:4840 ns=2;i=1001 42 --type=Int32

# Discover endpoints and security policies
opcua-cli endpoints opc.tcp://localhost:4840

# Export address space to NodeSet2.xml
opcua-cli dump:nodeset opc.tcp://localhost:4840 --output=MyPLC.NodeSet2.xml

# Generate PHP types from NodeSet2.xml
opcua-cli generate:nodeset MyPLC.NodeSet2.xml --output=src/Generated --namespace=App\\OpcUa

# Manage server certificate trust
opcua-cli trust opc.tcp://localhost:4840
opcua-cli trust:list
opcua-cli trust:remove ab:cd:12:34:...
```

All commands support `--json` output, full security configuration (`-s`, `-m`, `--cert`, `--key`), and debug logging (`--debug`, `--debug-stderr`, `--debug-file`).

---

## Pre-generated Companion Specification Types

```bash
composer require php-opcua/opcua-client-nodeset
```

51 OPC Foundation companion specifications, ready to use as typed PHP classes:

```php
use PhpOpcua\Client\ClientBuilder;
use PhpOpcua\Nodeset\Robotics\RoboticsRegistrar;
use PhpOpcua\Nodeset\Robotics\Enums\OperationalModeEnumeration;

$client = ClientBuilder::create()
    ->loadGeneratedTypes(new RoboticsRegistrar()) // auto-loads dependencies
    ->connect('opc.tcp://192.168.1.100:4840');

$mode = $client->read('ns=2;s=OperationalMode')->getValue();
// Returns: OperationalModeEnumeration::MANUAL_HIGH_SPEED (PHP enum, not raw int)
```

**Supported specifications include:** DI (Device Integration), Robotics, MachineTool, BACnet, MTConnect, AutoID, ISA-95, PackML, MachineVision, PROFINET, I4AAS, LADS, CNC, Safety, Scales, Woodworking, and 35 more.

---

## Test Suite

A Docker-based test infrastructure with 8 OPC UA servers for integration testing:

| Server | Port | Purpose |
|--------|------|---------|
| No Security | 4840 | Basic connectivity, anonymous access |
| Username/Password | 4841 | Credential authentication |
| Certificate | 4842 | X.509 certificate authentication |
| All Security | 4843 | All policies, modes, and auth methods |
| Discovery | 4844 | OPC UA Discovery Server |
| Auto-Accept | 4845 | Auto-trust any client certificate |
| Sign Only | 4846 | Message signing without encryption |
| Legacy Security | 4847 | Deprecated policies (Basic128Rsa15, Basic256) |

Each server exposes ~270 nodes: scalars, arrays, matrices, methods, dynamic variables, events, alarms, historical data, structured objects, and access-controlled variables.

### Docker

```bash
docker compose up -d
```

### GitHub Actions

```yaml
- uses: php-opcua/opcua-test-suite@v1
  with:
    servers: 'no-security,userpass,certificate'
```

---

## Framework Integrations

### Laravel (in development)

```bash
composer require php-opcua/laravel-opcua
```

```php
use PhpOpcua\Laravel\Facades\Opcua;

$client = Opcua::connect();
$value = $client->read('i=2259');
```

- Named connections in `config/opcua.php` (like database connections)
- Automatic PSR-3 logging and PSR-16 caching via Laravel's services
- Artisan command to start the session manager daemon: `php artisan opcua:session`
- Transparent session persistence when the daemon is running

### Symfony (planned)

A Symfony bundle with equivalent DI integration is planned.

---

## Why php-opcua

- **Pure PHP** &mdash; Native binary protocol. No C extensions, no FFI, no HTTP gateways, no sidecars.
- **Zero runtime dependencies** &mdash; Only `ext-openssl`. PSR-3, PSR-14, and PSR-16 are optional.
- **Full protocol support** &mdash; Browse, read, write, method calls, subscriptions, events, alarms, history read, type discovery, path resolution.
- **Industrial-grade security** &mdash; 6 security policies (None through Aes256Sha256RsaPss), 3 authentication modes, certificate trust management with TOFU support.
- **Developer experience** &mdash; Fluent builders, auto-batching, auto-retry, `readonly` DTOs, typed enums, MockClient for testing.
- **Cross-platform** &mdash; Linux, macOS, Windows. PHP 8.2 through 8.5.
- **Session persistence** &mdash; Optional ReactPHP daemon keeps connections alive across PHP requests (~150ms &rarr; ~5ms).
- **51 companion specs** &mdash; Pre-generated types for DI, Robotics, MachineTool, BACnet, and more.
- **Tested** &mdash; 1000+ tests, 8 Docker-based test servers, CI across PHP 8.2&ndash;8.5.

---

## License

Each package is released under its own license. See the individual repository for details.
