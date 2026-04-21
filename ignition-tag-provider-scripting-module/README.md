# Tag Provider Scripting Module

**ignition-tag-script-module-gem**: An Ignition module for creating tag providers via gateway scripts.

This module extends Ignition SCADA's scripting capabilities by providing programmatic control over tag provider lifecycle management.

## Installation

1. Download the `Tag-Provider-Scripting.unsigned.modl` from the `build` directory.
2. Install via Ignition Gateway Web Interface (Config -> System -> Modules).
3. Ensure developer mode is enabled if installing the unsigned module.

## Usage

```python
# Create a provider
system.tag.provider.create("MyProvider", "Description")

# List providers
providers = system.tag.provider.list()

# Delete a provider
system.tag.provider.delete("MyProvider")
```
