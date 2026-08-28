# Betaflight 4.5.5 개인 수정 내역

## 기록 정보

| 항목 | 내용 |
| --- | --- |
| 기준 버전 | Betaflight 4.5.5 (`4adbd3e`) |
| 최초 개인 수정 커밋 | `eef1e10` — `Add custom firmware changes` |
| 변경 기록일 | 2026-08-28 |

`2026-08-28`은 이 저장소에서 수정사항을 최초로 Git 커밋으로 기록한 날짜다. 커밋 이전부터 작업 폴더에 수정이 존재했으므로, 각 코드 줄의 실제 작성일은 현재 Git 이력만으로 판별할 수 없다.

## 변경 목적

기존 RC 입력 대신 외부 장치가 MSPv2로 전송하는 명령으로 기체를 제어할 수 있게 한다. Configurator의 Modes 탭에 별도로 표시되는 `MSP CTBR` 또는 `MSP RPM` 비행 모드가 활성화되었을 때만 해당 제어가 적용된다. 마지막으로 수신한 MSP 명령이 아니라 활성화된 AUX 모드가 제어 방식을 결정한다.

## 추가된 MSPv2 명령

### `MSP_CTBR` (`0x30F0`)

새 명령은 정확히 16바이트의 little-endian IEEE 754 `float` payload를 받아야 한다. 길이가 16바이트가 아니거나 `NaN`/무한대 값이 포함되면 명령은 오류로 거절된다.

| 순서 | 형식 | 원시값 의미 | 펌웨어 내부 값 |
| --- | --- | --- | --- |
| 1 | `float` | normalized collective thrust | 최종 범위 `0.0–1.0` |
| 2 | `float` | roll body rate (`rad/s`) | deg/s로 변환 후 `-1998–1998 deg/s`로 제한 |
| 3 | `float` | pitch body rate (`rad/s`) | deg/s로 변환 후 `-1998–1998 deg/s`로 제한 |
| 4 | `float` | yaw body rate (`rad/s`) | deg/s로 변환 후 `-1998–1998 deg/s`로 제한 |

예를 들어 collective thrust 50%는 첫 필드에 `0.5f`를 전송한다. Roll `2.0 rad/s`는 Roll 필드에 `2.0f`를 전송한다. 명령은 응답 payload 없이 처리된다.

`MSP2_SET_RATE_THRUST`는 이전 이름과의 소스 호환성을 위해 같은 `0x30F0` 값의 별칭으로 남아 있다. 단, `0x30F0`의 wire payload는 기존 8바이트 정수 형식에서 이 16바이트 float 형식으로 변경되었으므로 기존 외부 송신기는 반드시 함께 수정해야 한다.

### `MSP_RPM` (`0x30F1`)

`MSP_RPM`은 모터별 기계적 RPM 목표값을 직접 지정한다. Payload 길이는 현재 기체의 모터 수에 정확히 일치해야 하며, 각 값은 little-endian 부호 없는 32-bit 정수(`uint32_t`)다. RPM은 음수가 될 수 없으므로 unsigned integer를 사용한다.

| 기체 모터 수 | Payload 길이 | Payload 순서 |
| --- | --- | --- |
| Quad (4) | 16 bytes | Motor 1, 2, 3, 4의 RPM |
| Hexa (6) | 24 bytes | Motor 1–6의 RPM |
| Octo (8) | 32 bytes | Motor 1–8의 RPM |

각 RPM 값은 `0–4294967295 rpm` 범위의 정수이며, 펌웨어는 보드/모터 설정에서 계산한 추정 최대 RPM보다 높은 값은 그 최대값으로 제한한다. 값 `0`은 해당 모터의 출력을 disarm 값으로 설정한다. 이 명령도 응답 payload 없이 처리된다.

RPM 모드에서는 DShot telemetry가 **모든 활성 모터**에서 유효해야 한다. telemetry가 없거나 하나라도 유효하지 않으면 일반 mixer나 RC 제어로 되돌아가지 않고, 모든 모터 출력을 disarm 값으로 강제한다.

## 제어 동작

1. `MSP_CTBR`를 수신하면 추력과 세 축의 목표 각속도, 수신 시각을 저장한다.
2. `MSP CTBR` 모드가 활성화되면 `getSetpointRate()`가 RC 스틱 값 대신 저장된 외부 rate를 반환하고, 모터 믹서는 저장된 외부 thrust를 사용한다.
3. `MSP_RPM`을 수신하면 모터별 목표 RPM과 수신 시각을 저장한다. `MSP RPM` 모드가 활성화되어야 이 값이 사용된다.
4. RPM 상태에서는 일반 자세 PID mixer의 최종 모터 출력을 사용하지 않고, 모터별 RPM PI 제어기의 출력으로 교체한다. PI 제어기는 목표 RPM/추정 최대 RPM의 feed-forward에 현재 DShot RPM 오차의 비례·적분 보정을 더한다.
5. 마지막 외부 명령 후 100 ms(`100000 us`)가 지나면 명령은 만료된다. CTBR은 rate와 thrust를 `0`으로 처리하며, RPM 모드는 모든 모터를 disarm 값으로 설정한다.

외부 제어 모드가 아닌 경우 기존 RC 기반 동작을 유지한다.

`MSP CTBR`과 `MSP RPM`을 동시에 활성화하면 RPM 모드가 우선된다. 의도하지 않은 모드 전환을 막기 위해 Configurator에서 두 AUX 범위가 겹치지 않게 설정해야 한다.

## 함께 바뀐 동작

* 외부 제어 모드에서는 feedforward 출력을 `0`으로 만들어 외부 rate 명령에 RC feedforward가 섞이지 않게 했다.
* 외부 제어 모드에서는 `FEATURE_MOTOR_STOP` 조건을 적용하지 않는다. 따라서 ARM 상태에서 외부 제어가 활성화된 경우 `MOTOR_STOP`이 모터 출력을 별도로 차단하지 않는다.
* `MSP CTBR`과 `MSP RPM`은 `msp_override_channels_mask` 값과 관계없이 활성 Box 목록에 등록된다.
* 설정 검증 단계에서 override 채널 마스크가 비어 있다는 이유로 두 MSP 모드의 활성화 조건을 자동 삭제하지 않도록 했다.
* `MSP_RPM`은 DShot telemetry와 ARM 상태가 모두 필요하다. 명령 만료, telemetry 비활성, telemetry 무효, disarm, 또는 추정 최대 RPM 오류 시에는 모든 모터 출력을 disarm 값으로 만든다.

## 관련 소스 파일

| 파일 | 변경 내용 |
| --- | --- |
| `src/main/msp/msp_protocol_v2_betaflight.h` | `MSP_CTBR` 및 `MSP_RPM` 명령 ID 정의 추가 |
| `src/main/msp/msp.c` | CTBR float 16바이트 payload와 모터별 32-bit integer RPM payload 파싱, 외부 제어 값 전달 추가 |
| `src/main/fc/rc.h` | 외부 제어 API 선언 추가 |
| `src/main/fc/rc.c` | 별도 CTBR/RPM Box 모드, 외부 입력 저장, 범위 제한, 모드별 100 ms 만료 확인, rate 대체 및 feedforward 비활성화 추가 |
| `src/main/flight/mixer.c` | 외부 thrust 사용, 모터별 RPM PI 폐루프, telemetry failsafe 및 외부 제어 중 `MOTOR_STOP` 예외 처리 |
| `src/main/msp/msp_box.c` | `MSP CTBR` 및 `MSP RPM` Box의 상시 등록 |
| `src/main/config/config.c` | override 모드 조건 자동 삭제 방지 |

## 안전상 주의

외부 제어 명령은 실제 모터 출력에 영향을 준다. 송신 장치는 100 ms보다 충분히 짧은 주기로 명령을 보내야 하며, 링크 단절·프로세스 정지·잘못된 단위·잘못된 엔디언 처리 상황을 포함한 failsafe를 비행 전에 프로펠러를 분리한 상태에서 검증해야 한다. RPM 모드에서는 ESC가 bidirectional DShot telemetry를 안정적으로 제공해야 하며, 모터 pole count 설정이 실제 모터와 일치해야 한다. 명령 만료 시 추력은 0이 되지만, 외부 제어 중에는 `MOTOR_STOP`이 적용되지 않는다는 점을 고려해야 한다.

## MSP 수신 주기

`TASK_SERIAL`의 목표 주기는 2026-08-29에 100 Hz에서 **2 kHz**로 변경했다. 이 값은 CLI에서 설정하는 항목이 아니라 펌웨어 scheduler의 컴파일 타임 설정이므로, 변경된 펌웨어를 빌드하고 비행 컨트롤러에 플래시해야 적용된다. MSP 수신 task는 한 실행에서 패킷 하나를 처리하므로, CTBR/RPM 명령과 `MSP_STATUS` 같은 조회 명령을 합친 전체 패킷 빈도를 기준으로 계산해야 한다.

2 kHz는 목표 주기이며, 실제 주기는 낮은 scheduler 우선순위, CPU 부하, UART/USB 대역폭, 호스트 송신 지터에 따라 달라질 수 있다. Quad 기준 CTBR/RPM MSPv2 프레임은 약 25 bytes이므로, 2 kHz 연속 전송에는 UART 1 Mbaud 이상을 권장한다.

## `src/config` 서브모듈 상태

상위 저장소의 `src/config` gitlink는 4.5.5 기준 `92c695a`에서 `e95b1db`로 이동했다. 이는 Betaflight 공식 `config` 저장소의 더 최신 커밋을 가리키는 변경이며, 해당 서브모듈 전체의 여러 공식 보드 설정 변경을 함께 포함한다.

또한 현재 작업 폴더의 `src/config/configs/MICOAIR743V2/config.h`는 서브모듈 안의 **추적되지 않은 파일**이다. 이 파일은 상위 저장소의 개인 커밋에 포함되지 않았으며, 이 저장소만 새로 clone해도 자동으로 복원되지 않는다. 이 보드 설정도 재현 가능하게 관리하려면 `config` 서브모듈을 개인 fork로 만들고 해당 파일을 별도 커밋 및 푸시해야 한다.
