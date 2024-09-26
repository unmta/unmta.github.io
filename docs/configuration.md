# Configuration

UnMTA works out of the box without any configuration parameters. However, to adapt the server to your specific environment, there are number of parameters we can set via the cleverly named `unfig.toml` file. :wink:

Let's start by creating a `config` directory, and adding a default `unfig.toml` file to it:

```bash
mkdir config
touch config/unfig.toml
```

```toml title="config/unfig.toml"
# Below are all the configuration options, along with their default values
[smtp]
port=2525
listen="localhost" # Use 0.0.0.0 to listen on all interfaces
#hostname="your.mail.server" # Optional, defaults to the hostname of the machine
inactivityTimeout=300 # Seconds to wait before closing an idle connection
gracefulStopTimeout=300 # Seconds to wait before forcing the server to shut down after a stop() is issued

[auth]
enable=false # Enable SMTP authentication
requireTLS=true # Require TLS before authentication

[tls]
enableStartTLS=false # Enable STARTTLS Support
#key="tls_key.pem" # Path to the TLS key file
#cert="tls_cert.pem" # Path to the TLS certificate file

[log]
level = "smtp" # error, warn, info, debug, smtp

[plugins]
# Plugin specific configs. Example:
# [plugins.examplePlugin]
# doStuff=true
```

!!! note

    The `config` directory will also house the configuration files for your [plugins](writing-plugins.md#configuration).
