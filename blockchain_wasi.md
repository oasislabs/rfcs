# Blockchain-flavored WASI

## Abstract

  The WebAssembly System Interface [WASI] defines a standard for an
  interface for WebAssembly when run outside of a browser context.
  This interface can be adapted to a blockchain context, where the
  assembly intermediate representation is designed to run in a
  deterministic and unprivileged context. This document expresses the
  interpretations of the system interfaces in that context.

## Status of This Memo

  This is an active First Draft document under iteration.

## Copyright Notice

  Copyright (c) 2019 Oasis Labs and the persons identified as the
  document authors.

  Code Components extracted from this document must include Simplified
  BSD License and are provided without warranty as described in the
  Simplified BSD License.

1. Introduction

  WebAssembly provides a similar set of goals (cross-platform, compactness,
  sandboxing, performance), as have been traditionally strived for in a
  blockchain context. Blockchain has traditionally favored determinism over
  performance, however it is notable that WASM is also deterministic except
  for handling of floating point NaN [WADETE]. What is needed to use
  WebAssembly as a basis for a blockchain computation platform is access to
  chain state.

  The WASI systems interface provides a limited interface to an external
  system for WebAssembly. 

2.  Normative Language

  The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
  "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
  "OPTIONAL" in this document are to be interpreted as described in
  BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all
  capitals, as shown here.


3. Motivation

The primary motivation of this interpretation is to allow a program
targeting a WASI compilation target to work in a blockchain context
with minimal if any modification, while exposing the capabilities to
allow extension for blockchain interaction.

4. Behavior of WASI in a Blockchain Context

  A transaction MUST execute the WASI program, with input to the
  transaction available on the `stdin` stream passed to the program.
  Output of the program, as written to the `stdout` default file
  descriptor, will be considered to be the transaction output.
  The `proc_exit` call indicates the termination of execution, with
  the return status code used to determine if the transaction
  completed (0) or should be reverted (non-zero).

  The `stderr` (fd 2) stream MAY be used for debugging. When running
  a contract in debugging mode, the output to `stderr` MAY be made
  available to the caller even if the transaction execution aborts.
  When running in a production mode, output to `stderr` SHOULD be
  discarded.

  On-chain storage MUST be exposed as a virtual file system.

  A specific namespace SHOULD be reserved within the virtual file system
  for exposing blockchain-specific functionality.
  This functionality SHOULD be identified as `/opt/CHAIN`, where
  `CHAIN` MAY be either a common short name identifying the network
  OR the network identifier adopted by the chain.

  * `/opt/CHAIN/ADDRESS` SHOULD should be used as the conventional
  location for references to other contracts. Within these directories
  the following files SHOULD be exposed:
  * `code` - the code at that address, if any.
  * `sock` - A file descriptor for cross-contract calls to that address.
  * `balance` - The balance at that address.

  The context of the execution is most naturally exposed through the WASI
  `environ_get` call. This includes context of the exeuction in
  a blockchain context, including the gas constraints, the
  initiator of execution, and metadata of the transaction
  initiating the execution. Such metadata SHOULD be exposed as
  environmental variables.

5. Standards Considerations

5.1 File System Nodes

  Blockchain-specific functionality is exposed in the `/opt`
  namespace where it does not conflict with existing file
  system semantics.

  The home directory MUST be writable, and is mapped to persistant
  storage that will be committed with a transaction and
  subsequently available to future transactions.

  The `/tmp` directory SHOULD be used to provide a namespace that is
  writable but ephemeral, and will not be saved between
  transactions.

  The remainder of the file system presented SHOULD be read-only,
  and any attempts at writes SHOULD fail.

  The virtual file system translates paths to addresses in
  the underlying key-value store. In WASI, the paths and byte
  offsets are supplied to the Wasm runtime via `__wasi_fd_open`,
  `__wasi_fd_seek`. The runtime associates the key with the file
  descriptor so that it can service future reads and writes
  received from `__wasi_fd_(read|write)`.

5.1.1 Deviations from Expected File System Behavior

  Directory listing of the `/opt/CHAIN` directory will list 0
  items, since the full namespace cannot meaningfuly be represented.
  Attempts to `fd_fdstat_set_rights` (change permissions), or
  `_filestat_set_times` (changing metadata), on blockchain-namespace
  files will be a failure that aborts execution for safety.

5.2 Socket Address and Interoperability

  Opening of a socket to another blockchain executable via a
  FD obtained from `/opt/CHAIN/ADDRESS/sock` allows for cross
  contract calls. Data sent over the socket via `sock_send` will
  be used as the input available on the `stdin` descriptor available
  to the other execution.
  The first `sock_recv` call will begin execution and block until
  output is available.

5.3 Environment

  We expose the environmental variables
  * address
  * gas_left
  * sender
  * value
  * balance
  * block_number
  * $HOME is set to `/home/ADDRESS`, the current contract address.

5.4 Process Signals

  Any signal issued via `proc_raise`, or `proc_exit` will end the process.
  A call that is not a 0 exit status to `proc_exit` will abort and revert
  the execution.

5.5 Logs & Events

  Logs in a blockchain context allow a contract to expose a structured
  message to the outside world. Since these messages are structured,
  they do not map cleanly to file system items. Instead, they SHOULD be
  considered as a chain-defined protocol, similar to IPC messages
  a program would pass to a system logging service. An socket for
  communication to a logging service to ultimately emit externally
  visible events SHOULD live at `/srv/CHAIN/log.sock` if such a service
  is available. The protocol for submission of logs is left to
  chain-specific implementations.

6. Security Considerations

  The semantics of `stderr` provide a potential for a transaction to
  produce externally visible results even when an execution is aborted.
  As such, it should not be exposed in production uses.

  The potential for multiple threads can introduce race conditions and
  concurrency issues. For the current time, execution in this context
  will only allow for single-threaded execution to mitigate such
  potential.

  Fine-grained exposure of time and clock can allow for potential
  gaming of execution or non-determinism. (for instance, timing out
  in selective circumstances). As such, clock responses must be
  provided with sufficiently coarse granularity that they can not be
  relied upon for precise timing, or must be offset such that
  executions can be deterministically re-computed using the same timing
  offsets. Advancing of time within an execution MUST either be
  calcuated directly as a function of executed cycles, OR must be
  fixed to a static time returned throughout execution.

7. References

[WASI] https://github.com/CraneStation/wasmtime/blob/master/docs/WASI-intro.md

[WADETE] https://github.com/WebAssembly/design/blob/master/Nondeterminism.md

8. Author's Addresses

  Nick Hynes
  Oasis Labs
  Email: nhynes@oasislabs.com

  Will Scott
  Oasis Labs
  Email: will@oasislabs.com
