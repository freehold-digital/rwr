# Services Blueprint

The Services Blueprint manages system services: start, stop, enable, disable, restart, reload, status checks, and unit file creation or removal across operating systems.

See [Fields Common to Every Blueprint](common-fields.md) for `profiles`,
`import`, `interactive`, and the rule that an unknown key is an error.

## Blueprint Structure

The Services Blueprint is defined in a YAML, JSON, TOML, or CUE file and consists of an array of service objects. Each service object represents a system service and its associated properties.

```yaml
services:
  - name: nginx
    action: start
    elevated: true
  - name: mysql
    action: stop
    elevated: true
```

## Service Object Properties

A service object can have the following properties:

| Property | Required | Description |
|----------|----------|-------------|
| `name` | Yes, if `import` is not provided | The name of the service |
| `import` | Yes, if `name` is not provided | Path to import service definitions from another file (relative to blueprint directory) |
| `profiles` | No | List of profiles this service belongs to. If empty, service is always managed (base item) |
| `action` | Yes | The action to perform on the service (start, stop, enable, disable, restart, reload, status, create, delete) |
| `elevated` | No | Whether the service requires elevated privileges (default: false) |
| `target` | No | The target file for the service (used with create and delete actions) |
| `content` | No | The content of the service file (used with the create action) |
| `source` | No | The source file for the service (used with the create action) |
| `file` | No | The file associated with the service (used with the delete action) |
| `interactive` | No | Override global interactive mode for this service (`true`/`false`). If omitted, uses the global `--interactive` flag |

## Blueprint Imports

Import service definitions from other files to share common service configurations:

```yaml
services:
  # Import shared base services
  - import: ../../Common/services/base-services.yaml

  # Add environment-specific services
  - name: custom-app
    action: enable
    elevated: true
    profiles:
      - production
```


## Supported Actions

- `start`: Start the service
- `stop`: Stop the service
- `enable`: Enable the service to start automatically on system boot
- `disable`: Disable the service from starting automatically on system boot
- `restart`: Restart the service
- `reload`: Reload the service configuration
- `status`: Check the status of the service
- `create`: Create a new service file
- `delete`: Delete an existing service file

`rwr validate` accepts all nine actions.

## Platform-Specific Considerations

The Services Blueprint handles service management differently depending on the operating system:

### Linux (systemd)

On Linux systems with systemd, the Services Blueprint uses the `systemctl` command to manage services. The `create` and `delete` actions manage service unit files in the appropriate systemd directories.

### macOS (launchd)

On macOS, the Services Blueprint uses the `launchctl` command to manage services. The `create` and `delete` actions manage service plist files in the `/Library/LaunchDaemons` directory.

### Windows

On Windows, the Services Blueprint uses the `sc` command to manage services. The `create` and `delete` actions manage service configuration and binaries.

## Examples

More working trees in every supported format, including JSON and TOML: [examples](../../examples/README.md).
