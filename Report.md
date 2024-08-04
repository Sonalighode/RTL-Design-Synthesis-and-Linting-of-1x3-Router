<h1> RTL Design, Synthesis and Linting of 1x3 Router </h1>

<h2> Introduction </h2>

- **1x3 Router Design:** Routes packets from a single source LAN to one of three destination LANs.
- **Packet-based protocol:** Receives network packets byte by byte on the rising edge of the clock.
- **Synchronous reset:** Uses an active low synchronous reset signal (resetn).
- **Packet indication:** Start of a new packet indicated by asserting 'pkt_valid'; end indicated by de-asserting 'pkt_valid'.
- **FIFO storage:** Incoming packets are stored in one of three FIFOs based on the packet's address, corresponding to different destination LANs.
- **Read operation:** Destination LANs read packets using 'data_out_x' channels, monitored by 'vld_out_x' and controlled by 'read_enb_x'.
- **Busy state:** Indicated by a 'busy' signal, informing the source LAN to wait before sending the next byte.
- **Error detection:** Parity check mechanism for packet integrity; mismatch triggers an 'error' signal, prompting the source LAN to resend the packet.
- **Simultaneous reads:** Can receive one packet at a time but supports simultaneous reading of up to three packets.
- **Use:** Ensures efficient and error-checked data transmission between a source LAN and multiple destination LANs.

<h2> Block diagram </h2>

![image](https://github.com/user-attachments/assets/c76c5764-0949-4e0d-92f7-29ca48114ea6)

- **clock:** Active high clocking event.
- **pkt_valid:** Active high input signal that detects the arrival of a new packet from a source network.
- **resetn:** Active low synchronous reset.
- **data_in:** 8-bit input data bus that transmits the packet from the source network to the router.
- **read_enb_0, read_enb_1, read_enb_2:** Active high input signals for reading the packet through respective output data buses ('data_out_0', 'data_out_1', 'data_out_2').
- **data_out_0, data_out_1, data_out_2:** 8-bit output data buses that transmit the packet from the router to destination client networks 1, 2, and 3, respectively.
- **vld_out_0, vld_out_1, vld_out_2:** Active high signals that detect when a valid byte is available for destination client networks 1, 2, and 3, respectively.
- **busy:** Active high signal that detects a busy state for the router, stopping it from accepting any new byte.
- **error:** Active high signal that detects a mismatch between the packet parity and the internal parity.

<h2> Packet format </h2>

![image](https://github.com/user-attachments/assets/cd0aa39d-ef98-446d-9a1b-68815a910dbf)

**Packet Parts:** Header, Payload, Parity.
**Header Fields:**
DA: 2 bits, directs packet to correct port.
Length: 6 bits, indicates payload size.
**Payload:** Data information, 1 to 63 bytes.
**Parity:** Ensures data integrity, bitwise parity of header and payload.

<h2> Sub-blocks </h2>

1. 3 FIFOs
2. Synchronizer
3. Register
4. FSM

<h2> FIFO </h2>

![image](https://github.com/user-attachments/assets/58a49db2-2af3-4923-9f04-df6bbb1e526f)

1. **Reset Mechanism**
   - Synchronous active low reset and internal `soft_reset`.

2. **FIFO Memory Size**
   - 16 x 9 (extra bit for header detection).

3. **Write Operation**
   - Sampled at rising clock edge; requires `write_en` high and FIFO not full.

4. **Read Operation**
   - Data read at rising clock edge; requires `read_en` high and FIFO not empty.

5. **Header Detection**
   - Extra bit for header byte detection.

6. **FIFO Status Indicators**
   - `full` indicates all locations written; `empty` indicates all locations read.
  
<h3> Output </h3>

![image](https://github.com/user-attachments/assets/34438c75-b782-4ad2-afba-41694b169f50)

<h2> Synhronizer </h2>

![image](https://github.com/user-attachments/assets/4614437c-037e-4ba8-b486-14d22fc5e528)

1. **Synchronization Purpose**: Ensures communication between input port and three output ports.
2. **FIFO Selection**: 'detect_add' and 'data_in' signals select a FIFO for routing.
3. **FIFO Full Signal**: 'fifo_full' based on 'full_0', 'full_1', or 'full_2' status.
4. **Valid Output Signal**: 'vld_out_x' based on 'empty_x' status.
5. **Write Enable Signal**: 'write_enb_reg' generates 'write_enb' for the selected FIFO.
6. **Internal Reset Signals**: 'soft_rst_x' activated if 'read_enb_X' not asserted within 30 clock cycles of 'vld_out_X' assertion.

<h3> Output </h3>

![image](https://github.com/user-attachments/assets/9de79e1e-f334-48f2-a89e-06967ed76318)

<h2> FSM </h2>

![image](https://github.com/user-attachments/assets/02ec290b-48b0-4c52-abe0-f3571e85d61d)

![image](https://github.com/user-attachments/assets/f5910c8a-daaf-4c13-8c4a-e076675d3c23)

1. **FSM Purpose**: Controls ROUTER, generating signals to transfer packets to output ports.
2. **STATE-DECODE_ADDRESS**: Initial state, 'detect_add' latches the header byte.
3. **STATE-LOAD_FIRST_DATA**: 'Ifd_state' loads first data byte; 'busy' prevents header update.
4. **STATE-LOAD_DATA**: 'id_state' loads payload; 'write_enb_reg' writes to FIFO; transitions based on 'pkt_valid' and FIFO status.
5. **STATE-LOAD_PARITY**: Latches the parity byte; transitions to CHECK_PARITY_ERROR; 'busy' prevents new data.
6. **STATE-FIFO_FULL**: 'busy' high, 'write_enb_reg' low, detects FIFO full.
7. **STATE-LOAD_AFTER_FULL**: 'laf_state' latches post-full data; transitions based on 'parity_done' and 'low_pkt_valid'.
8. **STATE-WAIT_TILL_EMPTY**: 'busy' high, 'write_enb_reg' low.
9. **STATE-CHECK_PARITY_ERROR**: 'rst_int_reg' resets 'low_pkt_valid'; transitions based on FIFO status; 'busy' asserted.
10. **Soft-Reset Usage**: Returns to DECODE ADDRESS for packet timeout situations.

<h3> Output </h3>

![image](https://github.com/user-attachments/assets/1b67818a-f6b4-42d1-9072-733e40a6b91b)

<h2> Register </h2>

![image](https://github.com/user-attachments/assets/3cfb4cbc-9a91-49bb-ac72-a275270dd438)

1. **Registers**: Holds header byte, FIFO full state byte, internal parity, and packet parity byte.
2. **Clock Edge**: Registers latched on rising clock edge.
3. **Reset Condition**: 'resetn' low sets 'dout', 'err', 'parity_done', and 'low_pkt_valid' to low.
4. **Parity Done Conditions**:
   - 'Id_state' high and 'fifo_full' low.
   - Both 'laf_state' and 'low_pkt_valid' high with previous 'parity_done' low.
5. **Reset Signal**: 'rst_int_reg' resets 'low_pkt_valid'.
6. **Detect Add Signal**: Parity done signal is used.
7. **Low Packet Valid**: High when 'Id_state' high and 'pkt_valid' low.
8. **Header Byte Latching**: 'detect_add' and 'pkt_valid' high latch the first byte.
9. **Internal Parity Calculation**: Bitwise XOR of header, payload bytes, and previous parity.
10. **Error Calculation**: 'err' set if packet parity doesn't match internal parity.

<h3> Output </h3>

![image](https://github.com/user-attachments/assets/defe01dc-f570-4624-b10c-bea8064bd7c2)

<h2> Top block </h2>

![image](https://github.com/user-attachments/assets/03c9d772-1dbb-432f-b0e4-3ff1617198be)

<h3> Output </h3>

![image](https://github.com/user-attachments/assets/29e642f9-4709-4bc1-80d8-23349d02546d)

<h1> Synthesis </h1>

![image](https://github.com/user-attachments/assets/f1546174-f088-4907-a71c-b6e9e2448bcc)

![image](https://github.com/user-attachments/assets/9ceebd55-c0b3-4d27-a123-ed93f3a05cc1)

![image](https://github.com/user-attachments/assets/96a8c730-4387-4963-8ebf-9df90d120d70)















  








