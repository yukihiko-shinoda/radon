# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Radon is a Python static analysis tool that computes various code metrics:
- **Cyclomatic Complexity (CC)**: McCabe's complexity metric
- **Raw metrics**: SLOC, comment lines, blank lines, logical lines of code
- **Halstead metrics**: Volume, difficulty, effort, time, bugs estimate
- **Maintainability Index (MI)**: Compound metric used in Visual Studio

## Development Commands

### Testing
```bash
# Run all tests
python radon/tests/run.py

# Run tests with coverage
make cov

# Generate HTML coverage report
make htmlcov

# Run tests with tox across multiple Python versions
tox
```

### Code Quality
```bash
# Format code (isort + black)
make format

# Run linting (flake8)
make lint

# Format and lint together
make f

# Run PEP8 checks
make pep8

# Run pylint
make pylint
```

### Documentation
```bash
# Build documentation
make docs
# Or:
cd docs && make html
```

### Building and Publishing
```bash
# Build distribution packages
python setup.py sdist bdist_wheel

# Publish to PyPI (maintainers only)
make publish
```

## Architecture

### Core Modules

**radon/visitors.py**
- Contains AST visitors that analyze Python code structure
- `ComplexityVisitor`: Traverses AST to compute cyclomatic complexity
- `HalsteadVisitor`: Counts operators and operands for Halstead metrics
- Defines `Function` and `Class` namedtuples representing code blocks
- Key helper: `code2ast()` converts source strings to AST nodes

**radon/complexity.py**
- High-level API for cyclomatic complexity analysis
- `cc_visit()` and `cc_visit_ast()`: Main entry points for CC analysis
- `cc_rank()`: Converts numeric complexity to letter grade (A-F)
- `average_complexity()`: Computes average across code blocks
- `sorted_results()`: Sorts blocks by SCORE, LINES, or ALPHA
- `add_inner_blocks()`: Flattens nested closures/classes for reporting

**radon/raw.py**
- Analyzes raw source code metrics using Python's tokenizer
- `analyze()`: Returns `Module` namedtuple with loc, lloc, sloc, comments, multi, blank, single_comments
- `_logical()`: Counts logical lines (e.g., `if x: return 0` = 2 logical lines)
- Uses `tokenize.generate_tokens()` to parse source at token level

**radon/metrics.py**
- Implements Halstead metrics and Maintainability Index
- `h_visit()` and `h_visit_ast()`: Compute Halstead metrics returning `HalsteadReport`
- `mi_visit()`: Computes Maintainability Index (0-100 scale)
- `mi_rank()`: Converts MI score to letter grade (A/B/C)
- MI formula combines Halstead volume, complexity, SLOC, and comments

**radon/cli/__init__.py**
- CLI interface built with `mando` library
- Defines four main commands: `cc`, `raw`, `mi`, `hal`
- `Config` class: Holds configuration values from CLI args and config files
- `FileConfig`: Reads settings from `radon.cfg`, `setup.cfg`, `pyproject.toml`, or `~/.radon.cfg`
- Output formatting: JSON, XML, Markdown, Code Climate, or terminal

**radon/cli/harvest.py**
- `Harvester` base class for collecting metrics across files
- `CCHarvester`, `RawHarvester`, `MIHarvester`, `HCHarvester`: Specialized harvesters
- Handles file traversal, exclusion/ignore patterns, and output formatting
- Supports Jupyter notebooks via `--include-ipynb` flag

**radon/contrib/flake8.py**
- Flake8 plugin integration
- `Flake8Checker`: Reports R701 errors for functions exceeding complexity threshold
- Configurable via `--radon-max-cc`, `--radon-no-assert`, `--radon-show-closures`

### Key Design Patterns

1. **Visitor Pattern**: All analysis uses AST visitors (`ComplexityVisitor`, `HalsteadVisitor`) to traverse code structure
2. **Configuration Hierarchy**: CLI args override config files (order: `radon.cfg` > `setup.cfg` > `pyproject.toml` > `~/.radon.cfg`)
3. **Harvester Pattern**: Separate harvester classes collect metrics and format output
4. **Dual Entry Points**: Most functions have both source string (`_visit`) and AST (`_visit_ast`) versions

### Cyclomatic Complexity Calculation

Complexity increases by:
- +1 for `if`, `elif`, `while`, `for`, `async for`, comprehensions
- +N for boolean operators with N operands (e.g., `a and b or c` = +2)
- +N for try/except with N handlers (+ `else` block)
- +N for `match` statement with N cases (excluding wildcard `_`)
- +1 for `assert` (unless `--no-assert` flag is set)
- Functions/methods start at complexity 1

### File Organization

```
radon/
├── __init__.py          # Entry point, version
├── visitors.py          # AST visitors (ComplexityVisitor, HalsteadVisitor)
├── complexity.py        # CC high-level API
├── raw.py              # Raw metrics (LOC, SLOC, etc.)
├── metrics.py          # Halstead & MI metrics
├── cli/
│   ├── __init__.py     # CLI commands (cc, raw, mi, hal)
│   ├── harvest.py      # Harvester classes for file processing
│   ├── tools.py        # CLI utilities
│   └── colors.py       # Terminal color support
├── contrib/
│   └── flake8.py       # Flake8 plugin
└── tests/
    ├── run.py          # Test runner (uses pytest)
    └── test_*.py       # Test modules
```

## Python Version Support

- Supports Python 2.7 and 3.6+ (3.0-3.3 excluded)
- Uses conditional imports for Python 2/3 compatibility
- PyPy 3.5 v7.3.1 tested in CI
- Uses `tomllib` (3.11+) or `tomli` (3.6-3.10) for TOML parsing

## Special Considerations

1. **File Encoding**: Set `RADONFILESENCODING=UTF-8` on Windows for Unicode files
2. **Jupyter Notebooks**: Require `nbformat` package; use `--include-ipynb` and optionally `--ipynb-cells`
3. **Assert Statements**: By default count toward complexity; disable with `--no-assert`
4. **Closures**: Not shown by default; enable with `--show-closures`
5. **Multiline Strings**: Counted as comments in MI calculation unless `--multi=False`
