# claude-galaxy-plugins

A Claude Code plugin marketplace for Galaxy Project related plugins.

## Installation

```bash
# Add marketplace
/plugin marketplace add galaxyproject/claude-galaxy-plugins

# List available plugins
/plugin list claude-galaxy-plugins

# Install a plugin
/plugin install gx-arch-review@claude-galaxy-plugins
```

## Available Plugins

| Plugin | Description |
|--------|-------------|
| [gx-arch-review](plugins/gx-arch-review/README.md) | Code review commands for Galaxy codebase contributions |
| [gx-dev](plugins/gx-dev/README.md) | Skills and an exploration agent for writing Galaxy code: migrations, API endpoints, tests, linting |

## Development

The `gx-arch-review` plugin is built from the [galaxy-architecture](https://github.com/jmchilton/galaxy-architecture) repository.

### Local Testing

```bash
claude --plugin-dir /path/to/galaxy-plugins
```

## License

MIT License - see [LICENSE](LICENSE) for details.
