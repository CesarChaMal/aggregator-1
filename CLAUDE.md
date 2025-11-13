# CLAUDE.md - AI Assistant Guide for Aggregator Project

## Project Overview

**Aggregator** is a Java-based financial instrument processing system designed to efficiently calculate statistical metrics on large-scale time series data. The project demonstrates handling of gigabyte-scale input files with thousands of financial instruments while maintaining memory efficiency.

### Core Purpose
Process financial instrument time series data from CSV files and compute different statistical calculations per instrument:
- INSTRUMENT1: Mean (average) of all values
- INSTRUMENT2: Mean for November 2014 only
- INSTRUMENT3: On-the-fly statistical calculation (variance)
- INSTRUMENT4+: Sum of newest 10 elements for all other instruments

### Key Constraints
- **Date**: Current date assumed as 19-Dec-2014 (no data expected after this)
- **Business Days Only**: Validates dates are Monday-Friday
- **Scalability**: Must handle multi-gigabyte files with 10,000+ instruments
- **Memory Efficiency**: Streaming approach to avoid OutOfMemory errors
- **Performance**: Calculate metrics as quickly as possible

## Technology Stack

- **Language**: Java 1.7
- **Build Tool**: Maven 3.x
- **Testing**: JUnit 4.7
- **Logging**: SLF4J 1.6.6 + Log4j 1.2.15
- **Architecture**: Modular strategy pattern with streaming file processing

## Directory Structure

```
aggregator-1/
├── src/
│   ├── main/
│   │   └── java/
│   │       └── app/
│   │           └── aggregator/
│   │               ├── core/           # Core calculation engine
│   │               │   ├── Calculator.java
│   │               │   ├── EngineModule.java
│   │               │   └── modul/      # Module implementations
│   │               │       ├── AverageModule.java
│   │               │       ├── AverageMonthModule.java
│   │               │       ├── AverageNewstInstrumentsModule.java
│   │               │       └── OnFlyModule.java
│   │               ├── model/          # Domain models
│   │               │   └── Instrument.java
│   │               ├── util/           # Utilities
│   │               │   └── DefinerInstrument.java
│   │               └── launch/         # Application entry point
│   │                   └── Launcher.java
│   └── test/
│       ├── java/
│       │   └── app/aggregator/test/
│       │       └── ProcessingTest.java
│       └── resources/
│           └── input.txt               # Test data
├── pom.xml                             # Maven configuration
├── example_input.txt                   # Example input data
├── aggregator-jar-with-dependencies.jar
├── .gitignore
└── README.md
```

## Architecture & Design Patterns

### 1. Strategy Pattern (EngineModule)
The core architecture uses the Strategy pattern to allow different calculation strategies for different instruments:

```java
public abstract class EngineModule {
    protected abstract Float calculate();
}
```

Each instrument type gets its own module implementation:
- `AverageModule` - Simple mean calculation for INSTRUMENT1
- `AverageMonthModule` - Filtered mean for November 2014 for INSTRUMENT2
- `OnFlyModule` - Real-time variance calculation for INSTRUMENT3
- `AverageNewstInstrumentsModule` - Sum of newest 10 elements (default)

### 2. Streaming Architecture
The Calculator processes files using BufferedReader line-by-line to handle large files without loading everything into memory.

Key optimization in `Calculator.java:148-156`:
```java
private static void addAndSortToLastTenInstruments(Instrument instrument) {
    String name = instrument.getName();
    if (INSTRUMENTS.get(name).size() == NEWST) {
        // Replace oldest, then sort
        INSTRUMENTS.get(name).set(INSTRUMENTS.get(name).size() - 1, instrument);
        Collections.sort(INSTRUMENTS.get(name));
    } else {
        INSTRUMENTS.get(name).add(instrument);
    }
}
```

This maintains a sliding window of the 10 newest elements per instrument.

### 3. Memory Management Strategy
- INSTRUMENT1 & INSTRUMENT2: Store all values (needed for mean calculations)
- INSTRUMENT3: Store all values but calculate on-the-fly
- Other instruments: Store only newest 10 elements (sorted by date descending)

## Key Components

### 1. Launcher (`app.aggregator.launch.Launcher`)
**Location**: `src/main/java/app/aggregator/launch/Launcher.java`

Application entry point. Responsibilities:
- Parse command-line arguments (`file=<path>`)
- Initialize Calculator
- Register calculation modules
- Trigger processing

### 2. Calculator (`app.aggregator.core.Calculator`)
**Location**: `src/main/java/app/aggregator/core/Calculator.java`

Core orchestration class. Responsibilities:
- Manage instrument collections (HashMap of Lists)
- Manage calculation modules (HashMap of EngineModules)
- Stream-process input file line by line
- Route instruments to appropriate storage strategy
- Execute all registered modules

**Important Constants**:
- `INSTRUMENTS_COUNT = 10000` - Pre-allocates space for 10K instruments
- `NEWST = 11` - Actually stores 11 elements (processes 10 newest)

### 3. EngineModule (`app.aggregator.core.EngineModule`)
**Location**: `src/main/java/app/aggregator/core/EngineModule.java`

Abstract base class for calculation strategies. Subclasses must implement:
```java
protected abstract Float calculate();
```

### 4. Instrument (`app.aggregator.model.Instrument`)
**Location**: `src/main/java/app/aggregator/model/Instrument.java`

Domain model representing a single data point:
- `name` (String) - Instrument identifier
- `price` (float) - Value at that point in time
- `date` (Date) - Timestamp

Implements `Comparable<Instrument>` to sort by date descending (newest first).

### 5. DefinerInstrument (`app.aggregator.util.DefinerInstrument`)
**Location**: `src/main/java/app/aggregator/util/DefinerInstrument.java`

Parser and validator for CSV input lines. Responsibilities:
- Parse CSV format: `<INSTRUMENT_NAME>,<DATE>,<VALUE>`
- Parse date format: `dd-MMM-yyyy` (e.g., "12-Mar-2015")
- Validate business days (exclude Saturday/Sunday)
- Return null for invalid/weekend dates

## Input File Format

```
INSTRUMENT_NAME,DATE,VALUE
```

**Example**:
```
INSTRUMENT1,12-Mar-2015,12.21
INSTRUMENT2,15-Nov-2014,8.50
INSTRUMENT3,18-Dec-2014,15.75
```

**Date Format**: `dd-MMM-yyyy` (e.g., 12-Mar-2015)

**Validation Rules**:
- Must be valid date
- Must be Monday-Friday (business days only)
- Weekend dates are filtered out

## Development Workflows

### Building the Project

```bash
# Clean and compile
mvn clean compile

# Run tests
mvn test

# Build executable JAR with dependencies
mvn clean test assembly:assembly
```

**Output**: `target/aggregator-jar-with-dependencies.jar`

### Running the Application

```bash
# Using the pre-built JAR
java -jar aggregator-jar-with-dependencies.jar file=/path/to/input.txt

# Using Maven exec plugin
mvn exec:java -Dexec.mainClass="app.aggregator.launch.Launcher" -Dexec.args="file=/path/to/input.txt"
```

**Required Argument**: `file=<path>` - Path to input CSV file

### Running Tests

```bash
# Run all tests
mvn test

# Run specific test
mvn test -Dtest=ProcessingTest

# Run tests with verbose output
mvn test -X
```

### Git Workflow

**Current Branch**: `claude/claude-md-mhy2vustsv0b2jbh-01YFHZ1r9wAmNaw5XKY6qVnw`

**Standard Operations**:
```bash
# Check status
git status

# Commit changes
git add .
git commit -m "Description of changes"

# Push to remote (with retry logic for network issues)
git push -u origin claude/claude-md-mhy2vustsv0b2jbh-01YFHZ1r9wAmNaw5XKY6qVnw
```

**Important**: Branch names must start with `claude/` and match the session ID, or push will fail with 403.

## Code Conventions

### Naming Conventions
- **Classes**: PascalCase (e.g., `EngineModule`, `AverageModule`)
- **Methods**: camelCase (e.g., `calculate()`, `addModule()`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `INSTRUMENTS_COUNT`, `INSTRUMENT1`)
- **Variables**: camelCase (e.g., `inputPath`, `instrument`)

### Package Structure
- `app.aggregator.launch` - Application entry points
- `app.aggregator.core` - Core business logic
- `app.aggregator.core.modul` - Calculation module implementations
- `app.aggregator.model` - Domain models
- `app.aggregator.util` - Utility classes
- `app.aggregator.test` - Test classes

### Logging
Uses SLF4J with Log4j backend:
```java
private static final Logger LOGGER = LoggerFactory.getLogger(ClassName.class);

LOGGER.debug("Debug message");
LOGGER.info("Info message");
LOGGER.error("Error message", exception);
```

### Documentation
- JavaDoc comments for classes and public methods
- Author tags: `@author Ogarkov Sergey`
- Inline comments for complex logic

## Testing Strategy

### Unit Tests
Located in `src/test/java/app/aggregator/test/`

**ProcessingTest.java**:
- Tests `AverageModule` calculation
- Uses test resource file: `src/test/resources/input.txt`
- Validates parsing, filtering, and calculation accuracy

**Test Data**: `src/test/resources/input.txt`

### Testing Best Practices
1. Use JUnit assertions: `assertEquals()`, `assertTrue()`
2. Test files in `src/test/resources/` (accessible via `getResourceAsStream()`)
3. Test business logic separately from file I/O
4. Validate calculation accuracy with known expected values

## Performance Considerations

### Memory Optimization
1. **Streaming Processing**: Uses BufferedReader to read line-by-line
2. **Limited Storage**:
   - Only stores newest 10 elements for most instruments
   - Full storage only for INSTRUMENT1, INSTRUMENT2, INSTRUMENT3
3. **Pre-allocation**: Pre-allocates HashMap capacity for 10K instruments

### Scalability Notes
- Designed to handle multi-gigabyte files
- Supports up to 10,000 instruments (configurable via `INSTRUMENTS_COUNT`)
- Constant memory footprint regardless of file size for most instruments

## Common Tasks for AI Assistants

### 1. Adding a New Calculation Module

```java
// Create new class extending EngineModule
public class NewModule extends EngineModule {
    public NewModule() {
        super("INSTRUMENT_NAME");
    }

    @Override
    protected Float calculate() {
        // Implement calculation logic
        // Access data via getInstruments()
    }
}

// Register in Launcher.java
calculator.addModule(new NewModule());
```

### 2. Modifying Calculation Logic
- Locate appropriate module in `src/main/java/app/aggregator/core/modul/`
- Modify `calculate()` method
- Update corresponding tests
- Run `mvn test` to verify

### 3. Adding New Instrument Types
- Modify `INSTRUMENTS_COUNT` in `Calculator.java:46` if needed
- Add storage logic in `Calculator.add()` method
- Create new EngineModule implementation
- Register module in `Launcher.java`

### 4. Changing Input Format
- Modify `DefinerInstrument.defineOf()` parsing logic
- Update date format in `DefinerInstrument.getDate()`
- Update tests with new format examples

### 5. Adding Database Integration
**Note**: Original spec mentions database for INSTRUMENT_PRICE_MODIFIER table, but current implementation doesn't include it. To add:
1. Add H2/Derby dependency to `pom.xml`
2. Create DAO layer in new package `app.aggregator.dao`
3. Add multiplier lookup in `DefinerInstrument` or `Calculator`
4. Cache results for 5 seconds as per spec

## Important Implementation Details

### Date Handling
- Format: `dd-MMM-yyyy` with US locale
- Weekend filtering: Excludes Saturday (day 7) and Sunday (day 1)
- Calendar constants: Sunday=1, Monday=2, ..., Saturday=7

### Sorting
Instruments sorted by date **descending** (newest first):
```java
// In Instrument.java:48-54
return (this.date.getTime() < instrument.date.getTime() ? 1 : -1);
```

### Fluent API
Calculator uses method chaining:
```java
calculator
    .addModule(new AverageModule())
    .addModule(new OnFlyModule())
    .addModuleDefault()
    .calculate();
```

## File References

| File | Line | Description |
|------|------|-------------|
| `Launcher.java:43` | Main class instantiation | Creates Calculator with input path |
| `Calculator.java:116-128` | File processing | Main aggregation loop |
| `Calculator.java:148-156` | Sliding window | Maintains newest 10 elements |
| `DefinerInstrument.java:28-41` | CSV parsing | Parses and validates input lines |
| `Instrument.java:48-54` | Date comparison | Implements descending date sort |
| `EngineModule.java:37` | Calculate method | Abstract method for calculations |

## Known Issues & Limitations

1. **No Database Integration**: Original spec mentions INSTRUMENT_PRICE_MODIFIER table, but not implemented
2. **Hard-coded Instrument Count**: Limited to 10,000 instruments (configurable constant)
3. **Date Assumption**: Assumes current date is 19-Dec-2014
4. **Error Handling**: Limited error handling for malformed input
5. **Resource Management**: Some test code doesn't use try-with-resources

## AI Assistant Guidelines

### When Making Changes
1. **Read First**: Always read existing files before modifying
2. **Test After**: Run `mvn test` after any changes
3. **Maintain Patterns**: Follow existing architecture patterns (Strategy, Streaming)
4. **Memory Awareness**: Consider memory implications for large files
5. **Business Logic**: Respect business day validation and date constraints

### Code Quality
- Maintain existing code style and formatting
- Add appropriate logging statements
- Update tests when modifying business logic
- Keep methods focused and single-purpose
- Use meaningful variable names

### Git Practices
- Commit logical units of work
- Write clear commit messages
- Always push to the correct branch (starting with `claude/`)
- Include retry logic for network operations (up to 4 retries with exponential backoff)

### Documentation
- Update this CLAUDE.md when making architectural changes
- Add JavaDoc for new public methods
- Document complex algorithms inline
- Update README.md for user-facing changes

## Quick Reference Commands

```bash
# Build
mvn clean test assembly:assembly

# Run
java -jar aggregator-jar-with-dependencies.jar file=example_input.txt

# Test
mvn test

# Specific test
mvn test -Dtest=ProcessingTest

# Git operations
git status
git add .
git commit -m "message"
git push -u origin <branch-name>
```

## Contact & Metadata

**Original Author**: Ogarkov Sergey
**Java Version**: 1.7
**Maven Version**: 3.x+
**Created**: Interview/skills assessment project for financial instruments processing

---

*Last Updated*: 2025-11-13
*Document Version*: 1.0
*Maintained By*: AI Assistants working on this codebase
