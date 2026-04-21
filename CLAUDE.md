# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**openTCS** (open Transportation Control System) is a Java 21 framework for controlling fleets of automated guided vehicles (AGVs) and mobile robots. It is not a standalone product but a platform requiring custom vehicle driver integration. It provides core algorithms for routing, dispatching, and scheduling.

## Build Commands

```bash
# Full build (compile + test + quality checks)
./gradlew build

# Build without tests
./gradlew assemble

# Clean build
./gradlew clean build

# Install all application distributions locally
./gradlew installDist

# Create binary distribution ZIP
./gradlew distZip
```

## Testing

```bash
# Run all tests
./gradlew test

# Run tests for a specific module
./gradlew :opentcs-kernel:test

# Run a specific test class
./gradlew :opentcs-kernel:test --tests "org.opentcs.kernel.StandardKernelTest"

# Run a specific test method
./gradlew :opentcs-kernel:test --tests "org.opentcs.kernel.StandardKernelTest.methodName"

# Run tests matching a pattern
./gradlew :opentcs-kernel:test --tests "*.workingset.*Test"

# Generate aggregated JaCoCo coverage report
./gradlew jacocoAggregatedReport
# Output: build/reports/jacoco/jacocoAggregatedReport/html/index.html
```

**Compiler flags**: `-Werror` (warnings as errors), `-Xlint:all -Xlint:-serial` — all lint warnings must be resolved.

**Javadoc**: Strict enforcement (`-Xdoclint:all,-missing`) for public APIs.

## Running Applications

```bash
./gradlew :opentcs-kernel:run
./gradlew :opentcs-operationsdesk:run
./gradlew :opentcs-modeleditor:run
./gradlew :opentcs-kernelcontrolcenter:run
```

## Architecture

### Module Structure

The project is a Gradle multi-module build (~19 modules):

| Module | Purpose |
|---|---|
| `opentcs-api-base` | Core data models, service interfaces, component contracts |
| `opentcs-api-injection` | Dependency injection configuration interfaces |
| `opentcs-common` | Shared utilities |
| `opentcs-kernel` | Central control system; entry point `RunKernel` |
| `opentcs-strategies-default` | Default routing, dispatching, and scheduling implementations |
| `opentcs-kernel-extension-http-services` | REST/HTTP API for remote access |
| `opentcs-kernel-extension-rmi-services` | RMI services for remote access |
| `opentcs-commadapter-loopback` | Loopback (test/demo) vehicle driver |
| `opentcs-kernelcontrolcenter` | Kernel monitoring GUI |
| `opentcs-modeleditor` | Plant topology editor GUI |
| `opentcs-operationsdesk` | Operator/dispatch GUI |
| `opentcs-plantoverview-*` | Shared UI components and themes |
| `opentcs-impl-configuration-gestalt` | Configuration using Gestalt library |

### Kernel Architecture

`StandardKernel` is the central hub with a state machine:
- **MODELLING** → topology/model creation
- **OPERATING** → normal operation; accepts and dispatches transport orders
- **SHUTDOWN** → clean shutdown

Components are wired via **Google Guice** dependency injection. Plugins (strategies, vehicle adapters, extensions) are discovered via `ServiceLoader`. The event bus (`@ApplicationEventBus`) decouples components.

**Kernel services** (exposed to GUIs and external clients):
- `PlantModelService` — topology management
- `TransportOrderService` — order lifecycle
- `VehicleService` — vehicle monitoring/control
- `NotificationService` — system event broadcasting
- `DispatcherService` — dispatcher state access
- `PeripheralJobService` — peripheral device tasks

**Pluggable strategies** (swap implementations via Guice modules):
- `DefaultDispatcher` — which vehicle gets which order
- `DefaultRouter` — path finding (uses jGraphT)
- `DefaultScheduler` — resource allocation/conflict resolution
- `DefaultPeripheralJobDispatcher`

### Data Models

All model objects extend `TCSObject`, are **immutable**, and live in `opentcs-api-base` under `org.opentcs.data`:
- `Vehicle` — AGV/robot with state, energy level, claimed resources
- `Point` — network node with (x, y) coordinates
- `Path` — directional edge between two Points
- `Location` — parking/operation spot linked to Points
- `Block` — resource group for conflict avoidance
- `TransportOrder` — task with pickup/delivery drive orders
- `PeripheralJob` — task for peripheral devices

### Entry Points

Each GUI application has a `RunXxx` class in `src/guiceConfig/java/`:
- `org.opentcs.kernel.RunKernel`
- `org.opentcs.operationsdesk.RunOperationsDesk`
- `org.opentcs.modeleditor.RunModelEditor`
- `org.opentcs.kernelcontrolcenter.RunKernelControlCenter`

These use `Guice.createInjector(...)` to bootstrap the application.

### Configuration

Gestalt-based configuration, layered:
1. `config/opentcs-kernel-defaults-baseline.properties` — shipped defaults
2. `config/opentcs-kernel-defaults-custom.properties` — operator overrides
3. `config/opentcs-kernel.properties` — runtime config

Key system properties: `opentcs.base`, `opentcs.home`, `opentcs.configuration.provider`.

## Coding Conventions

- **Indentation**: 2 spaces (enforced by `.editorconfig`)
- **Line length**: ~100 characters
- **Formatter**: Eclipse JDT 4.38 (`config/eclipse-formatter-preferences.xml`)
- **Encoding**: UTF-8; Unix LF line endings

**Input validation** (required for all public/protected methods):
- `Objects.requireNonNull()` with `@Nonnull` for null checks
- `org.opentcs.util.Assertions.checkInRange()` for numeric range validation
- `org.opentcs.util.Assertions.checkArgument()` for value constraints

**Testing conventions**:
- JUnit Jupiter 5; test methods omit `public` modifier
- Prefer `assertThat()` (AssertJ) over `assertTrue()`
- Tests for new/modified non-trivial logic are required
- GUI tests run with `java.awt.headless=true`

## Key Dependency Versions

See `gradle/libs.versions.toml` for the full version catalog. Notable versions:
- Java 21
- Google Guice 7.0.0
- Javalin 7.1.0 (HTTP)
- Jackson 2.21.1 (JSON)
- jGraphT 1.5.2 (graph algorithms)
- JUnit Jupiter 6.0.3
- Mockito 5.23.0
- AssertJ 3.27.7

