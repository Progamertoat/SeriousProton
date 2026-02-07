SeriousProton
=============

C++ game engine coded on top of [SDL2](https://libsdl.org/) from scratch. There will be dragons and undocumented stuff in here.

## Documentation

- **[Multiplayer Architecture](MULTIPLAYER_ARCHITECTURE.md)** - Complete documentation of the multiplayer networking protocol, including how to implement custom clients.

## Master server

The `/masterserver` directory contains a basic PHP server for registering game servers and distributing a list of servers to clients. See the [Multiplayer Architecture](MULTIPLAYER_ARCHITECTURE.md) documentation for details on the master server protocol.
