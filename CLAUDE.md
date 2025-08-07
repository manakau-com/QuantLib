# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Building and Testing

### Build Commands

**CMake (Recommended):**
```bash
# Basic build
mkdir build && cd build
cmake ..
make -j$(nproc)

# Using presets (see CMakePresets.json)
cmake --preset linux-gcc-release
cmake --build --preset linux-gcc-release

# Debug build
cmake -DCMAKE_BUILD_TYPE=Debug ..

# With clang-tidy
cmake -DQL_USE_CLANG_TIDY=ON ..
```

**Running Tests:**
```bash
# Run all tests
./test-suite/quantlib-test-suite

# Run specific test suite
./test-suite/quantlib-test-suite --run_test=QuantLib/BondTests

# Run with detailed output
./test-suite/quantlib-test-suite --log_level=message

# Run benchmark
./test-suite/quantlib-benchmark
```

### Development Checks

```bash
# Check that all headers compile independently
./tools/check_all_headers.sh

# Check license headers
./tools/check_all_licenses.sh

# Apply clang-tidy fixes (use CMake preset)
cmake --preset linux-ci-build-with-clang-tidy
cd build/linux-ci-build-with-clang-tidy
cmake --build . -j1
```

## Architecture Overview

QuantLib is a comprehensive quantitative finance library with a modular architecture:

### Core Components

- **`/ql`** - Main library source
  - **Instruments** (`/instruments`) - Financial products (bonds, options, swaps)
  - **Pricing Engines** (`/pricingengines`) - Valuation algorithms
  - **Term Structures** (`/termstructures`) - Yield curves, volatility surfaces
  - **Models** (`/models`) - Mathematical models (Black-Scholes, Hull-White, etc.)
  - **Processes** (`/processes`) - Stochastic processes
  - **Math** (`/math`) - Numerical methods, optimizers, interpolations

### Design Patterns

1. **Handle-Body Idiom**: Most classes use shared pointers for reference semantics
   ```cpp
   Handle<YieldTermStructure> curve(ext::make_shared<FlatForward>(...));
   ```

2. **Observer Pattern**: Market data objects notify dependent calculations
   - Classes inherit from `Observer` and `Observable`
   - Automatic recalculation on market data changes

3. **LazyObject**: Caching pattern for expensive calculations
   - Results cached until inputs change
   - Used by `Instrument` and `PricingEngine`

4. **Visitor Pattern**: For instrument/engine dispatch

### Key Abstractions

- **Settings**: Global evaluation date and other settings
  ```cpp
  Settings::instance().evaluationDate() = Date(15, May, 2023);
  ```

- **Calendar**: Business day conventions for different markets
- **DayCounter**: Day count conventions (Actual360, Thirty360, etc.)
- **Schedule**: Payment/exercise date generation

## Common Tasks

### Creating an Instrument
```cpp
// Always start with evaluation date
Settings::instance().evaluationDate() = Date(15, May, 2023);

// Setup market data
Handle<YieldTermStructure> discountCurve(...);
Handle<BlackVolTermStructure> volatility(...);

// Create instrument
VanillaOption option(payoff, exercise);

// Attach pricing engine
option.setPricingEngine(ext::make_shared<AnalyticEuropeanEngine>(process));

// Calculate
Real npv = option.NPV();
```

### Working with Dates
```cpp
Date today = Date(15, May, 2023);
Calendar calendar = TARGET();  // European TARGET calendar
Date nextBusinessDay = calendar.advance(today, 1, Days);
```

## Important Configuration Options

**CMake Variables:**
- `QL_BUILD_EXAMPLES` - Build example programs
- `QL_BUILD_TEST_SUITE` - Build test suite
- `QL_ENABLE_OPENMP` - OpenMP parallelization
- `QL_ENABLE_THREAD_SAFE_OBSERVER_PATTERN` - Thread-safe observers
- `QL_USE_STD_CLASSES` - Use std:: instead of boost:: where possible
- `QL_USE_CLANG_TIDY` - Enable clang-tidy checks

## Development Guidelines

1. **Before Making Changes:**
   - Check existing patterns in similar files
   - Use library conventions (e.g., `ext::shared_ptr` not `std::shared_ptr` directly)
   - Follow existing naming conventions

2. **Testing:**
   - Add tests to `/test-suite` for new functionality
   - Run the full test suite before submitting changes
   - Use existing test files as examples

3. **Dependencies:**
   - Boost is required (minimum 1.58.0)
   - C++17 or higher required
   - Check CMakeLists.txt before assuming any library is available

4. **Performance Considerations:**
   - Use `LazyObject` pattern for expensive calculations
   - Consider caching in pricing engines
   - Be aware of the observer notification overhead

## File Organization

- Implementation files (.cpp) go in `/ql/[module]/`
- Headers (.hpp) mirror the source structure
- Tests go in `/test-suite/` with descriptive names
- Examples go in `/Examples/`

## Workflow

1. Create feature branch from master
2. Make changes following existing patterns
3. Run tests locally
4. Submit pull request (automated CI will run)
5. Address review feedback