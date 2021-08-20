# Go_Back_N

Explanation
  - This is the code that implements bidirectional Go back N.
  - If the sender is A, the receiver must be B, and if the sender is B, the receiver must be A.
  - When sending an ack signal, always use piggyback that is sent together with data.
  - After receiving the packet normally, it tries to send an ack, but if the message is not delivered from the upper         layer, the corresponding ack cannot be sent.
  - When sending a packet after receiving a message, the packet can be sent without an ack.
  - If an ack different from the waiting sequence number is received using the sequence number of the ACK, the               corresponding ack is treated as a nak.
  - Declare and use window_buffer, which is an array as much as the window size, to store the packets sent by the           sender.
  -In this project, the upper layer is called layer5 and the lower layer is set as layer 3.
  
  
input
  1. The first input is the number of messages coming from the upper layer.
  2. The second input is the packet loss probability.
    -> In case of packet loss, it is ignored because the packets in the desired order are not delivered.
    -> If ignored, the packet can be retransmitted because it will time out later.
  3. The third input is packet corruption probability.
    -> If the packet is corrupt, the payload, seqnum and acknum of the packet may be corrupted.
    -> When calculating the checksum, the payload, seqnum, and acknum of the packet are added and stored in the checksum        of the packet.
    -> After that, when the corresponding packet is received, the checksum is calculated using the data of the packet          once more, and the checksum of the packet is checked to see if it is corrupt or not.
  4. The fourth input is the average time of messages in layer5 of the sender.
  
- code description
  1. compute_checksum(packet)
    This is a function that receives a packet and calculates the checksum using the seqnum, acknum, and payload in the       packet.
   The seqnum and acknum of the packet can also be corrupted.
    1-1. Shift seqnum by the window size and add acknum afterward.
    1-2. It calculates the check sum and calculates the payload for each 1 digit on the result to get the check sum.
    
  2. make_packet(A or B, message, an)
    - It receives A or B and determines whether A creates a packet or B creates a packet.
    - The message received in layer5 is stored in the payload of the packet.
    - The seqnum of packet is stored as next_seqnum of A or B, and acknum is initialized with the argument an that             receives the ACKnum of A or B.
    - Calculate the checksum of the information of the corresponding packet using compute_checksum, store it in the           checksum variable, and return the packet.
    
  3. A_output(message), B_output(message) are implemented as output(AorB, message)
    - It is divided into A and B, but separate A and B by defining the output function separately.
    - Receives layer5 message, converts it into packet, sends only data (ack is processed 999 times), or performs the         sender operation by piggybacking.
    - When piggybacking an ack, when the ackstate is not 0, the ack number in ACKnum is used to make a packet along with       the message contents.
    - After that, the corresponding packet is stored in window_buffer.
    - The stored packet is sent to the receiver using tolayer3.
    - If base and next_seqnum are equal, timer is started.
  
  4. A_input(packet), B_input(packet) are implemented as input(AorB, packet)
    - It is divided into A and B, but separate A and B by defining the input function separately.
    - The input function receives a packet.
    - To check whether the packet is corrupted, it calculates the checksum of the received packet.
    - If the returned checksum value is the same as the checksum value stored in the original packet, it is normal,           otherwise it is corrupted.
    4-1. If it is corrupted, the accumulated acks received so far are sent to the other party.
    4-2. If it is a normal packet, use expected_seqnum to check whether the packet is in the expected order.
      4-2-1. If it is a waiting packet, the message is delivered to the upper layer through tolayer5.
             - After that, the corresponding expected_seqnum is stored in the ACKnum and the expected_seqnum is                        incremented by 1.
      4-2-2. If an out-of-order packet is received, the same operation is performed as in the case of corrupted.
    4-3. If the acknum of the packet is not 999, check the acknum of the packet.
      4-3-1. greater than or equal to base
             - Moves the base to the next of the received acks.
                - If base and next_seqnum are the same (when ack is received for all sent packets), the timer is                           stopped.
                - If not, restart the timer (if there are still packets to receive ack).
    
  5.  A_timerinterrupt() and B_timerinterrupt() are implemented as interrupt(AorB)
    - It is divided into A and B, but separate A and B by defining the input function separately.
    - This function is called when the timer expires, and it is a function that retransmits the previously sent packet         from the base.
        -> Retransmit using the saved window_buffer while generating the packet in the output function.
        -> In the project, window is 8, and the length of window_buffer is also 8.

  6. A_init() and B_init() are implemented as initialization(AorB)
    - This function initializes the variables of A and B.
    - Base, next_seqnum, and expected_seqnum are set to 1 because they are starting points.
    - ackstate and ACKnum are initialized to 0.
    
    
 - Results and validation
  1st input is 52
  The 2nd and 3rd inputs are 0.2
  4th input is 10
  If the 5th input is set to 2
  Both the operation for packet loss and the operation for packet corrupt can be checked.
  
  
  
