# TCP vs UDP

## Overview
The transport layer protocol determines how packets are delivered.

## 1. TCP (Transmission Control Protocol)
Connection-oriented, reliable delivery.
*   **Mechanisms**: Three-way handshake, Acknowledgements (ACK), Retransmission, Sequencing, Flow Control.
*   **Pros**:
    *   **Reliability**: Guarantees data arrives effectively and in order.
    *   **Congestion Control**: Prevents network flooding.
*   **Cons**:
    *   **Overhead**: Larger headers, Handshake latency.
    *   **Head-of-Line Blocking**: One lost packet delays the entire stream.
*   **Use Cases**: Web (HTTP), Email (SMTP), File Transfer (FTP), Shell (SSH).

## 2. UDP (User Datagram Protocol)
Connectionless, "Fire and Forget".
*   **Mechanisms**: Sends packets without checking if receiver is ready or exists.
*   **Pros**:
    *   **Speed**: No handshake, no ACK overhead.
    *   **Efficiency**: Smaller headers.
    *   **Real-time**: If a packet is lost, just send the next one. No waiting.
*   **Cons**:
    *   **Unreliable**: Packets can be lost, duplicated, or out of order.
*   **Use Cases**: Video Streaming (Netflix/YouTube use UDP-based QUIC), Online Gaming (Position updates), DNS, VoIP.

## QUIC (HTTP/3)
A modern protocol built on top of UDP to fix TCP's Head-of-Line blocking while maintaining reliability at the application layer.
