# AFXDP-rs

This module provides a Rust interface for AF_XDP which wraps [libbpf](https://github.com/libbpf/libbpf) built on [libbpf-sys](https://github.com/alexforster/libbpf-sys).

**Author:** Dan Siemon \<dan@coverfire.com\>

[![docs.rs](https://docs.rs/afxdp/badge.svg)](https://docs.rs/crate/afxdp)

AF_XDP:

* <https://www.kernel.org/doc/html/latest/networking/af_xdp.html>
* <https://lwn.net/Articles/750845/>

The goals of this crate, in order, are:

1. Correctness
2. Performance
3. Ease of use

## Current Status

The API works for my current use cases but I expect it will change to achieve higher performance and better usability.

If you have knowledge of Rust FFI and general Rust unsafe things I would greatly appreciate some help auditing this crate as my experience in this area is limited.

## Sample Programs

There are two sample programs in `examples/`:

* `l2fwd-1link`: Receives frames on a single link/queue, swaps the MAC and
  writes back to the same link/queue. This is roughly like the kernel
  xdpsock_user.c sample program in l2fwd mode.
* `l2fwd-2link`: Receives frames from two link/queue pairs (separate devices)
  and forwards the frames to the opposite link.

You can run with `cargo run --release --example <example_name> -- [example options]` to
see a list of options run `cargo run --release --example <example_name> -- --help`

## Performance

Test System:

* Intel(R) Xeon(R) CPU E5-2680 v4 @ 2.40GHz
* Single X710 (4 physical 10G ports) (i40e)
* Kernel: 5.4.2-300.fc31.x86_64
* Kernel boot args: skew_tick=1 mitigations=off selinux=0 isolcpus=4-27 nohz_full=4-27 rcu_nobcs=4-27 default_hugepagesz=1G hugepagesz=1G hugepages=4

Traffic Generator:

* Intel(R) Core(TM) i7-7700 CPU @ 3.60GHz
* Single X710 (4 physical 10G ports) (i40e)
* Traffic goes from one port, through test system and to second port on the same X710
* T-Rex traffic generator (https://trex-tgn.cisco.com/) (DPDK)
* 64-byte UDP packets

Scenario 1: l2fwd-2link on a single core running userspace and NAPI

Small amounts of packet loss starts at about 6.5M PPS unidirectional and 6.0M PPS bi-directionally (3M each direction).

Little effort has been put into optimizing this so I expect there are some easy performance wins.

## Supported Features
* HUGE TLB flag to mmap area: Optional (command line arg in the sample programs)

## Required AF_XDP Features

* Need wakeup flag (implies Linux >= 5.4)
* Aligned chunk mode

## Supported AF_XDP Features

* ZEROCOPY flag: Optional (command line arg in the sample programs)
* Aligned chunk mode

## Unsupported AF_XDP Features (for now)

* Unaligned chunk mode
* Shared Umem (new in Linux 5.10)
* Busy poll (coming in Linux 5.11)

## To Do

* Currently this module is not 'safe' because Bufs can outlive the memory pool they are associated with. I believe fixing this will require adding an Arc to each Buf. I have not had the time yet to determine the performance impact of this and would appreciate any other ideas.
* All interactions with the Tx, Rx, fill and completion queues are done with C functions that wrap the inline functions in libbpf. I believe this means that these hot functions cannot be inlined in Rust. In the medium term we should build pure Rust functions so they can be inlined.
* A more idiomatic Rust interface could be built by talking directly to the kernel vs. wrapping libbpf functions.