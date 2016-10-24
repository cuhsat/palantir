# Palantir ![Build](https://fxj.vela.uberspace.de/palantir/palantir/badges/master/build.svg)
Palantir is a [Lua](https://www.lua.org) scriptable, extendable, tiny reverse
shell, using a human readable protocol written in C and Lua.

## Usage
```
$ palantir [-dhlv] [-a TOKEN] [-c COMMAND] [-f FILE] HOST PORT
```

### Options:
* `-d` Starts in client mode _(passive)_
* `-h` Shows the usage information
* `-l` Shows the license
* `-v` Shows the version
* `-a` Authentication token
* `-c` Executes the command
* `-f` Executes the file

The option `-f` has precedence over the option `-c`. The program will not exit 
after all commands, either specified by `-c` or `-f`, are processed.

### Commands:
* `-- exit` Shutdown server
* `-- halt` Shutdown client

All input will be evaluated and execute as Lua commands. The internal function
`shell` will execute system commands by using the users default shell and 
return the results where `strerr` will be mapped to `stdout`.

Use <kbd>Ctrl</kbd>+<kbd>n</kbd> to insert a new line.

## Environment
An user specific profile can be place under `~/.palantir.lua`.

Here is an example profile:
```
-- greet client
function client_connected()
  return 'Hello\n'
end

-- debug
print('Profile loaded')
```

### Globals
A new global table named `P` will be defined which contains all shell specific
functions and properties.

#### Constants
* `MODE`    The command line option `-d`
* `HOST`    The command line argument `HOST`
* `PORT`    The command line argument `PORT`
* `TOKEN`   The command line argument `-a`
* `STACK`   The command line argument `-c`/`-f`
* `DEBUG`   The debug flag if compiled with `-DDEBUG=1`
* `VERSION` The semantic version number

#### Functions

##### `error(message)`
Prints the error `message`.

##### `event(source, event, param)`
Calls the function `<source>_<event>(<param>)` if exists.

##### `load(chunk)`
Returns the output of the executed `chunk` (global environment).

#### Network

##### `net.server(host, port)`
Starts on `host` and `port` in _server mode_.

##### `net.client(host, port)`
Starts on `host` and `port` in _client mode_.

##### `net.connect(host, port)`
Connects to `host` and `port`.

##### `net.listen(host, port)`
Listens on `host` and `port`.

##### `net.accept()`
Accepts incoming connections. _(blocking)_

##### `net.recv()`
Returns the received `command` and `param`. _(blocking)_

##### `net.send(command, param)`
Sends the given `command` and `param`. _(blocking)_

#### Operating System

##### `os.env(path)`
Returns the `user`, `host` and `path` variables, sets the `path` if given.

##### `os.execute(command)`
Returns the `command` output executed by the users default shell.

##### `os.readline(prompt)`
Prints `prompt` and returns the users input.

##### `os.sleep(milliseconds)`
Sleeps for the given `milliseconds`.

### Callbacks
The default shell functionality can be extended by creating custom event
callbacks in the users profile. There are four different event sources:

* `client_connected` Called when the client connects
* `client_<command>` Called when the client receives a `<command>`
* `server_<command>` Called when the server receives a `<command>`
* `server_prompt`    Called when the server processes a prompt

All callbacks except `client_connect` must return a `boolean`. In case `true`
is returned, all further processing will be prevented. The `client_connected`
callback must return a `string`, which will be displayed by the client.

The `<command>` names will be converted to lowercase.

Here is an example on how to implement an simple echo server:
```
function client_connected()
  return 'This is an echo server'
end
```
```
function client_echo(param)
  P.net.send('ECHO', param)
  return true
end
```
```
function server_echo(param)
  io.write(param .. '\n')
  return true
end
```
```
function server_prompt(line)
  P.net.send('ECHO', line)
  return true
end
```

### Shortcuts
The shortcut `shell` is provided as an alias of `P.os.execute`:
```
return shell('echo hello')
```

## Protocol
The protocol consists of two layers:

1. Network Layer (_transportation_ handled by `C`)
2. Command Layer (_interpretation_ handled by `Lua`)

### Network Layer
A network frame is build according to the following format:
```
CHECKSUM (4 bytes) | SIZE (4 bytes) | DATA (n bytes)
```
The `CHECKSUM` is a bitwise CRC-32 only over the `DATA` field.

#### Authentication
If an authentication `TOKEN` is provided, the network frames checksum will be 
pre-feed with the CRC-32 of the token before calculation.

### Command Layer
A command is build according to the following format:
```
COMMAND (4 bytes) | BLANK (1 byte) | PARAM (n bytes)
```
Each command consists of a `5` byte command header followed by `0` to `n`
bytes of `param`. A command header will end with a single blank ` ` character
for better readability.

#### Client
Commands issued by the client:

##### `HELO <user>@<host>:<path>`
Shows the command prompt.

##### `TEXT <text>`
Prints the text.

#### Server
Commands issued by the server:

##### `EXEC <command>`
Executes the command.

##### `PATH <path>`
Changes the path.

##### `HALT`
Halts the client.

### Examples
```
Client: HELO root@localhost:/
```
```
Server: PATH var
```

```
Client: HELO root@localhost:/var
```
```
Server: EXEC return shell('echo hello')
```
```
Client: TEXT hello
```

```
Client: HELO root@localhost:/var
```
```
Server: EXIT
```

## Building
```
$ make all test install
```

### Flags
Use these flags when calling [`make`](Makefile):

* `NO_DAEMON`   Disable daemonize support
* `NO_READLINE` Disable `readline` support

```
$ make all test install <flag>=1
```

### Dependancies
The following libraries are required:

* [Lua 5.3](https://www.lua.org)

The following libraries are supported:

* [Readline](https://cnswww.cns.cwru.edu/php/chet/readline/rltop.html)

## License
Licensed under the terms of the [MIT License](LICENSE).