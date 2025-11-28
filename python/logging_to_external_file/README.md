# Python Logging Setup

A simple, reusable Python logging configuration module with rotating file handler and optional console output.

## Features

- 🔄 **Rotating Log Files**: Automatically rotates logs when they reach a size limit (default: 5 MB)
- 📁 **Organized Storage**: Logs stored in a dedicated directory with configurable path
- 🖥️ **Optional Console Output**: Enable/disable console logging with a single parameter
- 🧹 **Clean Configuration**: Removes existing handlers to prevent duplicate logs
- ⚙️ **Customizable**: Configure log level, file size, backup count, and more

## Installation

Simply copy `logging_setup.py` to your project directory.

## Usage

### Basic Setup

```
from logging_setup import setup_logging
import logging

# Initialize logging (file only)
setup_logging()

logger = logging.getLogger(__name__)
logger.info("Application started")
```

### Enable Console Logging

```
from logging_setup import setup_logging
import logging

# Initialize with console output
setup_logging(console=True)

logger = logging.getLogger(__name__)
logger.info("This will appear in both file and console")
```

### Custom Configuration

```
from logging_setup import setup_logging
import logging

setup_logging(
    log_file="my_app.log",
    max_bytes=10 * 1024 * 1024,  # 10 MB
    backup_count=5,
    level=logging.DEBUG,
    console=True
)

logger = logging.getLogger(__name__)
logger.debug("Debug message")
logger.info("Info message")
logger.warning("Warning message")
logger.error("Error message")
```

## Code

### `logging_setup.py`

```
import logging
import os
from logging.handlers import RotatingFileHandler


def setup_logging(
    log_file: str = "app.log",
    max_bytes: int = 5 * 1024 * 1024,  # 5 MB
    backup_count: int = 10,
    level: int = logging.INFO,
    console: bool = False  # ✅ control console logging
):
    """
    Sets up a global logging configuration.
    Removes all previous handlers and ensures logs go to a rotating file.
    If console=True, also logs to the console.
    """
    log_dir = r"C:/logs/amh/employee_meal_request/replication"
    os.makedirs(log_dir, exist_ok=True)
    log_path = os.path.join(log_dir, log_file)

    # Remove all existing handlers from root logger
    root = logging.getLogger()
    for handler in root.handlers[:]:
        root.removeHandler(handler)

    # Define formatter
    formatter = logging.Formatter(
        "%(asctime)s - %(levelname)s - %(name)s - %(message)s"
    )

    # File handler with rotation
    file_handler = RotatingFileHandler(
        log_path,
        maxBytes=max_bytes,
        backupCount=backup_count,
        encoding="utf-8"
    )
    file_handler.setFormatter(formatter)

    # Add handlers to root logger
    handlers = [file_handler]

    if console:
        console_handler = logging.StreamHandler()
        console_handler.setFormatter(formatter)
        handlers.append(console_handler)

    logging.basicConfig(
        level=level,
        handlers=handlers,
    )

    logging.captureWarnings(True)
    logging.getLogger().info("Global logging initialized (console: %s)", console)
```

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `log_file` | `str` | `"app.log"` | Name of the log file |
| `max_bytes` | `int` | `5242880` (5 MB) | Maximum size of each log file before rotation |
| `backup_count` | `int` | `10` | Number of backup log files to keep |
| `level` | `int` | `logging.INFO` | Logging level (DEBUG, INFO, WARNING, ERROR, CRITICAL) |
| `console` | `bool` | `False` | Enable console output alongside file logging |

## Log File Location

By default, logs are saved to:
```
C:/logs/amh/employee_meal_request/replication/
```

You can modify the `log_dir` variable in the `setup_logging()` function to change this path.

## Log Format

```
2025-11-28 06:42:15,123 - INFO - __main__ - Application started
```

Format: `timestamp - level - logger_name - message`

## License

MIT

## Contributing

Feel free to submit issues or pull requests!
```
