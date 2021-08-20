# Go_Back_N

- 설명
  bidirectional Go back N을 구현한 코드이다.
  sender가 A라면 receiver가 B이고, sender가 B라면 receiver는 A가 되어야 한다.
  ack신호를 보낼 때는 항상 data와 같이 보내는 piggyback을 사용한다.
  packet을 정상적으로 수신한 뒤 ack를 보내려고 하는데, 상위 계층에서 message가 전달되지 않는다면 해당 ack를 보낼 수 없다.
  message를 받은 뒤 패킷을 보내는 경우에는 ack가 없어도 패킷을 전송할 수 있다.
  ACK의 sequence number를 사용해 기다리는 sequence number와 다른 ack가 수신되면 해당 ack를 nak로 취급한다.
  sender의 보낸 packet들을 저장해두기 위해서 window size만큼의 배열인 window_buffer를 선언해 사용한다.
  해당 프로젝트에서는 상위 계층을 layer5라고 하며 하위 계층을 layer 3이라고 설정했다
  
  
- 입력
  1. 첫 번째 입력은 상위계층에서 오는 message의 개수
  2. 두 번째 입력은 packet loss probability.
    -> packet loss경우, 원하는 순서의 packet이 전달되지 않기 때문에 무시한다.
    -> 무시하면 이후에 time out이 되기 때문에 해당 패킷을 재전송할 수 있다.
  3. 세 번째 입력은 packet corruption probability.
    -> packet이 corrupt 경우, packet의 payload, seqnum과 acknum가 손상될 수 있다.
    -> check sum을 계산할 때에는 패킷의 payload, seqnum, acknum을 추가해 패킷의 checksum에 저장한다. 
    -> 이후 해당 패킷을 받으면 패킷의 data를 이용해 check sum을 한번 더 계산한 후, 패킷이 갖은 checksum과 같은지 확인해
       corrupt가 일어났는지 아닌지 확인한다.
  4. 네 번째 입력은 sender의 layer5의 message 평균 시간이다.
  
- code description
  1. compute_checksum(packet)
    packet을 전달받아 해당 packet에 있는 seqnum과 acknum, payload를 사용해 check sum을 구하는 함수이다.
    packet의 seqnum과 acknum도 corrupted될 수 있다.
    1-1. seqnum을 window 크기만큼 shift한 뒤 뒤에 acknum을 더한다.
    1-2. check sum연산을 해주고 그 결과에 payload 각각 1자리에 대해 연산을 진행해 check sum을 구해준다.
    
  2. make_packet(A or B, message, an)
    A or B를 전달받아 A가 packet을 만드는지 B가 packet을 만드는지 구분한다.
    layer5에서 전달받은 message를 packet의 payload에 저장한다.
    packet의 seqnum은 A 또는 B의 next_seqnum으로 저장하고, acknum은 A또는 B의 ACKnum을 받은 인자 an으로 초기화한다.
    compute_checksum을 이용해 해당 packet의 정보의 checksum을 계산해 checksum변수에 저장해두고 packet을 반환한다.
    
  3. A_output(message), B_output(message)은 output(AorB, message)으로 구현
    A와 B로 구분이 되어있지만, output함수를 별도로 정의해 A, B를 구분한다.
    layer5의 message를 받아 packet으로 변환해서 data만 보내거나(ack는 999번 처리), piggyback해서 보내는 sender의 동작을 수행한다.
    ack를 piggyback할 때, ackstate가 0이 아닌 상태에서 ACKnum에 있는 ack번호를 사용해 message 내용과 함께 packet으로 만든다
    이후에 window_buffer에 해당 패킷을 저장한다.
    저장된 packet은 tolayer3를 사용해 receiver에게 보낸다.
    base와 next_seqnum이 같다면 timer를 시작한다.
  
  4. A_input(packet), B_input(packet)은 input(AorB, packet)으로 구현
    A와 B로 구분이 되어있지만, input함수를 별도로 정의해 A, B를 구분한다.
    input함수는 packet을 받는다.
    packet이 corrupted인지 확인하기 위해 받은 패킷의 checksum계산한다.
    반환된 checksum값과 원래 packet에 저장되어 있는 checksum값이 같으면 정상이며, 다르면corrupted이다.
    4-1. corrupted라면 상대방에게 현재까지 받은 누적 ack를 보내준다.
    4-2. 정상 패킷이라면 expected_seqnum을 이용해 기다리던 순서의 패킷인지 확인한다.
      4-2-1. 기다리던 패킷이라면 tolayer5를 통해 상위계층에 message를 전달한다.
             이후에 ACKnum에 해당 expected_seqnum을 저장하고 expected_seqnum을 1증가시킨다.
      4-2-2. 만약 순서가 잘못된 패킷이 온다면 corrupted된 경우와 동일한 동작을 한다.
    4-3. packet의 acknum이 999가 아니라면, 패킷의 acknum을 확인한다.
      4-3-1. base보다 크거나 같은 경우
             받은 ack의 다음으로 base를 이동시킨다.
                base와 next_seqnum이 같은 경우 (보낸 모든 packet에 대해 ack를 받은 경우)timer를 멈춘다. 
                아니라면 (아직 ack를 받을 패킷이 남은 경운) timer를 재시작한다. 
    
  5.  A_timerinterrupt(), B_timerinterrupt()은 interrupt(AorB)로 구현
    A와 B로 구분이 되어있지만, input함수를 별도로 정의해 A, B를 구분한다.
    timer가 만료됐을 때 호출되는 함수로, 이전까지 보냈던 packet을 base에서부터 재전송하는 함수이다.
    -> output함수에서 packet을 생성하면서 저장해둔 window_buffer를 사용해 재전송한다.
    -> 프로젝트에서 window는 8으로 window_buffer의 길이도 8이다. 
    
  6. A_init(), B_init()은 initialization(AorB)으로 구현
    A, B의 변수를 초기화 하는 함수이다.
    base, next_seqnum, expected_seqnum은 시작점이기 때문에 1으로 설정했다.
    ackstate와 ACKnum은 0으로 초기화해 시작한다.
    
    
 - 결과 및 검증
  1번째 입력은 52
  2번째, 3번째 입력은 0.2
  4번째 입력은 10
  5번째 입력은 2로 설정하면
  packet loss에 대한 동작, packet corrupt에 대한 동작 모두 확인할 수 있다.
  
  
  
