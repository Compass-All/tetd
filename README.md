# TETD: Trusted Execution in Trust Domains

This repository implements a full-stack prototype of **TETD (Trusted Execution in Trust Domains)**. TETD is a lightweight framework for securely running agents inside Intel TDX Trust Domains (TDs), with coordinated VMM-side logic. It supports both **exclusive** and **collaborative** execution modes for TD-resident agents.

The prototype consists of two main components:
- `TETD-G`: a Linux kernel module running inside the guest TD, managing agent loading, relocation, and secure communication with the VMM.
- `TETD-H`: a KVM-based hypervisor extension running on the host side, responsible for managing agent memory, handling TDCALLs, and coordinating agent invocations.

---

## Guest-Side (TETD-G) Overview

The TETD-G kernel module manages the agent lifecycle inside the TD:
- **Exclusive Mode**: Agent runs directly inside TD, coordinating with VMM via TDCALL.
- **Collaborative Mode**: Agent relocates itself to a secret zone, with a custom page table, GDT, and IDT. Executed in an isolated enclave-like environment.

The module provides runtime debugging via `/proc/tetdg_info` and supports:
- Encrypted agent arguments/results
- Dynamic agent registration
- Support for a privileged **admin agent** to inspect/control other agents



## Agent Development

TETD agents are lightweight binaries that register entry points with the TETD framework. To write a new agent:

1. Define an `agent_init()` function:

   ```c
   void agent_init(void)
   {
       tetdg_agent_call_register(1, &my_agent_handler);
   }
   ```

2. Implement the handler:

   ```c
   int my_agent_handler(void *args) {
       // process arguments, do work
       return 0;
   }
   ```

3. Add your agent to `agents/Makefile`.

4. Load the compiled agent into the TD along with the `tetdg` kernel module.

------

## Host-Side (TETD-H) Overview

TETD-H extends the KVM module to:

- Allocate and manage secret zones via CMA
- Intercept TDCALLs from TETD-G
- Maintain agent tables per TD
- Dynamically block/unblock pages via SEAMCALL
- Support `eventfd/io_uring` for agent call notification

### Key VMM Features

- Handles leaf functions like:
  - `TDCALL_FN_VMM_ENTER`, `TDCALL_FN_VMM_RESUME`
  - `TDCALL_FN_AGENT_ACCESS`, `TDCALL_FN_AGENT_DISABLE`
- Supports SEAMCALLs:
  - `TDH.MEM.PAGE.AUG`, `TDH.MEM.PAGE.REMOVE`
  - `TDH.MEM.RANGE.BLOCK`, `TDH.MEM.TRACK`

------

## Enabling SGX-SSL for Agent Encryption

To support agent-side encryption:

1. Download and build SGX-SSL, and place it under `sgx-ssl/`
2. Enable `ENCRYPT_REG` macro in your agent or tetdg compile-time flags
3. TETD will use the crypto APIs to encrypt all agent arguments and results

------

## Security Notes

- The guest kernel is considered untrusted; TETD enforces memory and control-flow isolation.
- Agent relocation uses independent page tables with masked interrupts.
- Communication with the VMM is only via TDCALL, minimizing the attack surface.

------

## License

This prototype is released under the MIT License. See `LICENSE` for details.

------

## Contact

For research inquiries or integration into your TDX environment, please contact the maintainer or open an issue.