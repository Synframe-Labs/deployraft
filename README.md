# deployraft (dr)

Run a command in an application directory over SSH, or schedule it with the server's `at` command. TOML file defines shared servers and applications.

## Installation

With Rust and Cargo installed:

```sh
cargo install deployraft
```

Ensure `~/.cargo/bin` is on your `PATH`. Deployraft runs on Linux and macOS and requires local OpenSSH (`ssh`). Remote servers need Linux and `/bin/sh`. Scheduling also requires `at`, a running `atd` service, and permission to submit jobs.

### Manual installation

Install Rust and Cargo, then download the source and build the executable:

```sh
git clone https://github.com/Synframe-Labs/deployraft.git
cd deployraft
cargo install --path .
```

## Configuration

Create `~/.config/deployraft/config.toml` with the following contents. If `XDG_CONFIG_HOME` points to an absolute directory, use `$XDG_CONFIG_HOME/deployraft/config.toml` instead. Configuration is independent of your working directory. Use `--config PATH` to select another file. The configuration file can be a symlink.

```toml
[servers.production]
host = "example.com"
port = 22
user = "deploy"
auth = "password"

[apps.website]
server = "production"
directory = "/srv/website"
command = "git pull --ff-only"
# at = "22:30"
```

Add more named entries under `servers` and `apps`. Applications reference a server by name and each contain one command. Port defaults to `22`. Other fields shown above are required, except `at`.

- `auth = "password"` skips public keys and lets SSH prompt securely for a password. Passwords are never stored. Run from a terminal when authentication needs input.
- `auth = "key"` uses standard SSH keys or the agent. An optional `identity_file = "/home/you/.ssh/id_ed25519"` adds a local key. Its path must be absolute, without `~`, `%` or `${` substitutions, and may be a symlink. Use a symlink with a plain path for keys with these special characters in their names. SSH may also try identity files from its configuration. Encrypted keys may prompt for a passphrase.
- `directory` must be an absolute path on the server. Symlinks are allowed and resolved when the command runs, including for scheduled jobs.
- OpenSSH handles known hosts and host key verification normally. Deployraft opens a separate connection each time, disables backgrounding, configured remote commands and port forwarding, and enables authentication prompts in both modes. SSH diagnostics use `INFO` logging with debug overrides disabled. Connection setup has a 15-second timeout, with unresponsive connections detected through 15-second keepalive probes and a maximum of three unanswered probes. These settings do not limit the runtime of a command on a responsive server.

## Usage

```sh
deployraft website
deployraft --config /path/to/deployraft.toml website
deployraft --verbose website
deployraft --help
```

Use `--` before application names starting with `-`, for example `deployraft -- -staging`. Options must appear before this separator.

Deployraft prints the application, connection, directory, and progress. Use `--verbose` to also print the full command, which may expose secrets. Remote command output is always forwarded and may itself contain sensitive data.

Deployraft changes directory before executing the command with `/bin/sh`. An unsuccessful directory change prevents execution. Commands run without a remote terminal or standard input, so interactive commands and `sudo` password prompts are unsupported. Configuration commands are trusted shell code. Use `&&` between dependent operations, such as `git pull --ff-only && ./deploy.sh`. Success means the shell returned zero. Earlier failures in a sequence or pipeline may otherwise be hidden by a successful final command.

The server account must use a POSIX-compatible shell, such as `sh`, `bash` or `dash`, and execute the submitted command. `ForceCommand` and `authorized_keys` command wrappers that replace the submitted command are unsupported. Deployraft cannot detect that replacement and may report success for the wrapper instead of the deployment or job submission.

Command output, SSH diagnostics, and errors go to stdout. Authentication prompts are handled directly by SSH through the terminal. Immediate execution ends with `SUCCESS` or `FAILURE`. Exit status is zero on success and nonzero on failure.

Configuration errors report the file and, when available, a byte position without printing configuration contents. A failure to write output exits with a nonzero status without panicking. If stdout is unavailable, the final status cannot be displayed. Keep secrets out of commands, which are also passed as process arguments.

If SSH disconnects without confirming the result, the remote operation may already have started or completed. Check the server state before retrying, including `atq` for scheduled jobs. Failures do not roll back changes or cancel previously accepted jobs.

## Scheduling

Set `at = "22:30"` on an application to submit its command to the server's queue instead of executing immediately. Only the 24-hour format `HH:MM` is accepted, from `00:00` to `23:59`. Values containing a date are rejected.

The time uses the **server's timezone and clock**. The server's `at` schedules the job for today, or the next day if that time has already passed when the submission is processed. The job runs once. Minute-boundary and daylight saving behavior follow the server's `at` implementation.

Deployraft validates the time and checks the directory and scheduler before submitting. There is no minimum lead time. The server's `at -v` response shows the selected execution date and job confirmation. Check that date, especially near the requested time or during clock changes. Successful submission ends with `SCHEDULED` and exit status zero.

Scheduled commands need credentials available on the server when they run. A forwarded SSH agent stops being available after the deployment connection closes. For example, a scheduled Git command may need a server-side deploy key.

`SCHEDULED` confirms acceptance only. Deployraft exits without monitoring later output, completion, or failures. The server's scheduling service must remain running. Repeating the deployment submits another job. Manage queued jobs directly on the server with `atq` and `atrm`.

## Author

Created by Synframe Labs - [synframe.eu](https://synframe.eu).

## License

Apache License 2.0. See [LICENSE](LICENSE).
