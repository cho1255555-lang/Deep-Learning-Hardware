# Tiny-YOLO 추론 가속기 설계

> 2025 Deep Learning Hardware 설계 경진대회 [장려상] | 3인 팀 프로젝트 | 2025.02 ~ 2025.05 |
>
> https://www.disu.ac.kr/LOCnLOS/AIX/notice?md=v&bbsidx=8032 

### Overview
무인 판매대 상품 인식을 위한 Tiny-YOLO 기반 추론 가속기를 설계한 프로젝트입니다. FPGA 자원 안에서 주어진 모델의 추론 가속기를 효과적으로 구현하는 것을 목표로 BRAM 활용을 극대화하고 외부 메모리 Access 접근을 최소화에 집중했습니다.

### Language and Tools
- Language: Verilog
- Tools: Xilinx Vivado
- FPGA: Arty A7-100T 

# Architecture
<img width="1059" height="540" alt="image" src="https://github.com/user-attachments/assets/30445d88-0cae-4e2b-93d8-0a087e668948" />

**Design Concepts**
1. **IFM BRAM 활용 극대화**: IFM을 채널별로 64개 BRAM에 분산 저장하여 MAC이 읽는 BRAM과 데이터를 쓰는 BRAM이 겹치지 않도록 구성
2. **reg_array 이중화**: 두 개의 reg_array를 번갈아 사용해 다음 데이터를 미리 로드함으로써 MAC 입력이 끊기지 않게 유지
4. **Weight BRAM Ping/Pong**: 32필터씩 연산하는 동안 나머지 32필터를 동시에 로딩하여 대기 시간 제거
5. **IFM BRAM 저장 영역 분리**: IFM BRAM을 상하로 나눠 입력과 출력 저장 위치가 겹치지 않도록 처리

# My Role

**기초 Verilog 문법 교육**
- RTL 설계를 처음 접한 팀원을 위해 기초 Verilog 문법 교육 진행

**BRAM Controller 설계**

- Skeleton 코드 시뮬레이션에서 중간 Layer의 IFM을 외부 메모리에 저장했다가 다시 읽어오는 구조를 확인하고 IFM은 내부 BRAM에서만 처리하고 외부 BUS는 Weight 공급에만 사용하는 구조를 제안하여 팀 설계에 반영
- MAC 연산 결과를 다음 Layer의 IFM 형태로 정렬해 BRAM에 저장하는 Controller 설계
- MAC 연산 담당 팀원과 valid 신호 기반으로 연산 결과 저장 타이밍 조율
- Weight 전송 담당 팀원과 valid 신호에 따른 Weight 저장 BRAM 선택 방식 조율

# Result

- Quantization 정확도: **mAP 81.76%**
- 추론 시간: **48.38ms @ 100MHz** (제어 신호 기준 RTL 시뮬레이션)

<img width="1641" height="704" alt="image" src="https://github.com/user-attachments/assets/b8aaed21-1077-4d28-898a-c7486a2d0d66" />


# Technical Challenges & Solutions
 
### Problem 1. 외부 메모리 접근으로 인한 추론 시간 증가
**문제 상황**
 
Skeleton 코드는 MAC 연산이 끝날 때마다 다음 Layer의 IFM을 외부 RAM에 Write했다가 다음 Layer 연산 때 다시 Read하는 구조임. 따라서 추론 효울이 떨어지게 됨. Simulation waveform에서 초기 Input 전송에 약 0.5ms, 외부 RAM 왕복에 1ms의 지연이 발생하는 것을 확인함
 
**분석**
 
Layer별 데이터 크기를 분석한 결과 IFM은 Layer가 깊어질수록 채널은 늘지만 해상도가가 줄어 전체 크기가 작아짐(256x256x16 → 128x128x32 → 64x64x64). 반면 Weight는 특정 Conv Layer에서 급격히 커짐. 따라서 IFM은 내부 BRAM으로 감당할 수 있고 외부 메모리 대역폭은 Weight 공급에 써야 한다고 판단함
 
**해결**
 
BUS는 최초 Input IFM 전송을 제외하면 Weight만 외부 메모리에서 Read하고 중간 IFM은 모두 내부 BRAM에 저장하도록 구조를 변경함. 이때 MAC이 현재 IFM을 읽고 있는 BRAM에는 다음 Layer의 IFM을 쓸 수 없기 때문에 MAC 출력 타이밍에 맞춰 다음 IFM을 다른 BRAM 영역에 Addressing해서 저장하는 BRAM Controller를 설계함

- 초반 Layer: IFM이 IFM BRAM을 가득 채우므로 Weight가 작아 여유가 있는 Weight BRAM에 임시 저장 후 IFM BRAM으로 이동
- 이후 Layer: IFM이 충분히 작아지면 IFM BRAM을 Ping/Pong으로 나눠 읽는 영역과 쓰는 영역을 분리해 바로 저장

**결과**
 
IFM 메모리접근 시간 제거 및 연산 효율을 높임

### Problem 2. 모듈 통합 과정에서 Port 연결 오류
**문제 상황**

개별 모듈을 Top에서 통합하는 과정에서 Port 연결 실수가 반복적으로 발생함

**분석**

팀원마다 Port 네이밍 방식이 달라 신호 이름만으로는 해당 Port가 모듈의 입력인지 출력인지 구분하기 어려웠고 이로 인해 방향을 잘못 연결하는 실수가 많았음

**해결**

신호 이름 앞에 `i_`, `o_` 접두어를 붙여 Port의 방향을 나타내는 Code Convention을 수립했고 모든 모듈의 Port 네이밍을 이 규칙으로 통일함

**결과**

Port 이름만으로 입출력 방향을 바로 알 수 있게 되어 연결 실수를 줄였고 통합 후 제어 신호 기준 RTL 시뮬레이션으로 전체 Layer의 연산 흐름을 확인해 48.38ms의 연산 시간을 도출함

# Limitations

- Layer별 IFM을 Reference data와 비교할 때 waveform에서 값을 눈으로 하나씩 확인하는 방식으로 진행하여 모든 Layer의 출력을 정확하게 검증하지 못함
- 모듈별로 Synthesis를 먼저 확인하지 않고 Integration 이후에 Synthesis를 진행하여 합성 오류를 해결하지 못하고 Synthesis를 완료하지 못함
- 프로젝트 후반부 Weight Ping/Pong 구조와 Convolution 모듈 간 연결에서 문제가 발생해 해당 구조를 완전히 적용하지 못했고 그것으로 인해 연산 지연이 발생함
