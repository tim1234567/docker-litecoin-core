# uphold/litecoin-core

A Litecoin Core docker image.

[![uphold/litecoin-core][docker-pulls-image]][docker-hub-url] [![uphold/litecoin-core][docker-stars-image]][docker-hub-url] [![uphold/litecoin-core][docker-size-image]][docker-hub-url] [![uphold/litecoin-core][docker-layers-image]][docker-hub-url]

## Tags

- `0.16.3`, `latest` ([0.16/Dockerfile](https://github.com/uphold/docker-litecoin-core/blob/master/0.16/Dockerfile))
- `0.15.1`, `0.15` ([0.15/Dockerfile](https://github.com/uphold/docker-litecoin-core/blob/master/0.15/Dockerfile))
- `0.14.2`, `0.14` ([0.14/Dockerfile](https://github.com/uphold/docker-litecoin-core/blob/master/0.14/Dockerfile))

**Picking the right tag**

- `uphold/litecoin-core:latest`: points to the latest stable release available of Litecoin Core. Use this only if you know what you're doing as upgrading Litecoin Core blindly is a risky procedure.
- `uphold/litecoin-core:<version>`: based on a slim Debian image, points to a specific version branch or release of Litecoin Core. Uses the pre-compiled binaries which are fully tested by the Litecoin Core team.

## What is Litecoin Core?

Litecoin Core is the Litecoin reference client and contains all the protocol rules required for the Litecoin network to function. This client is used by mining pools, merchants and services all over the world for its rock solid stability, feature set and security. Learn more about [Litecoin Core](https://litecoincore.org/).

## Usage

### How to use this image

This image contains the main binaries from the Litecoin Core project - `litecoind`, `litecoin-cli` and `litecoin-tx`. It behaves like a binary, so you can pass any arguments to the image and they will be forwarded to the `litecoind` binary:

```sh
❯ docker run --rm uphold/litecoin-core \
  -printtoconsole \
  -regtest=1 \
  -rpcallowip=172.17.0.0/16 \
  -rpcauth='foo:1e72f95158becf7170f3bac8d9224$957a46166672d61d3218c167a223ed5290389e9990cc57397d24c979b4853f8e'
```

By default, `litecoind` will run as user `litecoin` for security reasons and with its default data dir (`~/.litecoin`). If you'd like to customize where `litecoind` stores its data, you must use the `LITECOIN_DATA` environment variable. The directory will be automatically created with the correct permissions for the `litecoin` user and `litecoind` automatically configured to use it.

```sh
❯ docker run -e LITECOIN_DATA=/var/lib/litecoind --rm uphold/litecoin-core \
  -printtoconsole \
  -regtest=1
```

You can also mount a directory in a volume under `/home/litecoin/.litecoin` in case you want to access it on the host:

```sh
❯ docker run -v ${PWD}/data:/home/litecoin/.litecoin --rm uphold/litecoin-core \
  -printtoconsole \
  -regtest=1
```

You can optionally create a service using `docker-compose`:

```yml
litecoin-core:
  image: uphold/litecoin-core
  command:
    -printtoconsole
    -regtest=1
```

### Using RPC to interact with the daemon

There are two communications methods to interact with a running Litecoin Core daemon.

The first one is using a cookie-based local authentication. It doesn't require any special authentication information as running a process locally under the same user that was used to launch the Litecoin Core daemon allows it to read the cookie file previously generated by the daemon for clients. The downside of this method is that it requires local machine access.

The second option is making a remote procedure call using a username and password combination. This has the advantage of not requiring local machine access, but in order to keep your credentials safe you should use the newer `rpcauth` authentication mechanism.

#### Using cookie-based local authentication

Start by launch the Litecoin Core daemon:

```sh
❯ docker run --rm --name litecoin-server -it uphold/litecoin-core \
  -printtoconsole \
  -regtest=1
```

Then, inside the running `litecoin-server` container, locally execute the query to the daemon using `litecoin-cli`:

```sh
❯ docker exec --user litecoin litecoin-server litecoin-cli -regtest getmininginfo

{
  "blocks": 0,
  "currentblocksize": 0,
  "currentblockweight": 0,
  "currentblocktx": 0,
  "difficulty": 4.656542373906925e-10,
  "errors": "",
  "networkhashps": 0,
  "pooledtx": 0,
  "chain": "regtest"
}
```

In the background, `litecoin-cli` read the information automatically from `/home/litecoin/.litecoin/regtest/.cookie`. In production, the path would not contain the regtest part.

#### Using rpcauth for remote authentication

Before setting up remote authentication, you will need to generate the `rpcauth` line that will hold the credentials for the Litecoin Core daemon. You can either do this yourself by constructing the line with the format `<user>:<salt>$<hash>` or use the official `rpcuser.py` script to generate this line for you, including a random password that is printed to the console.

Example:

```sh
❯ curl -sSL https://raw.githubusercontent.com/litecoin-project/litecoin/master/share/rpcuser/rpcuser.py | python - <username>

String to be appended to litecoin.conf:
rpcauth=foo:1e72f95158becf7170f3bac8d9224$957a46166672d61d3218c167a223ed5290389e9990cc57397d24c979b4853f8e
Your password:
-ngju1uqGUmAJIQDBCgYbatzhcJon_YGU23t313388g=
```

Note that for each run, even if the username remains the same, the output will be always different as a new salt and password are generated.

Now that you have your credentials, you need to start the Litecoin Core daemon with the `-rpcauth` option. Alternatively, you could append the line to a `litecoin.conf` file and mount it on the container.

Let's opt for the Docker way:

```sh
❯ docker run --rm --name litecoin-server -it uphold/litecoin-core \
  -printtoconsole \
  -regtest=1 \
  -rpcallowip=172.17.0.0/16 \
  -rpcauth='foo:1e72f95158becf7170f3bac8d9224$957a46166672d61d3218c167a223ed5290389e9990cc57397d24c979b4853f8e'
```

Two important notes:

1. Some shells require escaping the rpcauth line (e.g. zsh), as shown above.
2. It is now perfectly fine to pass the rpcauth line as a command line argument. Unlike `-rpcpassword`, the content is hashed so even if the arguments would be exposed, they would not allow the attacker to get the actual password.

You can now connect via `litecoin-cli` or any other [compatible client](https://github.com/ruimarinho/bitcoin-core). You will still have to define a username and password when connecting to the Litecoin Core RPC server.

To avoid any confusion about whether or not a remote call is being made, let's spin up another container to execute `litecoin-cli` and connect it via the Docker network using the password generated above:

```sh
❯ docker run --link litecoin-server --rm uphold/litecoin-core \
  litecoin-cli \
  -rpcconnect=litecoin-server \
  -regtest \
  -rpcuser=foo \
  -rpcpassword='-ngju1uqGUmAJIQDBCgYbatzhcJon_YGU23t313388g=' \
  getmininginfo

{
  "blocks": 0,
  "currentblocksize": 0,
  "currentblockweight": 0,
  "currentblocktx": 0,
  "difficulty": 4.656542373906925e-10,
  "errors": "",
  "networkhashps": 0,
  "pooledtx": 0,
  "chain": "regtest"
}
```

### Exposing Ports

Depending on the network (mode) the Litecoin Core daemon is running as well as the chosen runtime flags, several default ports may be available for mapping.

Ports can be exposed by mapping all of the available ones (using `-P` and based on what `EXPOSE` documents) or individually by adding `-p`. This mode allows assigning a dynamic port on the host (`-p <port>`) or assigning a fixed port `-p <hostPort>:<containerPort>`.

Example for running a node in `regtest` mode mapping JSON-RPC/REST and P2P ports:

```sh
docker run --rm -it \
  -p 19332:19332 \
  -p 19444:19444 \
  uphold/litecoin-core \
  -printtoconsole \
  -regtest=1 \
  -rpcallowip=172.17.0.0/16 \
  -rpcauth='foo:1e72f95158becf7170f3bac8d9224$957a46166672d61d3218c167a223ed5290389e9990cc57397d24c979b4853f8e'
```

To test that mapping worked, you can send a JSON-RPC curl request to the host port:

```
curl --data-binary '{"jsonrpc":"1.0","id":"1","method":"getnetworkinfo","params":[]}' http://foo:-ngju1uqGUmAJIQDBCgYbatzhcJon_YGU23t313388g=@127.0.0.1:19332/
```

#### Mainnet

- JSON-RPC/REST: 9332
- P2P: 9333

#### Testnet

- JSON-RPC: 19332
- P2P: 19333

#### Regtest

- JSON-RPC/REST: 19332
- P2P: 19444

## Archived tags

For historical reasons, the following tags are still available and automatically updated when the underlying base image is updated as well:

- `0.13.2`, `0.13` ([0.13/Dockerfile](https://github.com/uphold/docker-litecoin-core/blob/master/0.13/Dockerfile))
- `0.10.4`, `0.10` ([0.10/Dockerfile](https://github.com/uphold/docker-litecoin-core/blob/master/0.10/Dockerfile))

## Supported Docker versions

This image is officially supported on Docker version 17.09, with support for older versions provided on a best-effort basis.

## License

The [uphold/litecoin-core][docker-hub-url] docker project is under MIT license.

[docker-hub-url]: https://hub.docker.com/r/uphold/litecoin-core
[docker-layers-image]: https://img.shields.io/imagelayers/layers/uphold/litecoin-core/latest.svg?style=flat-square
[docker-pulls-image]: https://img.shields.io/docker/pulls/uphold/litecoin-core.svg?style=flat-square
[docker-size-image]: https://img.shields.io/imagelayers/image-size/uphold/litecoin-core/latest.svg?style=flat-square
[docker-stars-image]: https://img.shields.io/docker/stars/uphold/litecoin-core.svg?style=flat-square
