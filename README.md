![Betaflight](images/bf_logo.png)

[![Latest version](https://img.shields.io/github/v/release/betaflight/betaflight)](https://github.com/betaflight/betaflight/releases) [![Build](https://img.shields.io/github/actions/workflow/status/betaflight/betaflight/nightly.yml?branch=master)](https://github.com/betaflight/betaflight/actions/workflows/nightly.yml) [![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0) [![Join us on Discord!](https://img.shields.io/discord/868013470023548938)](https://discord.gg/n4E6ak4u3c)

Betaflight is flight controller software (firmware) used to fly multi-rotor craft and fixed wing craft.

This fork differs from Baseflight and Cleanflight in that it focuses on flight performance, leading-edge feature additions, and wide target support.

## CUSTOM

이 저장소는 Betaflight **4.5.5**를 기반으로 개인 용도에 맞게 수정한 펌웨어입니다.

**변경 기록일: 2026-08-28**

이 날짜는 개인 저장소에 수정사항을 최초로 커밋한 날짜입니다. 수정은 커밋 전에 작업 폴더에 존재했으므로 실제 구현 날짜는 별도로 확인할 수 없습니다.

주요 변경 사항은 MSPv2를 통한 외부 제어입니다.

* `MSP_CTBR` (`0x30F0`)는 float collective thrust (`0.0–1.0`)와 float Roll, Pitch, Yaw body rate (`rad/s`)로 기체를 제어합니다.
* `MSP_RPM` (`0x30F1`)은 모터별 정수 RPM 목표값을 받아 DShot telemetry 기반의 폐루프 제어를 수행합니다.
* Configurator의 Modes 탭에서 `MSP CTBR` 또는 `MSP RPM`을 각각 별도 AUX 모드로 설정할 수 있습니다. 활성화된 모드가 제어 방식을 결정합니다.
* 두 모드가 동시에 활성화되면 안전하게 한 가지 출력 경로만 쓰도록 `MSP RPM`이 우선됩니다. 두 AUX 범위는 겹치지 않게 설정하세요.
* 외부 명령이 100 ms 동안 갱신되지 않으면 안전을 위해 rate와 추력을 0으로 처리합니다.
* 외부 제어 중 feedforward를 끄고, `MOTOR_STOP`이 모터를 멈추지 않도록 변경했습니다.

명령 형식, 단위, 제한값, RPM 제어 조건, 관련 소스 파일과 보드 설정 서브모듈의 주의사항은 [edited_parts.md](edited_parts.md)를 참고하세요.

### 테스트 및 빌드 환경

이 커스텀 펌웨어는 **MICOAIRH743V2** 비행 컨트롤러에서 테스트했습니다. 빌드 구성 이름은 `MICOAIR743V2`이며, STM32H743 MCU를 대상으로 합니다.

### MICOAIRH743V2 플래시 파일 만들기

프로젝트 루트에서 아래 명령을 실행합니다.

```bash
git submodule update --init --recursive
make MICOAIR743V2
```

성공하면 `obj/betaflight_*_STM32H743_MICOAIR743V2.hex` 형식의 HEX 플래시 파일이 생성됩니다. 현재 버전의 예시는 다음과 같습니다.

```text
obj/betaflight_4.5.5_STM32H743_MICOAIR743V2.hex
```

생성된 `.hex` 파일은 Betaflight Configurator의 로컬 펌웨어 플래시 기능 또는 사용 중인 STM32 플래싱 도구에서 선택해 쓸 수 있습니다.

> 참고: `MICOAIR743V2` 설정 파일은 현재 `src/config` 서브모듈 안의 로컬 파일입니다. 새 환경에서 빌드하려면 `src/config/configs/MICOAIR743V2/config.h`가 존재해야 합니다. 이 설정 파일을 개인 저장소에서도 자동으로 복원할 수 있게 하려면 `config` 서브모듈도 별도 fork로 관리해야 합니다.

## Events

| Date  | Event |
| - | - |
| 01-11-2022 | Firmware 4.4 Feature freeze |
| 29-01-2023 | Firmware 4.4 Release |


## News

### Requirements for the submission of new and updated targets

The following new requirements for pull requests adding new targets or modifying existing targets are put in place from now on:

1. Read the [hardware specification](https://betaflight.com/docs/development/manufacturer/manufacturer-design-guidelines)

2. No new F3 based targets will be accepted;

3. For any new target that is to be added, only a Unified Target config into https://github.com/betaflight/unified-targets/tree/master/configs/default needs to be submitted. See the [instructions](https://betaflight.com/docs/manufacturer/creating-an-unified-target) for how to create a Unified Target configuration. If there is no Unified Target for the MCU type of the new target (see instructions above), then a 'legacy' format target definition into `src/main/target/` has to be submitted as well;

4. For changes to existing targets, the change needs to be applied to the Unified Target config in https://github.com/betaflight/unified-targets/tree/master/configs/default. If no Unified Target configuration for the target exists, a new Unified Target configuration will have to be created and submitted. If there is no Unified Target for the MCU type of the new target (see instructions above), then an update to the 'legacy' format target definition in `src/main/target/` has to be submitted alongside the update to the Unified Target configuration.


## Features

Betaflight has the following features:

* Multi-color RGB LED strip support (each LED can be a different color using variable length WS2811 Addressable RGB strips - use for Orientation Indicators, Low Battery Warning, Flight Mode Status, Initialization Troubleshooting, etc)
* DShot (150, 300 and 600), Multishot, Oneshot (125 and 42) and Proshot1000 motor protocol support
* Blackbox flight recorder logging (to onboard flash or external microSD card where equipped)
* Support for targets that use the STM32 F4, G4, F7 and H7 processors
* PWM, PPM, SPI, and Serial (SBus, SumH, SumD, Spektrum 1024/2048, XBus, etc) RX connection with failsafe detection
* Multiple telemetry protocols (CRSF, FrSky, HoTT smart-port, MSP, etc)
* RSSI via ADC - Uses ADC to read PWM RSSI signals, tested with FrSky D4R-II, X8R, X4R-SB, & XSR
* OSD support & configuration without needing third-party OSD software/firmware/comm devices
* OLED Displays - Display information on: Battery voltage/current/mAh, profile, rate profile, mode, version, sensors, etc
* In-flight manual PID tuning and rate adjustment
* PID and filter tuning using sliders
* Rate profiles and in-flight selection of them
* Configurable serial ports for Serial RX, Telemetry, ESC telemetry, MSP, GPS, OSD, Sonar, etc - Use most devices on any port, softserial included
* VTX support for Unify Pro and IRC Tramp
* and MUCH, MUCH more.

## Installation & Documentation

See: https://betaflight.com/docs/wiki

## Support and Developers Channel

There's a dedicated Discord server here:

https://discord.gg/n4E6ak4u3c

We also have a Facebook Group. Join us to get a place to talk about Betaflight, ask configuration questions, or just hang out with fellow pilots.

https://www.facebook.com/groups/betaflightgroup/

Etiquette: Don't ask to ask and please wait around long enough for a reply - sometimes people are out flying, asleep or at work and can't answer immediately.

## Configuration Tool

To configure Betaflight you should use the Betaflight-configurator GUI tool (Windows/OSX/Linux) which can be found here:

https://github.com/betaflight/betaflight-configurator/releases/latest

## Contributing

Contributions are welcome and encouraged. You can contribute in many ways:

* implement a new feature in the firmware or in configurator (see [below](#Developers));
* documentation updates and corrections;
* How-To guides - received help? Help others!
* bug reporting & fixes;
* new feature ideas & suggestions;
* provide a new translation for configurator, or help us maintain the existing ones (see [below](#Translators)).

The best place to start is the Betaflight Discord (registration [here](https://discord.gg/n4E6ak4u3c)). Next place is the github issue tracker:

https://github.com/betaflight/betaflight/issues
https://github.com/betaflight/betaflight-configurator/issues

Before creating new issues please check to see if there is an existing one, search first otherwise you waste people's time when they could be coding instead!

If you want to contribute to our efforts financially, please consider making a donation to us through [PayPal](https://paypal.me/betaflight).

If you want to contribute financially on an ongoing basis, you should consider becoming a patron for us on [Patreon](https://www.patreon.com/betaflight).

## Developers

Contribution of bugfixes and new features is encouraged. Please be aware that we have a thorough review process for pull requests, and be prepared to explain what you want to achieve with your pull request.
Before starting to write code, please read our [development guidelines](https://betaflight.com/docs/development) and [coding style definition](https://betaflight.com/docs/development/CodingStyle).

GitHub actions are used to run automatic builds

## Translators

We want to make Betaflight accessible for pilots who are not fluent in English, and for this reason we are currently maintaining translations into 21 languages for Betaflight Configurator: Català, Dansk, Deutsch, Español, Euskera, Français, Galego, Hrvatski, Bahasa Indonesia, Italiano, 日本語, 한국어, Latviešu, Português, Português Brasileiro, polski, Русский язык, Svenska, 简体中文, 繁體中文.
We have got a team of volunteer translators who do this work, but additional translators are always welcome to share the workload, and we are keen to add additional languages. If you would like to help us with translations, you have got the following options:
- if you help by suggesting some updates or improvements to translations in a language you are familiar with, head to [crowdin](https://crowdin.com/project/betaflight-configurator) and add your suggested translations there;
- if you would like to start working on the translation for a new language, or take on responsibility for proof-reading the translation for a language you are very familiar with, please head to the Betaflight Discord chat (registration [here](https://discord.gg/n4E6ak4u3c)), and join the ['translation'](https://discord.com/channels/868013470023548938/1057773726915100702) channel - the people in there can help you to get a new language added, or set you up as a proof reader.

## Hardware Issues

Betaflight does not manufacture or distribute their own hardware. While we are collaborating with and supported by a number of manufacturers, we do not do any kind of hardware support.
If you encounter any hardware issues with your flight controller or another component, please contact the manufacturer or supplier of your hardware, or check RCGroups https://rcgroups.com/forums/showthread.php?t=2464844 to see if others with the same problem have found a solution.

## Betaflight Releases

https://github.com/betaflight/betaflight/releases

## Open Source / Contributors

Betaflight is software that is **open source** and is available free of charge without warranty to all users.

Betaflight is forked from Cleanflight, so thanks goes to all those who have contributed to Cleanflight and its origins.

Origins for this fork (Thanks!):
* **Alexinparis** (for MultiWii),
* **timecop** (for Baseflight),
* **Dominic Clifton** (for Cleanflight),
* **borisbstyle** (for Betaflight), and
* **Sambas** (for the original STM32F4 port).

The Betaflight Configurator is forked from Cleanflight Configurator and its origins.

Origins for Betaflight Configurator:
* **Dominic Clifton** (for Cleanflight configurator), and
* **ctn** (for the original Configurator).

Big thanks to current and past contributors:
* Budden, Martin (martinbudden)
* Bardwell, Joshua (joshuabardwell)
* Blackman, Jason (blckmn)
* ctzsnooze
* Höglund, Anders (andershoglund)
* Ledvina, Petr (ledvinap) - **IO code awesomeness!**
* kc10kevin
* Keeble, Gary (MadmanK)
* Keller, Michael (mikeller) - **Configurator brilliance**
* Kravcov, Albert (skaman82) - **Configurator brilliance**
* MJ666
* Nathan (nathantsoi)
* ravnav
* sambas - **bringing us the F4**
* savaga
* Stålheim, Anton (KiteAnton)

And many many others who haven't been mentioned....
