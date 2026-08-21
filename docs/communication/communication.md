# Communication

The Hyper-K Trigger Timing Distributor (HKTTD) uses two communication mechanisms.
[SiTCP](https://www.sitcp.net/) connects a computer to the Main and Sub modules, while the MIKUMARI data-interface multi-channel interface (MMCI) carries data between the Main and Sub modules.

## SiTCP

SiTCP provides data transfer and register access between each module and the computer.

Transmission Control Protocol (TCP) uses a first-in, first-out (FIFO)-like interface.
The FPGA writes data to the transmit FIFO, and SiTCP transfers the data to the computer.

Remote Bus Control Protocol (RBCP) allows user software to read and write registers in the FPGA.
These registers are used to configure the FPGA and monitor its status.

HKTTD uses a 16-bit address composed of a 4-bit module ID and a 12-bit local address:

| Bits | Description |
| --- | --- |
| 15–12 | Module ID |
| 11–0 | Local address |

The register maps below list only the modules specific to HKTTD.
Registers provided by the AMANEQ platform are outside the scope of this guide.

### Main module

#### TCP FIFO configuration

The Main firmware implements one SiTCP connection.
Its TCP transmit FIFO sends the measured phase-shift value for MIKUMARI link 0 when the value changes.
Each transmitted value contains 32 bits.
The value is transmitted in big-endian byte order, with the most-significant byte first.

#### HKTTD register map

The Main `BusController` uses the following module ID assignments:
The assignments are defined in `HkttdMain/hdl/defBusAddressMap.vhd`.

| Module ID | Constant | Destination |
| --- | --- | --- |
| `0x0` | `kHPP` | `HkttdPulseEventPublisher` |
| `0x1`–`0x8` | `kHDL1`–`kHDL8` | `HkttdDeliverLink` 1–8 |

##### HkttdPulseEventPublisher registers

The `HkttdPulseEventPublisher` registers are assigned to Main module ID `0x0`.
Their local addresses are defined in `hkttd-hdl/hkttdCore/defHkttdPulseEventPublisher.vhd`.

| Local address | Access | Size | Description |
| --- | --- | --- | --- |
| `0x000` | Read/write | 1 byte | Enable the external signal pulse and dummy pulse |
| `0x100` | Read/write | 4 bytes | Dummy-pulse period |

##### HkttdDeliverLink registers

Main module IDs `0x1` through `0x8` select `HkttdDeliverLink` 1 through 8, respectively.
Each link uses the same local address map.
The local addresses are defined in `hkttd-hdl/hkttdCore/defHkttdDeliverLink.vhd`.

| Local address | Access | Size | Description |
| --- | --- | --- | --- |
| `0x000` | Write | 1 byte | Reset the signal-pulse counter |
| `0x010` | Read | 4 bytes | Signal-pulse counter |
| `0x100` | Write | 1 byte | Reset the dummy-pulse counter |
| `0x110` | Read | 4 bytes | Dummy-pulse counter |
| `0x200` | Read | 2 bytes | Transmission latency |
| `0x300` | Write | 1 byte | Reset the MIKUMARI monitoring counters |
| `0x310` | Read | 4 bytes | Lane-up counter (`counter_cbt_lane_up`) |
| `0x320` | Read | 4 bytes | MIKUMARI link-up counter |
| `0x330` | Read | 4 bytes | Pattern-error counter |
| `0x340` | Read | 4 bytes | Checksum-error counter |
| `0x350` | Read | 4 bytes | Shut-off counter |
| `0x360` | Read | 4 bytes | Measured phase shift |
| `0x370` | Read | 4 bytes | Positive phase-shift counter |
| `0x380` | Read | 4 bytes | Negative phase-shift counter |

### Sub module

#### TCP FIFO configuration

The Sub firmware implements one SiTCP connection.
Its TCP transmit FIFO sends the measured latency difference when a dummy pulse is generated.
Each transmitted value contains 16 bits.
The value is transmitted in big-endian byte order, with the most-significant byte first.

#### HKTTD register map

The Sub `BusController` uses the following module ID assignments:
The assignments are defined in `HkttdSub/hdl/defBusAddressMap.vhd`.

| Module ID | Constant | Destination |
| --- | --- | --- |
| `0x0` | `kHPS` | `HkttdPulseScheduler` |

##### HkttdPulseScheduler registers

The `HkttdPulseScheduler` registers are assigned to Sub module ID `0x0`.
Their local addresses are defined in `hkttd-hdl/hkttdCore/defHkttdPulseScheduler.vhd`.

| Local address | Access | Size | Description |
| --- | --- | --- | --- |
| `0x000` | Read/write | 2 bytes | Pulse-generation delay |
| `0x010` | Read/write | 2 bytes | Pulse width |
| `0x100` | Read | 8 bytes | Signal-pulse scaler |
| `0x110` | Read | 8 bytes | Dummy-pulse scaler |
| `0x200` | Read | 4 bytes | Pattern-error counter |
| `0x210` | Read | 4 bytes | Checksum-error counter |
| `0x220` | Read | 4 bytes | Lane-up counter (`lane_up_bus`) |
| `0x230` | Read | 4 bytes | MIKUMARI link-up counter |
| `0x300` | Write | 1 byte | Assert the `flag_pc` strobe |

The addresses in these tables are base local addresses.
Multi-byte values are accessed one byte at a time, beginning with the least-significant byte at the base address.

## MMCI

MMCI multiplexes multiple data channels over each MIKUMARI link between the Main and Sub modules.
Each channel is identified by an 8-bit address and has an assigned priority and data width.

The current channel assignments are defined in `defMmciAddress.vhd`:

| Address | Direction | Data | Transmission condition | Priority |
| --- | --- | --- | --- | --- |
| `0x00` | Main → Sub | Phase-compensated pulse timing | When a pulse is sampled and phase compensation is ready | 0 |
| `0x01` | Main → Sub | Uncompensated pulse timing | When a pulse is sampled | 0 |
| `0x02` | Main → Sub | Signal-pulse counter | When local address `0x000` of the corresponding `HkttdDeliverLink` module (`0x1`–`0x8`) is written | 1 |
| `0x03` | Main → Sub | Dummy-pulse counter | When local address `0x100` of the corresponding `HkttdDeliverLink` module (`0x1`–`0x8`) is written | 1 |

The counter channels transmit the counter value captured before the corresponding counter is cleared.
The selected `HkttdDeliverLink` module determines which Sub module receives the counter value.
The phase-compensated timing data is used for `NIM_OUT 1`, while the uncompensated timing data is used for `NIM_OUT 2`.

The MMCI implementation supports channels in both directions.
All HKTTD channels currently defined in `defMmciAddress.vhd` transmit data from the Main module to the Sub module.
