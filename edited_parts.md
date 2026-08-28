# Betaflight 4.5.5 개인 수정 내역

## 기록 정보

| 항목 | 내용 |
| --- | --- |
| 기준 버전 | Betaflight 4.5.5 (`4adbd3e`) |
| 개인 수정 커밋 | `eef1e10` — `Add custom firmware changes` |
| 변경 기록일 | 2026-08-28 |

`2026-08-28`은 이 저장소에서 수정사항을 최초로 Git 커밋으로 기록한 날짜다. 커밋 이전부터 작업 폴더에 수정이 존재했으므로, 각 코드 줄의 실제 작성일은 현재 Git 이력만으로 판별할 수 없다.

## 변경 목적

기존 RC 입력 대신 외부 장치가 MSPv2로 전송하는 **추력(thrust)** 및 **각속도(rate) 명령**으로 기체를 제어할 수 있게 한다. 외부 제어는 `BOXMSPOVERRIDE` 비행 모드가 활성화되었을 때만 적용된다.

## 추가된 MSPv2 명령

### `MSP2_SET_RATE_THRUST` (`0x30F0`)

새 명령은 정확히 8바이트의 payload를 받아야 한다. 길이가 8바이트가 아니면 명령은 오류로 거절된다.

| 순서 | 형식 | 원시값 의미 | 펌웨어 내부 값 |
| --- | --- | --- |
| 1 | `uint16_t` | thrust | `raw / 10000.0`, 최종 범위 `0.0–1.0` |
| 2 | `int16_t` | roll rate | `raw × 0.1 deg/s`, 최종 범위 `-1998–1998 deg/s` |
| 3 | `int16_t` | pitch rate | `raw × 0.1 deg/s`, 최종 범위 `-1998–1998 deg/s` |
| 4 | `int16_t` | yaw rate | `raw × 0.1 deg/s`, 최종 범위 `-1998–1998 deg/s` |

예를 들어 thrust를 50%로 설정하려면 첫 필드에 `5000`을 전송한다. Roll을 `120 deg/s`로 설정하려면 Roll 필드에 `1200`을 전송한다. 명령은 응답 payload 없이 처리된다.

## 제어 동작

1. MSP 명령을 수신하면 추력과 세 축의 목표 각속도, 수신 시각을 저장한다.
2. `BOXMSPOVERRIDE`가 활성화된 경우 `getSetpointRate()`가 RC 스틱 값 대신 저장된 외부 rate를 반환한다.
3. 모터 믹서는 계산한 스로틀 대신 저장된 외부 thrust를 사용한다.
4. 마지막 수신 후 100 ms(`100000 us`)가 지나면 명령은 만료된다. 만료된 상태에서는 rate와 thrust가 모두 `0`이 된다.

외부 제어 모드가 아닌 경우 기존 RC 기반 동작을 유지한다.

## 함께 바뀐 동작

* 외부 제어 모드에서는 feedforward 출력을 `0`으로 만들어 외부 rate 명령에 RC feedforward가 섞이지 않게 했다.
* 외부 제어 모드에서는 `FEATURE_MOTOR_STOP` 조건을 적용하지 않는다. 따라서 ARM 상태에서 외부 제어가 활성화된 경우 `MOTOR_STOP`이 모터 출력을 별도로 차단하지 않는다.
* `BOXMSPOVERRIDE`는 `msp_override_channels_mask` 값과 관계없이 활성 Box 목록에 등록된다.
* 설정 검증 단계에서 override 채널 마스크가 비어 있다는 이유로 `BOXMSPOVERRIDE` 활성화 조건을 자동 삭제하지 않도록 했다.

## 관련 소스 파일

| 파일 | 변경 내용 |
| --- | --- |
| `src/main/msp/msp_protocol_v2_betaflight.h` | `MSP2_SET_RATE_THRUST` 명령 ID 정의 추가 |
| `src/main/msp/msp.c` | 8바이트 payload 파싱 및 외부 제어 값 전달 추가 |
| `src/main/fc/rc.h` | 외부 제어 API 선언 추가 |
| `src/main/fc/rc.c` | 외부 입력 저장, 범위 제한, 100 ms 만료 확인, rate 대체 및 feedforward 비활성화 추가 |
| `src/main/flight/mixer.c` | 외부 thrust 사용 및 외부 제어 중 `MOTOR_STOP` 예외 처리 |
| `src/main/msp/msp_box.c` | `BOXMSPOVERRIDE`의 상시 등록 |
| `src/main/config/config.c` | override 모드 조건 자동 삭제 방지 |

## 안전상 주의

외부 제어 명령은 실제 모터 출력에 영향을 준다. 송신 장치는 100 ms보다 충분히 짧은 주기로 명령을 보내야 하며, 링크 단절·프로세스 정지·잘못된 단위·잘못된 엔디언 처리 상황을 포함한 failsafe를 비행 전에 프로펠러를 분리한 상태에서 검증해야 한다. 명령 만료 시 추력은 0이 되지만, 외부 제어 중에는 `MOTOR_STOP`이 적용되지 않는다는 점을 고려해야 한다.

## `src/config` 서브모듈 상태

상위 저장소의 `src/config` gitlink는 4.5.5 기준 `92c695a`에서 `e95b1db`로 이동했다. 이는 Betaflight 공식 `config` 저장소의 더 최신 커밋을 가리키는 변경이며, 해당 서브모듈 전체의 여러 공식 보드 설정 변경을 함께 포함한다.

또한 현재 작업 폴더의 `src/config/configs/MICOAIR743V2/config.h`는 서브모듈 안의 **추적되지 않은 파일**이다. 이 파일은 상위 저장소의 개인 커밋에 포함되지 않았으며, 이 저장소만 새로 clone해도 자동으로 복원되지 않는다. 이 보드 설정도 재현 가능하게 관리하려면 `config` 서브모듈을 개인 fork로 만들고 해당 파일을 별도 커밋 및 푸시해야 한다.
