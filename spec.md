---
author:
- The FIRRTL Specification Contributors
title: Specification for the FIRRTL Language
date: \today
# Options passed to the document class
classoption:
- 12pt
# Link options
colorlinks: true
linkcolor: blue
filecolor: magenta
urlcolor: cyan
toccolor: blue
# General pandoc configuration
toc: true
numbersections: true
# Header Setup
pagestyle:
  fancy: true
# Margins
geometry: margin=1in
# pandoc-crossref
autoSectionLabels: true
figPrefix:
  - Figure
  - Figures
eqnPrefix:
  - Equation
  - Equations
tblPrefix:
  - Table
  - Tables
lstPrefix:
  - Listing
  - Listings
secPrefix:
  - Section
  - Sections
# This 'lastDelim' option does not work...
lastDelim: ", and"
---

# Introduction

## Background

FIRRTL(RTL을 위한 유연한 중간 표현)에 대한 아이디어는 고도로 파라미터화된 회로 설계 생성기를 작성하는 데 사용되는 Scala에 내장된 하드웨어 설명 언어(HDL)인 Chisel에 대한 작업에서 시작되었습니다. 치즐 설계자는 Scala 함수를 사용하여 회로 구성 요소를 조작하고, 인터페이스를 Scala 유형으로 인코딩하며, Scala의 객체 지향 기능을 사용하여 자체 회로 라이브러리를 작성합니다. 이러한 형태의 메타 프로그래밍은 표현력이 풍부하고 신뢰할 수 있으며 유형 안전성이 뛰어난 제너레이터를 구현하여 RTL 설계 생산성과 견고성을 향상시킵니다.

U.C. 버클리의 컴퓨터 아키텍처 연구 그룹은 대학원생으로 구성된 소규모 팀이 정교한 RTL 회로를 설계할 수 있도록 Chisel에 크게 의존하고 있습니다. 대학원생 12명 미만으로 구성된 이 아키텍처 그룹은 3년 동안 10가지가 넘는 설계를 테이프로 제작했습니다.

내부적으로는 치즐의 개발과 학습에 대한 투자가 생산성 향상이라는 큰 성과로 보답했습니다. 하지만 다음과 같은 이유로 Chisel의 외부 도입 속도는 더디게 진행되었습니다.

1. 커스텀 회로 변압기를 작성하려면 Chisel 컴포넌트 내부에 대한 내부에 대한 깊은 지식이 필요합니다.
2. 치즐 시맨틱이 제대로 지정되지 않았기 때문에 다른 언어에서 타겟팅할 수 없습니다.
3. 지정되지 않은 의미론으로 인해 오류 검사가 원칙에 어긋나며 그 결과 이해할 수 없는 오류 메시지가 발생합니다.
4. 함수형 프로그래밍 언어(스칼라)를 배우는 것은 프로그래밍 언어 경험이 부족한 RTL 디자이너에게는 함수형 프로그래밍 언어(스칼라)를 배우기가 어렵습니다.
5. 임베디드 Chisel과 호스트 언어를 개념적으로 분리하는 것이 어렵습니다. HDL을 호스트 언어와 개념적으로 분리하는 것은 새로운 사용자에게는 어렵습니다.
6. 치즐(Verilog)의 출력은 읽을 수 없고 시뮬레이션 속도가 느립니다.

결과적으로 Chisel은 처음부터 다시 설계해야 했습니다. 시맨틱을 표준화하고, 컴파일 프로세스를 모듈화하며, 깔끔하게 프론트엔드, 중간 표현, 백엔드를 분리해야 했습니다. 잘 정의된 중간 표현(IR)을 통해 시스템을 타겟팅할 수 있습니다. 다른 호스트 프로그래밍 언어에 임베디드된 다른 HDL이 시스템을 타겟팅할 수 있습니다. RTL 설계자는 이미 익숙한 언어 내에서 작업할 수 있습니다. A 구체적인 구문으로 명확하게 정의된 IR을 사용하면 회로 생성기 및 변압기의 출력을 회로 생성기 및 변압기의 출력을 검사할 수 있으므로 호스트 언어와 구성 언어 사이의 명확하게 구분할 수 있습니다. 명확하게 정의된 의미론 컴파일러 구현에 대한 지식이 없는 사용자도 회로를 작성할 수 있습니다. 트랜스포머를 작성할 수 있으며, 시뮬레이션 속도를 위한 회로 최적화, 신호 활동 카운터 자동 삽입 등을 예로 들 수 있습니다. 잘 정의된 IR의 또 다른 이점은 잘 정의된 IR의 또 다른 이점은 각 컴파일 단계 전후에 적용될 수 있는 구조적 불변성입니다. 각 컴파일 단계 전후에 적용될 수 있는 구조적 불변성으로, 보다 강력한 컴파일러와 구조화된 오류 검사를 위한 메커니즘을 제공합니다.

## Design Philosophy

FIRRTL은 Chisel HDL이 생성하는 표준화된 정교한 회로를 나타냅니다. FIRRTL은 치즐의 정교화 직후 회로를 나타냅니다. 모든 메타 프로그래밍이 실행된 후 Chisel HDL과 유사하도록 설계되었습니다. 따라서 메타 프로그래밍 기능을 거의 사용하지 않는 사용자 프로그램은 생성된 FIRRTL과 거의 동일하게 보일 것입니다.

이러한 이유로 FIRRTL은 벡터 유형, 번들 유형, 조건문 및 모듈과 같은 상위 수준 구성을 최고 수준으로 지원합니다. FIRRTL 컴파일러는 Verilog를 생성하기 전에 하이레벨 구문을 로우레벨 구문으로 변환하도록 선택할 수 있습니다.

이제 호스트 언어가 메타 프로그래밍 기능에만 사용되기 때문에 프론트엔드를 매우 가볍게 만들 수 있으며, 다른 언어로 작성된 추가 HDL은 FIRRTL을 대상으로 하여 컴파일러 툴체인의 대부분을 재사용할 수 있습니다.

# Acknowledgments

FIRRTL 사양은 원래 아담 이즈라엘레비츠(`[@azidar](https://github.com/azidar)`, 패트릭 리(`[@CuppoJava](https://github.com/CuppoJava)`, 조나단 바크라치(`[@jackbackrack](https://github.com/jackbackrack)`가 작성한 UC 버클리 기술 보고서([UCB/EECS-2016-9](https://www2.eecs.berkeley.edu/Pubs/TechRpts/2016/EECS-2016-9.html)로 발표되었습니다. 이후 FIRRTL에 대한 비전은 [ICCAD 논문](https://ieeexplore.ieee.org/abstract/document/8203780)과 [아담의 논문](https://www2.eecs.berkeley.edu/Pubs/TechRpts/2019/EECS-2019-168.html)에서 확장되었습니다.

그 기간과 그 이후로 사양에 대한 많은 기여와 개선이 있었습니다. 최초 기술 보고서 이후 기여자들의 작업을 더 잘 반영하기 위해 FIRRTL 사양은 *FIRRTL 사양 기여자*가 작성하는 것으로 변경되었습니다. 이러한 기여자 목록은 아래와 같습니다:

<!-- This can be generated using ./scripts/get-authors.sh -->
- [`@albert-magyar`](https://github.com/albert-magyar)
- [`@azidar`](https://github.com/azidar)
- [`@ben-marshall`](https://github.com/ben-marshall)
- [`@boqwxp`](https://github.com/boqwxp)
- [`@chick`](https://github.com/chick)
- [`@dansvo`](https://github.com/dansvo)
- [`@darthscsi`](https://github.com/darthscsi)
- [`@debs-sifive`](https://github.com/debs-sifive)
- [`@donggyukim`](https://github.com/donggyukim)
- [`@dtzSiFive`](https://github.com/dtzSiFive)
- [`@ekiwi`](https://github.com/ekiwi)
- [`@ekiwi-sifive`](https://github.com/ekiwi-sifive)
- [`@felixonmars`](https://github.com/felixonmars)
- [`@grebe`](https://github.com/grebe)
- [`@jackkoenig`](https://github.com/jackkoenig)
- [`@jared-barocsi`](https://github.com/jared-barocsi)
- [`@keszocze`](https://github.com/keszocze)
- [`@mwachs5`](https://github.com/mwachs5)
- [`@prithayan`](https://github.com/prithayan)
- [`@richardxia`](https://github.com/richardxia)
- [`@seldridge`](https://github.com/seldridge)
- [`@sequencer`](https://github.com/sequencer)
- [`@shunshou`](https://github.com/shunshou)
- [`@tdb-alcorn`](https://github.com/tdb-alcorn)
- [`@tymcauley`](https://github.com/tymcauley)
- [`@youngar`](https://github.com/youngar)

# File Preamble

FIRRTL 파일은 파일이 준수하는 이 표준의 버전을 나타내는 매직 문자열과 버전 식별자로 시작됩니다([@sec:versioning-scheme-of-this-document] 참조). 이 서문을 포함하는 이 표준의 첫 번째 버전 릴리스 이전에 이 표준의 버전에 따라 생성된 파일에는 이 매직 문자열이 존재하지 않습니다.

``` firrtl
FIRRTL version 1.1.0
circuit MyTop...
```

# Circuits and Modules

## Circuits

모든 FIRRTL 회로는 모듈 목록으로 구성되며, 각 모듈은 인스턴스화할 수 있는 하드웨어 블록을 나타냅니다. 회로는 최상위 모듈의 이름을 지정해야 합니다.

``` firrtl
circuit MyTop :
  module MyTop :
    ; ...
  module MyModule :
    ; ...
```

## Modules

각 모듈에는 주어진 이름, 포트 목록, 모듈 내의 회로 연결을 나타내는 문 목록이 있습니다. 모듈 포트는 입력 또는 출력일 수 있는 방향, 이름, 포트의 데이터 유형으로 지정됩니다.

다음 예는 입력 포트 1개, 출력 포트 1개, 입력 포트와 출력 포트를 연결하는 문 1개가 있는 모듈을 선언합니다. connect 문에 대한 자세한 내용은 [@sec:connects]를 참조하세요.

``` firrtl
module MyModule :
  input foo: UInt
  output bar: UInt
  bar <= foo
```

모듈 정의는 모듈이 최종 회로에 물리적으로 존재한다는 것을 *표시하지* 않는다는 점에 유의하세요. 모듈을 인스턴스화하는 방법에 대한 자세한 내용은 인스턴스 문 설명을 참조하세요([@sec:instance]).

## Externally Defined Modules

외부 정의 모듈은 현재 회로에 구현이 제공되지 않는 모듈입니다. 외부 정의 모듈의 포트와 이름만 회로에 지정됩니다. 외부에서 정의된 모듈에는 결과 Verilog에서 외부 모듈의 이름을 설정하는 선택적 *defname*과 0개 이상의 이름-값 *parameter* 문이 순서대로 포함될 수 있습니다. 각 이름-값 매개변수 문은 결과 Verilog에서 명명된 매개변수로 값이 전달됩니다.

외부에서 정의된 모듈의 예는 다음과 같습니다:

``` firrtl
extmodule MyExternalModule :
  input foo: UInt<2>
  output bar: UInt<4>
  output baz: SInt<8>
  defname = VerilogName
  parameter x = "hello"
  parameter y = 42
```

외부에서 정의된 모든 모듈 포트의 너비를 지정해야 합니다. [@sec:width-inference]에 설명된 너비 추론은 모듈 포트에 대해 지원되지 않습니다.

외부에서 정의된 모듈의 일반적인 용도는 별도로 작성되어 다운스트림 도구에 FIRRTL로 생성된 Verilog와 함께 제공될 Verilog 모듈을 나타내는 것입니다.

# Types

FIRRTL에는 두 가지 타입 클래스가 있습니다: _ground_ 타입과 _aggregate_ 타입입니다. 그라운드 타입은 기본 타입이며 다른 타입으로 구성되지 않습니다. Aggregate 타입은 하나 이상의 Aggregate 또는 Ground 타입으로 구성됩니다.

## Ground Types

FIRRTL의 ground 타입에는 부호 없는 정수 타입, 부호 있는 정수 타입, 부호 있는 정수 타입, 클록 타입, 리셋 타입, 아날로그 타입입니다.

### Integer Types

부호 없는 정수 타입과 부호 있는 정수 타입 모두 선택적으로 알려진 음수가 아닌 정수 비트 폭을 지정할 수 있습니다.

``` firrtl
UInt ; unsigned int type with inferred width
SInt ; signed int type with inferred width
UInt<10> ; unsigned int type with 10 bits
SInt<32> ; signed int type with 32 bits
```

또는 비트 폭을 생략하면 [@sec:width-inference]에 자세히 설명된 대로 FIRRTL의 폭 유추기에 의해 자동으로 유추됩니다.

#### Zero Bit Width Integers

너비가 0인 정수는 허용됩니다. 항상 0으로 확장됩니다. 따라서 양수 비트 폭으로 확장되는 연산에 사용하면 0처럼 동작합니다. 비트 폭이 0인 정수는 아무런 정보를 전달하지 않지만 편의상 0 비트 정수 상수 0을 허용합니다: `UInt<0>(0)`{.firrtl} 및 `SInt<0>(0)`{.firrtl}.

``` firrtl
wire zero_u : UInt<0>
zero_u is invalid
wire zero_s : SInt<0>
zero_s is invalid

wire one_u : UInt<1>
one_u <= zero_u
wire one_s : SInt<1>
one_s <= zero_s
```

위는 아래와 동등합니다:

```
wire one_u : UInt<1>
one_u <= UInt<1>(0)
wire one_s : SInt<1>
one_s <= SInt<1>(0)
```

### Clock Type

클럭 타입은 클럭 신호를 전달하기 위한 wire 및 포트를 설명하는 데 사용됩니다. 클럭 타입이 있는 컴포넌트의 사용은 제한됩니다.  클럭 신호는 대부분의 primitive 연산에서 사용할 수 없으며, 클럭 신호는 클럭 타입으로 선언된 컴포넌트에만 연결할 수 있습니다.

클럭 타입은 다음과 같이 지정됩니다:

``` firrtl
Clock
```

### Reset Type

유추되지 않은 `Reset`{.firrtl} 타입은 컴파일 중에 `UInt<1>`{.firrtl}(동기 리셋) 또는 `AsyncReset`{.firrtl}(비동기 리셋)으로 유추됩니다.

``` firrtl
Reset ; inferred type
AsyncReset
```

레지스터에 사용되는 동기 리셋은 동기 리셋에 대한 하드웨어 기술 언어 표현에 매핑됩니다.

다음 예제는 동기 리셋으로 추론되는 비추론 리셋을 보여줍니다.

``` firrtl
input a : UInt<1>
wire reset : Reset
reset <= a
```

리셋 추론 후, `reset`{.firrtl}은 동기식으로 추론됩니다.
`UInt<1>`{.firrtl} 유형으로 추론됩니다:

``` firrtl
input a : UInt<1>
wire reset : UInt<1>
reset <= a
```

레지스터에 사용되는 비동기 재설정은 비동기 재설정을 위한 하드웨어 기술 언어 표현에 매핑됩니다.

다음 예제는 비동기 리셋의 사용법을 보여줍니다.

``` firrtl
input clock : Clock
input reset : AsyncReset
input x : UInt<8>
reg y : UInt<8>, clock with : (reset => (reset, UInt(123)))
; ...
```

추론 규칙은 다음과 같습니다:

1. 비동기 리셋에 의해 구동되거나 비동기 리셋만 구동되는 추론되지 않은 리셋은 비동기 리셋으로 추론됩니다.
1. 비동기 및/또는 동기 리셋에 의해 구동되거나 동기 리셋을 모두 구동하는 추론되지 않은 리셋은 오류입니다.
1. 그렇지 않으면 리셋이 동기식으로 추론됩니다(즉, 추론되지 않은 리셋은 무효화되거나 동기식 리셋에 의해서만 구동되거나 동기식 리셋만 구동됩니다).

동기든 비동기든 `Reset`{.firrtl}은 다른 타입으로 형변환(캐스팅) 할 수 있습니다. 리셋 타입 간에 형변환하는 것도 합법적입니다:

``` firrtl
input a : UInt<1>
output y : AsyncReset
output z : Reset
wire r : Reset
r <= a
y <= asAsyncReset(r)
z <= asUInt(y)
```

캐스팅에 대한 자세한 내용은 [@sec:primitive-operations]를 참조하세요.

### Analog Type

아날로그 타입은 와이어 또는 포트를 여러 드라이버에 연결할 수 있음을 지정합니다. `Analog`{.firrtl}는 노드 또는 레지스터 유형의 일부로 사용할 수 없으며, 메모리의 데이터 유형의 일부로도 사용할 수 없습니다. 이 점에서 Verilog에서 `inout`{.firrtl} 포트가 사용되는 방식과 유사하며, FIRRTL 아날로그 신호는 종종 외부 Verilog 또는 VHDL IP와 인터페이스하는 데 사용됩니다.

다른 모든 그라운드 타입과 달리 아날로그 신호는 커넥션 문 양쪽에 나타날 수 없습니다. 대신, 아날로그 신호는 정류(commutative) `attach`{.firrtl} 문을 사용하여 서로 연결됩니다. 아날로그 신호는 여러 개의 attach 문에 나타날 수 있으며, 법적 회로에는 연결되지 않은 아날로그 신호도 포함될 수 있습니다. 아날로그 신호에 적용될 수 있는 유일한 프리미티브 연산은 다른 신호 유형으로 형변환하는 것입니다.

아날로그 신호가 aggregate 타입의 필드로 나타나는 경우 aggregate는 표준 연결 구문에 나타날 수 없습니다.

정수 유형과 마찬가지로 아날로그 유형은 다중 비트 신호를 나타낼 수 있습니다. 아날로그 신호에 구체적인 폭이 지정되지 않은 경우, 주어진 연결 연산에 대한 모든 인수의 폭이 동일해야 하는 매우 제한적인 폭 추론 규칙에 따라 폭이 추론됩니다.

``` firrtl
Analog<1>  ; 1-bit analog type
Analog<32> ; 32-bit analog type
Analog     ; analog type with inferred width
```

## Aggregate Types

FIRRTL은 벡터와 번들의 두 가지 aggregate 타입을 지원합니다.  aggregate 타입은 ground 타입 또는 다른 aggregate 타입으로 구성됩니다.

### Vector Types

벡터 유형은 주어진 타입의 요소의 정렬된 시퀀스를 표현하는 데 사용됩니다. 시퀀스의 길이는 음수가 아니며 알려진 길이여야 합니다.

다음 예는 16비트 부호 없는 정수로 구성된 10개의 요소 벡터를 지정합니다.

``` firrtl
UInt<16>[10]
```

다음 예제는 생략되었지만 동일한 비트 폭의 부호 없는 정수로 구성된 10개의 요소 벡터를 지정합니다.

``` firrtl
UInt[10]
```

다른 aggregate 타입을 포함한 모든 타입을 벡터의 요소 타입으로 사용할 수 있습니다. 다음 예제에서는 20개의 요소 벡터를 지정하며, 각 요소는 16비트 부호 없는 정수로 구성된 10개의 요소 벡터입니다.

``` firrtl
UInt<16>[10][20]
```

### Bundle Types

번들 타입은 중첩 및 명명된 타입의 컬렉션을 표현하는 데 사용됩니다.  번들 타입의 모든 필드에는 지정된 이름과 유형이 있어야 합니다.

다음은 복소수를 표현하는 데 사용할 수 있는 타입의 예입니다. 여기에는 `real`{.firrtl}과 `imag`{.firrtl}이라는 두 개의 필드가 있으며, 둘 다 10비트 부호 있는 정수입니다.

``` firrtl
{real: SInt<10>, imag: SInt<10>}
```

또한 필드는 선택적으로 *반전된* 방향으로 선언할 수 있습니다.

``` firrtl
{word: UInt<32>, valid: UInt<1>, flip ready: UInt<1>}
```

번들 타입이 있는 회로 구성 요소 간의 연결에서 뒤집힌 필드가 전달하는 데이터는 뒤집히지 않은 필드가 전달하는 데이터와 반대 방향으로 흐릅니다.

예를 들어 다음 타입으로 선언된 모듈 출력 포트를 생각해 보겠습니다:

``` firrtl
output a: {word: UInt<32>, valid: UInt<1>, flip ready: UInt<1>}
```

a`{.firrtl} 포트에 대한 커넥션에서 `word`{.firrtl} 및 `valid`{.firrtl} 하위 필드가 전달하는 데이터는 모듈 외부로 흐르고, `ready`{.firrtl} 하위 필드가 전달하는 데이터는 모듈 내부로 흐르게 됩니다. 번들 필드 방향이 연결에 미치는 영향에 대한 자세한 내용은 [@sec:connects]에 설명되어 있습니다.

벡터 타입의 경우와 마찬가지로 번들 필드는 다른 aggregate 타입을 포함한 모든 타입으로 선언할 수 있습니다.

``` firrtl
{real: {word: UInt<32>, valid: UInt<1>, flip ready: UInt<1>},
 imag: {word: UInt<32>, valid: UInt<1>, flip ready: UInt<1>}}

```

데이터 흐름의 최종 방향을 계산할 때 필드의 방향은 필드에 중첩된 모든 유형에 재귀적으로 적용됩니다. 예를 들어 중첩된 번들 유형을 포함하는 번들 유형으로 선언된 다음 모듈 포트를 생각해 보겠습니다.

``` firrtl
output myport: {a: UInt, flip b: {c: UInt, flip d: UInt}}
```

`myport`{.firrtl}에 대한 연결에서 `a`{.firrtl} 하위 필드는 모듈 밖으로 흘러나옵니다.  `b`{.firrtl} 하위 필드에 포함된 `c`{.firrtl} 하위 필드는 모듈로 유입되고, `b`{.firrtl} 하위 필드에 포함된 `d`{.firrtl} 하위 필드는 모듈 밖으로 흘러나옵니다.

## Passive Types

일부 회로 구성 요소가 데이터가 양방향으로 흐르도록 허용하는 유형으로 선언되는 것은 부적절합니다. 예를 들어 메모리의 모든 하위 요소는 같은 방향으로 흘러야 합니다. 이러한 컴포넌트는 패시브 타입만 가질 수 있도록 제한됩니다.

직관적으로 패시브 타입은 모든 데이터가 같은 방향으로 흐르는 타입으로, 방향이 뒤바뀐 필드를 재귀적으로 포함하지 않는 타입으로 정의됩니다. 따라서 모든 접지 타입은 패시브 타입입니다. 벡터 유형은 요소 유형이 패시브인 경우 패시브 유형입니다. 그리고 번들 유형은 반전된 필드가 없고 모든 필드 유형이 패시브인 경우 패시브 유형입니다.

## Type Equivalence

유형 등가 관계는 두 컴포넌트 간의 연결이 합법적인지 여부를 결정하는 데 사용됩니다. 연결 문에 대한 자세한 내용은 [@sec:connects]를 참조하세요.

부호 없는 정수 유형은 비트 폭에 관계없이 항상 다른 부호 없는 정수 유형과 동등하며 다른 유형과 동등하지 않습니다. 마찬가지로 부호 있는 정수 타입은 비트 폭에 관계없이 항상 다른 부호 있는 정수 타입과 같으며 다른 타입과 같지 않습니다.

클럭 타입은 클럭 타입과 동등하며 다른 타입과 동등하지 않습니다.

추론되지 않은 `Reset`{.firrtl}은 다른 `Reset`{.firrtl}, 폭을 알 수 없는 `UInt`{.firrtl}, `UInt<1>`{.firrtl} 또는 `AsyncReset`{.firrtl}에 연결할 수 있습니다. `UInt`{.firrtl}와 `AsyncReset`{.firrtl} 둘 다에 연결할 수 없습니다.

`AsyncReset`{.firrtl} 타입은 다른 `AsyncReset`{.firrtl} 또는 `Reset`{.firrtl}에 연결할 수 있습니다.

두 벡터 유형은 길이가 같고 요소 유형이 같으면 동등합니다.

두 번들 유형은 필드 수가 같고 번들의 i번째 필드 이름과 방향이 일치하고 유형이 같으면 동일합니다. 따라서 `{a:UInt, b:UInt}`{.firrtl}은 `{b:UInt, a:UInt}`{.firrtl} 및 `{a: {flip b:UInt}}`{.firrtl}은 `{flip a: {b: UInt}}`{.firrtl}와 같지 않습니다.

# Statements

Statements는 모듈 내의 구성 요소와 이들이 상호 작용하는 방식을 설명하는 데 사용됩니다.

## Connects

connect 문은 두 회로 구성 요소 간에 물리적으로 유선 연결을 지정하는 데 사용됩니다.

다음 예제에서는 모듈의 입력 포트를 출력 포트에 연결하는 것을 보여 줍니다. 여기서 `myinput`{.firrtl} 포트는 `myoutput`{.firrtl} 포트에 연결됩니다.

``` firrtl
module MyModule :
  input myinput: UInt
  output myoutput: UInt
  myoutput <= myinput
```

합법적인 연결이 되려면 다음 조건이 충족되어야 합니다:

1.  왼쪽 및 오른쪽 표현식의 유형이 동일해야 합니다(자세한 내용은 [@sec:type-equivalence] 참조).

2.  왼쪽 표현식의 흐름이 싱크 또는 이중이어야 합니다(흐름에 대한 설명은 [@sec:flows] 참조).

3.  오른쪽 표현식의 흐름이 소스 또는 이중이거나 오른쪽 표현식에 수동 유형이 있습니다.

좁은 접지 유형 구성 요소에서 더 넓은 접지 유형 구성 요소로 연결 문을 연결하면 해당 값이 자동으로 더 큰 비트 폭으로 부호가 확장되거나 0으로 확장됩니다. 더 넓은 접지 유형 컴포넌트에서 더 좁은 접지 유형 컴포넌트로의 연결 문은 더 작은 비트 폭에 맞게 값이 자동으로 잘립니다. 집계 유형이 있는 두 회로 컴포넌트 간의 연결 문의 동작은 [@sec:the-connection-algorithm]의 연결 알고리즘에 의해 정의됩니다.

### The Connection Algorithm

접지 유형 간의 Connect 문은 더 확장할 수 없습니다.

두 벡터 유형 컴포넌트 간의 Connect 문은 오른쪽 표현식의 각 하위 요소를 왼쪽 표현식의 해당 하위 요소에 재귀적으로 연결합니다.

두 번들 유형 컴포넌트 간의 연결 문은 오른쪽 표현식의 i번째 필드와 왼쪽 표현식의 i번째 필드를 연결합니다. i번째 필드가 반전되지 않으면 오른쪽 필드가 왼쪽 필드에 연결됩니다.  반대로 i번째 필드가 반전되면 왼쪽 필드가 오른쪽 필드에 연결됩니다.

## Statement Groups

하나 이상의 statement로 구성된 정렬된 시퀀스를 단일 문으로 그룹화할 수 있으며, 이를 statement 그룹이라고 합니다. 다음 예에서는 세 개의 connect 문으로 구성된 statement 그룹을 보여 줍니다.

``` firrtl
module MyModule :
  input a: UInt
  input b: UInt
  output myport1: UInt
  output myport2: UInt
  myport1 <= a
  myport1 <= b
  myport2 <= a
```

### Last Connect Semantics

statement 그룹에서 statement 순서는 중요합니다. 직관적으로, 정교화 중에 statement는 순서대로 실행되며, 이후 statement가 이전 statement보다 우선합니다. 앞의 예에서 결과 회로에서 포트 `b`{.firrtl}는 `myport1`{.firrtl}에 연결되고, 포트 `a`{.firrtl}는 `myport2`{.firrtl}에 연결됩니다.

조건문도 마지막 연결 의미론의 영향을 받으며, 자세한 내용은 [@sec:조건부-마지막-연결-의미론]을 참조하세요.

aggregate 유형의 회로 구성 요소에 대한 연결 뒤에 해당 구성 요소의 하위 요소에 대한 연결이 뒤따르는 경우 하위 요소에 대한 연결만 덮어씁니다. 다른 하위 요소에 대한 연결은 영향을 받지 않습니다. 다음 예제에서 결과 회로에서 포트 `portx`{.firrtl}의 `c`{.firrtl} 하위 요소는 `myport`{.firrtl}의 `c`{.firrtl} 하위 요소에 연결되고, 포트 `porty`{.firrtl}는 `myport`{.firrtl}의 `b`{.firrtl} 하위 요소에 연결됩니다.

``` firrtl
module MyModule :
  input portx: {b: UInt, c: UInt}
  input porty: UInt
  output myport: {b: UInt, c: UInt}
  myport <= portx
  myport.b <= porty
```

위의 회로는 다음과 같이 동일하게 재작성할 수 있습니다.

``` firrtl
module MyModule :
  input portx: {b: UInt, c: UInt}
  input porty: UInt
  output myport: {b: UInt, c: UInt}
  myport.b <= porty
  myport.c <= portx.c
```

Aggregate 회로 구성 요소의 하위 요소에 대한 연결 다음에 전체 회로 구성 요소에 대한 연결이 이어지는 경우, 나중에 연결이 이전 연결을 완전히 덮어씁니다.

``` firrtl
module MyModule :
  input portx: {b: UInt, c: UInt}
  input porty: UInt
  output myport: {b: UInt, c: UInt}
  myport.b <= porty
  myport <= portx
```

위의 회로는 다음과 같이 동일하게 재작성할 수 있습니다.

``` firrtl
module MyModule :
  input portx: {b: UInt, c: UInt}
  input porty: UInt
  output myport: {b: UInt, c: UInt}
  myport <= portx
```

하위 필드 표현식에 대한 자세한 내용은 [@sec:sub-fields]를 참조하세요.

## Empty

empty statement는 아무 작업도 수행하지 않으며 단순히 statement가 예상되는 자리 표시자로 사용됩니다. `skip`{.firrtl} 키워드를 사용하여 지정합니다.

아래는 예제입니다:

``` firrtl
a <= b
skip
c <= d
```

와 동일하게 표현할 수 있습니다:

``` firrtl
a <= b
c <= d
```

empty statement은 조건문에서 `else`{.firrtl} 분기로 가장 자주 사용되거나 변환 패스 중에 제거된 구성 요소를 위한 편리한 자리 표시자로 사용됩니다. 조건문에 대한 자세한 내용은 [@sec:conditionals]를 참조하세요.

## Wires

wire는 연결 문을 사용하여 연결하고 연결할 수 있는 명명된 조합 회로 구성 요소입니다.

다음 예제에서는 `mywire`{.firrtl}라는 이름과 `UInt`{.firrtl} 타입으로 wire를 인스턴스화하는 방법을 보여 줍니다.

``` firrtl
wire mywire: UInt
```

## Registers

레지스터는 명명된 상태 저장 회로 컴포넌트입니다.  레지스터에서 읽기는 요소의 현재 값을 반환하고 쓰기는 레지스터의 클록 포트의 양수 에지가 끝날 때까지 표시되지 않습니다.

레지스터의 클럭 신호는 `Clock`{.firrtl} 유형이어야 합니다.  레지스터의 타입은 패시브 타입이어야 합니다([@sec:passive-types] 참조).

다음 예제는 주어진 이름이 `myreg`{.firrtl}이고, 유형이 `SInt`{.firrtl}이며, 클럭 신호 `myclock`{.firrtl}에 의해 구동되는 레지스터를 인스턴스화하는 것을 보여 줍니다.

``` firrtl
wire myclock: Clock
reg myreg: SInt, myclock
; ...
```

레지스터는 리셋 신호와 값으로 선언할 수 있습니다.  레지스터의 값은 리셋이 어설트될 때 리셋 값으로 업데이트됩니다.  리셋 신호는 `Reset`{.firrtl}, `UInt<1>`{.firrtl} 또는 `AsyncReset`{.firrtl}이어야 하며, 초기화 값의 유형은 선언된 레지스터의 유형과 동일해야 합니다(자세한 내용은 [@sec:type-equivalence] 참조).  레지스터의 동작은 리셋 신호의 유형에 따라 달라집니다.  `AsyncReset`{.firrtl}은 레지스터의 값을 즉시 변경합니다.  `UInt<1>`{.firrtl}은 클록 신호의 다음 포지티브 에지까지 레지스터 값을 변경하지 않습니다([@sec:reset-type] 참조).  Reset`{.firrtl}은 리셋 추론에 따라 동작이 달라지는 추상 리셋입니다.  다음 예제에서는 신호 `myreset`{.firrtl}이 높을 때 `myreg`{.firrtl}에 `myinit`{.firrtl} 값이 할당됩니다.

``` firrtl
wire myclock: Clock
wire myreset: UInt<1>
wire myinit: SInt
reg myreg: SInt, myclock with: (reset => (myreset, myinit))
; ...
```

레지스터는 불확정 값으로 초기화됩니다([@sec:indeterminate-values] 참조).

## Invalidates

invalidate 문은 회로 구성 요소에 불확정 값이 포함되어 있음을 나타내는 데 사용됩니다([@sec:불확정-값] 참조). 다음과 같이 지정됩니다:

``` firrtl
wire w: UInt
w is invalid
```

invalidate 문은 모든 유형의 회로 컴포넌트에 적용할 수 있습니다. 그러나 회로 컴포넌트에 연결할 수 없는 경우 해당 문은 컴포넌트에 아무런 영향을 미치지 않습니다. 따라서 무효화 문을 모든 컴포넌트에 적용하여 초기화 범위 오류를 명시적으로 무시할 수 있습니다.

다음 예제는 aggregate 타입의 다양한 회로 컴포넌트를 무효화할 때의 효과를 보여줍니다. 무효화 대상을 결정하는 알고리즘에 대한 자세한 내용은 [@sec:the-invalidate-algorithm]을 참조하세요.

``` firrtl
module MyModule :
  input in: {flip a: UInt, b: UInt}
  output out: {flip a: UInt, b: UInt}
  wire w: {flip a: UInt, b: UInt}
  in is invalid
  out is invalid
  w is invalid
```

는 다음과 같습니다:

``` firrtl
module MyModule :
  input in: {flip a: UInt, b: UInt}
  output out: {flip a: UInt, b: UInt}
  wire w: {flip a: UInt, b: UInt}
  in.a is invalid
  out.b is invalid
  w.a is invalid
  w.b is invalid
```

무효화된 컴포넌트의 전달은 [@sec:indeterminate-values]에서 다룹니다.

### The Invalidate Algorithm

Ground 타입이 있는 컴포넌트를 무효화하면 컴포넌트에 싱크 또는 이중 흐름이 있는 경우 컴포넌트의 값이 결정되지 않음을 나타냅니다([@sec:flows] 참조). 그렇지 않으면 컴포넌트는 영향을 받지 않습니다.

벡터 타입의 컴포넌트를 무효화하면 벡터의 각 하위 엘리먼트가 재귀적으로 무효화됩니다.

번들 타입의 컴포넌트를 무효화하면 번들의 각 하위 엘리먼트가 재귀적으로 무효화됩니다.

## Attaches

`attach`{.firrtl} 문은 두 개 이상의 아날로그 신호를 연결할 때 사용되며, 일반 연결의 방향성이 없는 정류 방식으로 값이 동일하도록 정의합니다. 아날로그 타입의 신호에만 적용할 수 있으며, 각 아날로그 신호는 0회 이상 연결할 수 있습니다.

``` firrtl
wire x: Analog<2>
wire y: Analog<2>
wire z: Analog<2>
attach(x, y)      ; binary attach
attach(z, y, x)   ; attach all three signals
```

## Nodes

노드는 단순히 회로에서 명명된 중간 값입니다. 노드는 패시브 타입의 값으로 초기화되어야 하며 연결할 수 없습니다. 노드는 복잡한 복합 표현식을 명명된 하위 표현식으로 분할하는 데 자주 사용됩니다.

다음 예는 멀티플렉서의 출력으로 초기화된 `mynode`{.firrtl}이라는 이름의 노드를 인스턴스화하는 방법을 보여줍니다([@sec:multiplexers] 참조).

``` firrtl
wire pred: UInt<1>
wire a: SInt
wire b: SInt
node mynode = mux(pred, a, b)
```

## Conditionals

이전에 선언된 구성 요소에 연결하는 조건문 내의 연결은 주어진 조건이 높을 때만 유지됩니다. 조건은 1비트 부호 없는 정수 유형이어야 합니다.

다음 예제에서 와이어 `x`{.firrtl}은 `en`{.firrtl} 신호가 높을 때만 입력 `a`{.firrtl}에 연결됩니다. 그렇지 않으면 와이어 `x`{.firrtl}은 입력 `b`{.firrtl}에 연결됩니다.

``` firrtl
module MyModule :
  input a: UInt
  input b: UInt
  input en: UInt<1>
  wire x: UInt
  when en :
    x <= a
  else :
    x <= b
```

### Syntactic Shorthands

조건문의 `else`{.firrtl} 브랜치는 생략할 수 있으며, 이 경우 빈 문으로 구성된 기본 `else`{.firrtl} 브랜치가 제공됩니다.

따라서 다음 예제는 다음과 같습니다:

``` firrtl
module MyModule :
  input a: UInt
  input b: UInt
  input en: UInt<1>
  wire x: UInt
  when en :
    x <= a
```

와 동일하게 표현할 수 있습니다:

``` firrtl
module MyModule :
  input a: UInt
  input b: UInt
  input en: UInt<1>
  wire x: UInt
  when en :
    x <= a
  else :
    skip
```

긴 조건문 체인의 가독성을 높이기 위해 `else`{.firrtl} 분기가 단일 조건문으로 구성된 경우 `else`{.firrtl} 키워드 뒤의 콜론은 생략할 수 있습니다.

따라서 다음 예제가 그렇습니다:

``` firrtl
module MyModule :
  input a: UInt
  input b: UInt
  input c: UInt
  input d: UInt
  input c1: UInt<1>
  input c2: UInt<1>
  input c3: UInt<1>
  wire x: UInt
  when c1 :
    x <= a
  else :
    when c2 :
      x <= b
    else :
      when c3 :
        x <= c
      else :
        x <= d
```

로 동일하게 작성할 수 있습니다:

``` firrtl
module MyModule :
  input a: UInt
  input b: UInt
  input c: UInt
  input d: UInt
  input c1: UInt<1>
  input c2: UInt<1>
  input c3: UInt<1>
  wire x: UInt
  when c1 :
    x <= a
  else when c2 :
    x <= b
  else when c3 :
    x <= c
  else :
    x <= d
```

가독성을 높이기 위해 `when`{.firrtl} 분기의 내용을 한 줄로 구성하는 조건문을 한 줄로 합칠 수 있습니다. `else`{.firrtl} 분기가 존재하는 경우 `else`{.firrtl} 키워드를 같은 줄에 포함해야 합니다.

다음 문장을 예로 들어보겠습니다:

``` firrtl
when c :
  a <= b
else :
  e <= f
```

는 `when`{.firrtl} 키워드, `when`{.firrtl} 브랜치, `else`{.firrtl} 키워드를 한 줄로 표현할 수 있습니다:

``` firrtl
when c : a <= b else :
  e <= f
```

`else`{.firrtl} 브랜치를 한 줄에 추가할 수도 있습니다:

``` firrtl
when c : a <= b else : e <= f
```

### Nested Declarations

컴포넌트가 조건문 내에서 선언되면 컴포넌트에 대한 연결은 조건의 영향을 받지 않습니다. 다음 예제에서 `myreg1`{.firrtl} 레지스터는 항상 `a`{.firrtl}에 연결되고, `myreg2`{.firrtl} 레지스터는 항상 `b`{.firrtl}에 연결됩니다.

``` firrtl
module MyModule :
  input a: UInt
  input b: UInt
  input en: UInt<1>
  input clk : Clock
  when en :
    reg myreg1 : UInt, clk
    myreg1 <= a
  else :
    reg myreg2 : UInt, clk
    myreg2 <= b
```

컴포넌트에 대한 연결과 해당 컴포넌트의 선언 사이에 직관적으로 선을 그릴 수 있습니다. 선이 교차하는 모든 조건문은 해당 연결에 적용됩니다.

### Initialization Coverage

조건문으로 인해 모든 조건에서 연결되지 않은 전선이 포함된 회로를 구문적으로 표현할 수 있습니다.

다음 예제에서는 `en`{.firrtl}이 높을 때 전선 `a`{.firrtl}이 전선 `w`{.firrtl}에 연결되지만, `en`{.firrtl}이 낮을 때는 `w`{.firrtl}에 무엇이 연결되는지 명시되지 않았습니다.

``` firrtl
module MyModule :
  input en: UInt<1>
  input a: UInt
  wire w: UInt
  when en :
    w <= a
```

이는 잘못된 FIRRTL 회로이며 컴파일 중에 오류가 발생합니다. 연결할 수 있는 모든 와이어, 메모리 포트, 인스턴스 포트 및 모듈 포트는 모든 조건에서 연결되어야 합니다.  레지스터는 연결되지 않은 경우 이전 값을 유지하므로 모든 조건에서 연결할 필요는 없습니다.

### Scoping

조건문은 각각의 `when`{.firrtl} 및 `else`{.firrtl} 브랜치 내에 새로운 *scope*를 생성합니다. 브랜치가 종료된 후에 브랜치 내에서 선언된 컴포넌트를 참조하는 것은 오류입니다. [@sec:namespaces]에서 언급했듯이 모듈의 회로 컴포넌트 선언은 모듈의 플랫 네임스페이스 내에서 고유해야 합니다. 즉, 조건문 내에서 같은 이름의 컴포넌트로 둘러싸는 스코프에 컴포넌트를 섀도잉하는 것은 허용되지 않습니다.

### Conditional Last Connect Semantics

회로 구성 요소에 대한 연결 뒤에 동일한 구성 요소에 대한 연결이 포함된 조건문이 뒤따르는 경우 조건이 유지될 때만 연결을 덮어씁니다. 직관적으로 멀티플렉서는 조건이 낮을 때 이전 값을 반환하고 그렇지 않으면 새 값을 반환하도록 생성됩니다.  멀티플렉서에 대한 자세한 내용은 [@sec:multiplexers]를 참조하세요.

다음 예제입니다:

``` firrtl
wire a: UInt
wire b: UInt
wire c: UInt<1>
wire w: UInt
w <= a
when c :
  w <= b
```

는 멀티플렉서를 사용하여 다음과 같이 동등하게 재작성할 수 있습니다:

``` firrtl
wire a: UInt
wire b: UInt
wire c: UInt<1>
wire w: UInt
w <= mux(c, b, a)
```

유효하지 않은 statement는 컴포넌트에 불확정 값을 할당하기 때문에, FIRRTL 컴파일러는 마지막 연결 의미를 해석할 때 불확정 값에 대해 특정 값을 자유롭게 선택할 수 있습니다.  예를 들어, 다음 회로에서 `c`{.firrtl}가 거짓일 때 `w`{.firrtl}는 불확정 값을 갖습니다.

``` firrtl
wire a: UInt
wire c: UInt<1>
wire w: UInt
w is invalid
when c :
  w <= a
```

FIRRTL 컴파일러는 `c`{.firrtl}가 거짓일 때 `w`{.firrtl}가 `a`{.firrtl}의 값을 취한다고 가정하여 이를 다음 회로로 자유롭게 최적화할 수 있습니다.

``` firrtl
wire a: UInt
wire c: UInt<1>
wire w: UInt
w <= a
```

불확정 값에 대한 자세한 내용은 [@sec:indeterminate-values]을 참조하세요.

집계 유형이 있는 회로 컴포넌트에 대한 조건부 연결의 동작은 먼저 각 연결을 해당 접지 요소의 개별 연결 문으로 확장한 다음(연결 알고리즘에 대해서는 [@sec:the-connection-algorithm] 참조) 조건부 마지막 연결 시맨틱을 적용하여 모델링할 수 있습니다.

예를 들어 다음 스니펫이 있습니다:

``` firrtl
wire x: {a: UInt, b: UInt}
wire y: {a: UInt, b: UInt}
wire c: UInt<1>
wire w: {a: UInt, b: UInt}
w <= x
when c :
  w <= y
```

는 다음과 같이 동일하게 재작성할 수 있습니다:

``` firrtl
wire x: {a:UInt, b:UInt}
wire y: {a:UInt, b:UInt}
wire c: UInt<1>
wire w: {a:UInt, b:UInt}
w.a <= mux(c, y.a, x.a)
w.b <= mux(c, y.b, x.b)
```

마지막 연결 의미론에 따른 aggregate 타입의 동작과 유사하게([@sec:last-connect-semantics] 참조), aggregate 구성 요소의 하위 요소에 대한 조건부 연결은 덮어쓰는 하위 요소에 대한 멀티플렉서만 생성합니다.

예를 들어 다음 스니펫이 있습니다:

``` firrtl
wire x: {a: UInt, b: UInt}
wire y: UInt
wire c: UInt<1>
wire w: {a: UInt, b: UInt}
w <= x
when c :
  w.a <= y
```

는 다음과 같이 동일하게 재작성할 수 있습니다:

``` firrtl
wire x: {a: UInt, b: UInt}
wire y: UInt
wire c: UInt<1>
wire w: {a: UInt, b: UInt}
w.a <= mux(c, y, x.a)
w.b <= x.b
```

## Memories

메모리는 하드웨어 메모리를 추상적으로 표현한 것입니다. 다음과 같은 매개변수가 특징입니다.

1.  메모리에 있는 각 요소의 유형을 나타내는 패시브 유형입니다.

2.  메모리에 있는 요소의 수를 나타내는 양의 정수입니다.

3.  가변적인 수의 명명된 포트, 각각 읽기 포트, 쓰기 포트 또는 읽기/쓰기 포트입니다.

4.  읽기 대기 시간을 나타내는 음수가 아닌 정수로, 포트의 읽기 대기 시간을 설정한 후의 포트의 읽기 주소를 설정한 후 해당 요소의 값을 읽을 수 있기까지의 요소의 값을 포트의 데이터 필드에서 읽을 수 있기까지의 사이클 수를 나타내는 음수가 아닌 정수입니다.

5.  포트의 쓰기 대기 시간을 나타내는 양의 정수로, 포트의 쓰기 주소를 설정한 후 사이클 수입니다. 포트의 쓰기 주소와 데이터를 설정한 후 메모리 내의 해당 엘리먼트가 메모리 내의 해당 요소가 새 값을 보유하기까지의 사이클 수를 나타내는 양수입니다.

6.  메모리 위치가 쓰여질 때 동작을 나타내는 읽기-언더-쓰기 플래그. 해당 위치에 대한 읽기가 진행 중일 때 동작을 나타내는 플래그입니다.

다음 예제에서는 실수 및 허수 구성 요소에 대해 각각 16비트 부호 있는 정수 필드가 있는 256개의 복소수가 포함된 메모리를 인스턴스화하는 방법을 보여줍니다. 이 메모리에는 두 개의 읽기 포트인 `r1`{.firrtl}과 `r2`{.firrtl}, 하나의 쓰기 포트인 `w`{.firrtl}가 있습니다. 이 포트는 조합 읽기(읽기 레이턴시는 0주기)이며 쓰기 레이턴시는 1주기입니다. 마지막으로 읽기-쓰기 동작은 정의되지 않았습니다.

``` firrtl
mem mymem :
  data-type => {real:SInt<16>, imag:SInt<16>}
  depth => 256
  reader => r1
  reader => r2
  writer => w
  read-latency => 0
  write-latency => 1
  read-under-write => undefined
```

위의 예에서 `mymem`{.firrtl}의 유형은 다음과 같습니다:

``` firrtl
{flip r1: {addr: UInt<8>,
           en: UInt<1>,
           clk: Clock,
           flip data: {real: SInt<16>, imag: SInt<16>}},
 flip r2: {addr: UInt<8>,
           en: UInt<1>,
           clk: Clock,
           flip data: {real: SInt<16>, imag: SInt<16>}},
 flip w: {addr: UInt<8>,
          en: UInt<1>,
          clk: Clock,
          data: {real: SInt<16>, imag: SInt<16>},
          mask: {real: UInt<1>, imag: UInt<1>}}}
```

다음 섹션에서는 메모리의 필드 유형이 계산되는 방법과 각 메모리 포트 유형의 동작에 대해 설명합니다.

### Read Ports

메모리가 요소 유형 `T`{.firrtl}로 선언되고 크기가 $2^N$ 이하인 경우 해당 읽기 포트의 유형은 다음과 같습니다:

``` firrtl
{addr: UInt<N>, en: UInt<1>, clk: Clock, flip data: T}
```

`en`{.firrtl} 필드가 높으면 적절한 읽기 대기 시간이 지난 후 `data`{.firrtl} 필드에서 읽어서 `addr`{.firrtl} 필드의 주소와 연결된 요소 값을 검색할 수 있습니다. en`{.firrtl} 필드가 낮으면 적절한 읽기 대기 시간 후에 `data`{.firrtl} 필드의 값이 정의되지 않은 것입니다. 포트는 `clk`{.firrtl} 필드의 클럭 신호에 의해 구동됩니다.

### Write Ports

메모리가 요소 유형 `T`{.firrtl}로 선언되고 크기가 $2^N$ 이하인 경우, 해당 쓰기 포트의 유형은 다음과 같습니다:

``` firrtl
{addr: UInt<N>, en: UInt<1>, clk: Clock, data: T, mask: M}
```

여기서 `M`{.firrtl}은 요소 유형 `T`{.firrtl}에서 계산된 마스크 유형입니다.  직관적으로 마스크 유형은 모든 접지 유형이 1비트 부호 없는 정수 유형으로 대체된 것을 제외하고는 요소 유형의 집계 구조를 반영합니다. 데이터 값의 *마스크되지 않은 부분*은 해당 마스크 리프 하위 요소가 높은 데이터 값 리프 하위 요소의 집합으로 정의됩니다.

`en`{.firrtl} 필드가 높으면 `data`{.firrtl} 필드 값의 마스크되지 않은 부분이 적절한 쓰기 대기 시간 후에 `addr`{.firrtl} 필드에 표시된 위치에 기록됩니다. en`{.firrtl} 필드가 낮으면 적절한 쓰기 레이턴시 후에 값이 기록되지 않습니다. 포트는 `clk`{.firrtl} 필드에 있는 클럭 신호에 의해 구동됩니다.

### Readwrite Ports

마지막으로 읽기/쓰기 포트에는 유형이 있습니다:

``` firrtl
{addr: UInt<N>, en: UInt<1>, clk: Clock, flip rdata: T, wmode: UInt<1>,
 wdata: T, wmask: M}
```

읽기-쓰기 포트는 주어진 사이클에서 읽기 또는 쓰기 포트로 사용할 수 있는 단일 포트입니다. 읽기-쓰기 포트가 쓰기 모드가 아닌 경우(`wmode`{.firrtl} 필드가 낮음) `rdata`{.firrtl}, `addr`{.firrtl}, `en`{.firrtl} 및 `clk`{.firrtl} 필드가 읽기 포트 필드를 구성하므로 그에 맞게 사용해야 합니다. 읽기/쓰기 포트가 쓰기 모드인 경우(`wmode`{.firrtl} 필드가 높음) `wdata`{.firrtl}, `wmask`{.firrtl}, `addr`{.firrtl}, `en`{.firrtl}, `clk`{.firrtl} 필드가 쓰기 포트 필드를 구성하므로 그에 맞게 사용해야 합니다.

### Read Under Write Behavior

read-under-write 플래그는 읽기 포트의 `data`{.firrtl} 필드에 메모리 위치를 읽는 동안 쓰면 해당 필드에 저장되는 값을 나타냅니다. 플래그는 세 가지 설정이 가능합니다: old`{.firrtl}, `new`{.firrtl}, `undefined`{.firrtl}.

read-under-write 플래그가 `old`{.firrtl}로 설정되어 있으면 읽기 포트는 항상 읽기가 요청된 것과 동일한 주기에 메모리에 존재하는 값을 반환합니다.

조합 읽기가 항상 메모리에 저장된 값을 반환한다고 가정하면(쓰기 전달 없음), 직관적으로 이것은 적절한 읽기 대기 시간만큼 지연되는 메모리로부터의 조합 읽기로 모델링됩니다.

read-under-write 플래그가 `new`{.firrtl}로 설정된 경우 읽기 포트는 항상 읽기가 사용 가능하게 된 것과 동일한 주기에 메모리에 존재하는 값을 반환합니다. 직관적으로 이것은 적절한 읽기 대기 시간만큼 읽기 주소를 지연시킨 후 메모리에서 조합하여 읽는 것으로 모델링됩니다.

read-under-write 플래그가 `undefined`{.firrtl}로 설정된 경우, 적절한 읽기 지연 시간 이후 읽기 포트가 보유하는 값은 정의되지 않습니다.

이러한 충돌을 정의하기 위해 "활성 쓰기 포트"는 주어진 클록 에지에서 쓰기 작업을 시작하는 데 사용되는 쓰기 포트 또는 읽기 쓰기 포트로, `en`{.firrtl}이 설정되어 있고 읽기 쓰기의 경우 `wmode`{.firrtl}이 설정되어 있는 포트입니다. "활성 읽기 포트"는 주어진 클록 에지에서 읽기 연산을 시작하는 데 사용되는 읽기 포트 또는 읽기 쓰기 포트로, `en`{.firrtl}이 설정되어 있고 읽기 쓰기의 경우 `wmode`{.firrtl}이 설정되어 있지 않은 경우입니다.  각 오퍼레이션은 해당 포트에 입력이 제공된 사이클부터 시작하여 해당 레이턴시에 의해 설정된 사이클 수 동안 "활성"으로 정의됩니다. 여기에는 단순히 저장된 값에서 조합적으로 선택하는 것으로 모델링되는 조합 읽기는 제외됩니다.

독립적으로 클럭이 설정된 포트가 있는 메모리의 경우, 독립적인 클럭을 가진 읽기 작업과 쓰기 작업 간의 충돌은 활성 쓰기 포트의 주소와 활성 읽기 포트의 주소가 겹치는 클럭 주기에 대해 동일하거나 읽기 작업의 일부가 주소가 일치하는 쓰기 작업의 일부와 겹치는 경우 발생하는 것으로 정의됩니다. 이러한 경우 읽기 포트에서 읽혀지는 데이터는 정의되지 않습니다.

### Write Under Write Behavior

모든 경우에 동일한 사이클에서 두 개 이상의 포트가 메모리 위치에 쓰면 저장된 값은 정의되지 않습니다.

## Instances

FIRRTL 모듈은 인스턴스 문으로 인스턴스화됩니다. 다음 예제는 최상위 모듈 `Top`{.firrtl} 내에 `MyModule`{.firrtl} 모듈의 `myinstance`{.firrtl}라는 인스턴스를 생성하는 방법을 보여줍니다.

``` firrtl
circuit Top :
  module MyModule :
    input a: UInt
    output b: UInt
    b <= a
  module Top :
    inst myinstance of MyModule
```

결과 인스턴스에는 번들 유형이 있습니다. 인스턴스화된 모듈의 각 포트는 번들에서 포트와 동일한 이름 및 유형을 가진 필드로 표시됩니다. 입력 포트에 해당하는 필드는 반전되어 데이터가 출력 포트와 반대 방향으로 흐른다는 것을 나타냅니다.  위 예제에서 `myinstance`{.firrtl} 인스턴스의 유형은 `{flip a:UInt, b:UInt}`입니다.

모듈은 회로의 의미론에 영향을 주지 않고 인스턴스가 항상 부모 모듈에 *인라인*될 수 있다는 속성을 가지고 있습니다.

무한 재귀 하드웨어를 허용하지 않기 위해 모듈은 직접적으로 또는 인스턴스화한 다른 모듈의 인스턴스를 통해 간접적으로 자신의 인스턴스를 포함할 수 없습니다.

## Stops

stop 문은 회로의 시뮬레이션을 중지하는 데 사용됩니다. 백엔드에서는 디버깅을 위해 실행 중인 회로를 중지하는 하드웨어를 자유롭게 생성할 수 있지만, FIRRTL 사양에서는 이를 요구하지 않습니다.

중지 문에는 클럭 신호, 단일 비트 부호 없는 정수 타입의 중지 조건 신호, 정수 종료 코드가 필요합니다.

환경에서 부작용이 있는 클럭 문(중지, 인쇄 및 확인 문)의 경우 동일한 클럭 에지에서 트리거되는 이러한 문의 실행 순서는 둘러싸는 모듈의 구문 순서에 따라 결정됩니다. 서로 다른 모듈에 있거나 동시에 트리거되는 서로 다른 클럭을 가진 클럭이 있는 부작용이 있는 문의 실행 순서는 정의되지 않습니다.

stop 문에는 문에 메타데이터를 첨부하는 데 사용할 수 있는 선택적 이름 속성이 있습니다. 이름은 모듈 수준 네임스페이스의 일부입니다. 그러나 유효한 유형이 아니므로 참조에서는 사용할 수 없습니다.

``` firrtl
wire clk: Clock
wire halt: UInt<1>
stop(clk, halt, 42) : optional_name
```

## Formatted Prints

형식이 지정된 인쇄 문은 회로 시뮬레이션 중에 형식이 지정된 문자열을 인쇄하는 데 사용됩니다. 백엔드는 이 정보를 하드웨어 테스트 하네스에 전달하는 하드웨어를 자유롭게 생성할 수 있지만, FIRRTL 사양에서는 이를 요구하지 않습니다.

`printf`{.firrtl} 문에는 클록 신호, 인쇄 조건 신호, 형식 문자열 및 가변 인수 신호 목록이 필요합니다. 조건 신호는 단일 비트 부호 없는 정수 유형이어야 하며, 인자 신호는 각각 접지 유형이어야 합니다.

관찰 가능한 환경 부작용이 있는 클럭 문의 실행 순서에 대한 자세한 내용은 [@sec:stops]를 참조하십시오.

`printf`{.firrtl} 문에는 문에 메타데이터를 첨부하는 데 사용할 수 있는 선택적 이름 속성이 있습니다. 이름은 모듈 수준 네임스페이스의 일부입니다. 그러나 유효한 유형이 아니므로 참조에서는 사용할 수 없습니다.

``` firrtl
wire clk: Clock
wire cond: UInt<1>
wire a: UInt
wire b: UInt
printf(clk, cond, "a in hex: %x, b in decimal:%d.\n", a, b) : optional_name
```

각 양의 클록 에지에서 조건 신호가 높으면 `printf`{.firrtl} 문은 인수 자리 표시자가 해당 인수의 값으로 대체된 형식 문자열을 출력합니다.

### Format Strings

Format strings support the following argument placeholders:

- `%b` : Prints the argument in binary

- `%d` : Prints the argument in decimal

- `%x` : Prints the argument in hexadecimal

- `%%` : Prints a single `%` character

Format strings support the following escape characters:

- `\n` : New line

- `\t` : Tab

- `\\` : Back slash

- `\"` : Double quote

- `\'` : Single quote

## Verification

시뮬레이션, 모델 확인 및 공식적인 방법을 용이하게 하기 위해 다음 세 가지를 사용할 수 있습니다. 합성할 수 없는 검증 문이 있습니다: 주장, 가정, 커버. 각 유형의 검증 문에는 클럭 신호, 술어 신호, 활성화 신호 및 문자열 리터럴이 필요합니다.  술어 및 활성화 신호는 단일 비트 부호 없는 정수 유형이어야 합니다.  어서트 및 가정은 문자열을 설명 메시지로 사용합니다.  커버문의 경우 문자열은 제안된 설명을 나타냅니다.  어설트 또는 가정이 위반되면 설명 메시지가 지침으로 발행될 수 있습니다.  설명 메시지는 "다음을 확인합니다\..."라는 단어가 접두사로 붙는 것처럼 표현될 수 있습니다.

백엔드는 대상 언어로 해당 모델 검사 구문을 자유롭게 생성할 수 있지만 FIRRTL 사양에서는 이를 요구하지 않습니다. 이러한 구문을 생성하지 않는 백엔드는 경고를 발행해야 합니다. 예를 들어, SystemVerilog 이미터는 SystemVerilog 주장, 가정 및 커버 문을 생성하지만 Verilog 이미터는 생성하지 않고 대신 확인 문이 발생하면 사용자에게 경고를 표시합니다.

관찰 가능한 환경 부작용이 있는 클럭된 문의 실행 순서에 대한 자세한 내용은 [@sec:stops]를 참조하세요.

모든 확인 문에는 문에 메타데이터를 첨부하는 데 사용할 수 있는 선택적 이름 속성이 있습니다. 이름은 모듈 수준 네임스페이스의 일부입니다. 그러나 유효한 유형이 아니므로 참조에는 사용할 수 없습니다.

### Assert

어설트 문은 활성화가 참일 때 모든 클록 사이클의 상승 에지에서 술어가 참인지 확인합니다. 즉, 활성화가 술어를 내포하는지 확인합니다.

``` firrtl
wire clk: Clock
wire pred: UInt<1>
wire en: UInt<1>
pred <= eq(X, Y)
en <= Z_valid
assert(clk, pred, en, "X equals Y when Z is valid") : optional_name
```

### Assume

assume statement은 클록 사이클의 상승 에지에서 enable이 참이고 predicate이 참이 아닌 모든 상태를 무시하도록 모델 검사기에 지시합니다. 즉, 검사할 상태를 enable이 정의상 predicate가 true임을 암시하는 상태로만 축소합니다. 시뮬레이션에서 assume은 assert로 취급됩니다.

``` firrtl
wire clk: Clock
wire pred: UInt<1>
wire en: UInt<1>
pred <= eq(X, Y)
en <= Z_valid
assume(clk, pred, en, "X equals Y when Z is valid") : optional_name
```

### Cover

커버 문은 enable이 true일 때 일부 클록 사이클의 상승 에지에서 술어가 true인지 확인합니다. 즉, 모델 검사기에 특정 시간 단계에서 enable과 predicate를 모두 true로 만드는 방법을 찾도록 지시합니다. 문자열 인수는 cover와 함께 코멘트로 전달될 수 있습니다.

``` firrtl
wire clk: Clock
wire pred: UInt<1>
wire en: UInt<1>
pred <= eq(X, Y)
en <= Z_valid
cover(clk, pred, en, "X equals Y when Z is valid") : optional_name
```

# Expressions

FIRRTL 표현식은 리터럴 부호 없는 정수 및 부호 있는 정수 생성, 선언된 회로 컴포넌트 참조, 컴포넌트 내의 중첩된 요소 정적 및 동적 액세스, 멀티플렉서 생성, 프리미티브 연산 수행에 사용됩니다.

## Unsigned Integers

음수가 아닌 정수 값과 양수 비트 폭(선택 사항)이 주어지면 리터럴 부호 없는 정수를 생성할 수 있습니다. 다음 예제는 숫자 42를 나타내는 10비트 부호 없는 정수를 생성합니다.

``` firrtl
UInt<10>(42)
```

주어진 값에 충분히 크지 않은 비트 폭을 입력하는 것은 오류입니다. 비트 폭을 생략하면 주어진 값에 맞는 데 필요한 최소 비트 수가 추론됩니다.

``` firrtl
UInt(42)
```

## Unsigned Integers from Literal Bits

또는 비트 표현을 나타내는 문자열과 선택적 비트 폭이 주어지면 리터럴 부호 없는 정수를 생성할 수 있습니다.

지원되는 래디스는 다음과 같습니다:

1.`b`{.firrtl}: 이진수 표현용.

2.`o`{.firrtl} : 8진수 표현용.

3.`h`{.firrtl} : 16진수 표현용.

비트 폭을 지정하지 않으면 비트 표현의 비트 수가 문자열로 직접 표현됩니다. 다음 예제는 숫자 13을 나타내는 8비트 정수를 생성합니다.

``` firrtl
UInt("b00001101")
UInt("h0D")
```

제공된 비트 폭이 문자열 값을 나타내는 데 필요한 비트 수보다 큰 경우 결과 값은 제공된 비트 폭까지 0으로 확장된 문자열과 동일합니다. 제공된 비트 폭이 문자열로 표시되는 비트 수보다 작은 경우 결과 값은 제공된 비트 폭으로 잘린 문자열과 같습니다. 잘린 비트는 모두 0이어야 합니다.

다음 예에서는 숫자 13을 나타내는 7비트 정수를 생성합니다.

``` firrtl
UInt<7>("b00001101")
UInt<7>("o015")
UInt<7>("hD")
```

## Signed Integers

부호 없는 정수와 마찬가지로 정수 값과 양수 비트 폭(선택 사항)이 주어지면 리터럴 부호 정수를 만들 수 있습니다. 다음 예에서는 숫자 -42를 나타내는 10비트 부호 없는 정수를 생성합니다.

``` firrtl
SInt<10>(-42)
```

2의 보수 표현을 사용하여 주어진 값에 맞출 만큼 충분히 크지 않은 비트 폭을 제공하는 것은 오류입니다. 비트 폭을 생략하면 주어진 값에 맞는 데 필요한 최소 비트 수가 추론됩니다.

``` firrtl
SInt(-42)
```

## Signed Integers from Literal Bits

부호가 없는 정수와 마찬가지로, 비트 표현을 나타내는 문자열과 선택적 비트 폭이 주어지면 리터럴 부호 정수를 만들 수도 있습니다.

비트 표현에는 이진, 8진수 또는 16진수 표시기 다음에 선택적 부호, 값이 포함됩니다.

비트 폭을 지정하지 않으면 비트 표현의 비트 수가 문자열로 표시되는 값을 나타내는 최소 비트 폭이 됩니다. 다음 예에서는 숫자 -13을 나타내는 8비트 정수를 생성합니다. 모든 염기에 대해 음수 부호는 단항 부호인 것처럼 작동합니다. 즉, 음수 리터럴은 숫자 패턴의 부호 없는 해석의 덧셈 역을 생성합니다.

``` firrtl
SInt("b-1101")
SInt("h-d")
```

제공된 비트 폭이 문자열로 표시되는 비트 수보다 크면 결과 값이 변경되지 않습니다. 문자열 값을 나타내는 데 필요한 비트 수보다 작은 비트 폭을 제공하면 오류가 발생합니다.

## References

레퍼런스는 단순히 이전에 선언된 회로 컴포넌트를 가리키는 이름입니다. 모듈 포트, 노드, 와이어, 레지스터, 인스턴스 또는 메모리를 나타낼 수 있습니다.

다음 예는 이전에 선언된 포트 `in`{.firrtl}을 참조하는 참조 표현식 `in`{.firrtl}을 이전에 선언된 포트 `out`{.firrtl}을 참조하는 참조 표현식 `out`{.firrtl}에 연결합니다.

``` firrtl
module MyModule :
  input in: UInt
  output out: UInt
  out <= in
```

이 문서의 나머지 부분에서는 간결성을 위해 컴포넌트의 이름을 해당 컴포넌트에 대한 참조 표현식을 참조하는 데 사용할 것입니다. 따라서 위의 예제는 "포트 `in`{.firrtl}은 포트 `out`{.firrtl}에 연결됩니다."로 다시 작성됩니다.

## Sub-fields

하위 필드 표현식은 번들 유형이 있는 표현식의 하위 요소를 나타냅니다.

다음 예는 `in`{.firrtl} 포트를 `out`{.firrtl} 포트의 `a`{.firrtl} 하위 요소에 연결합니다.

``` firrtl
module MyModule :
  input in: UInt
  output out: {a: UInt, b: UInt}
  out.a <= in
```

## Sub-indices

하위 인덱스 표현식은 인덱스를 통해 벡터 유형을 가진 표현식의 하위 요소를 정적으로 참조합니다. 인덱스는 음수가 아닌 정수여야 하며 인덱싱하는 벡터의 길이와 같거나 초과할 수 없습니다.

다음 예제는 `in`{.firrtl} 포트를 `out`{.firrtl} 포트의 다섯 번째 하위 요소에 연결합니다.

``` firrtl
module MyModule :
  input in: UInt
  output out: UInt[10]
  out[4] <= in
```

## Sub-accesses

하위 액세스 표현식은 계산된 인덱스를 사용하여 벡터 유형 표현식의 하위 요소를 동적으로 참조합니다. 인덱스는 부호 없는 정수 타입의 표현식이어야 합니다.  범위를 벗어난 요소에 액세스하면 불확정 값이 생성됩니다([@sec:indeterminate-values] 참조).  범위를 벗어난 요소는 각각 다른 불확정 값입니다.  상수 인덱스가 있는 하위 액세스 연산은 범위를 벗어난 불확정 값에 대한 동작을 컴파일 타임 오류로 변환하더라도 하위 인덱스 연산으로 변환될 수 있습니다.

다음 예제는 `in`{.firrtl} 포트의 n번째 하위 요소를 `out`{.firrtl} 포트에 연결합니다.

``` firrtl
module MyModule :
  input in: UInt[3]
  input n: UInt<2>
  output out: UInt
  out <= in[n]
```

하위 액세스 표현식의 연결은 벡터의 모든 하위 요소에서 조건부로 연결하여 모델링할 수 있으며, 여기서 조건은 동적 인덱스가 하위 요소의 정적 인덱스와 같을 때 유지됩니다.

``` firrtl
module MyModule :
  input in: UInt[3]
  input n: UInt<2>
  output out: UInt
  when eq(n, UInt(0)) :
    out <= in[0]
  else when eq(n, UInt(1)) :
    out <= in[1]
  else when eq(n, UInt(2)) :
    out <= in[2]
  else :
    out is invalid
```

다음 예는 `in`{.firrtl} 포트를 `out`{.firrtl} 포트의 n번째 하위 요소에 연결합니다. `out`{.firrtl} 포트의 다른 모든 하위 요소는 `default`{.firrtl} 포트의 해당 하위 요소에서 연결됩니다.

``` firrtl
module MyModule :
  input in: UInt
  input default: UInt[3]
  input n: UInt<2>
  output out: UInt[3]
  out <= default
  out[n] <= in
```

하위 액세스 표현식에 대한 연결은 벡터의 모든 하위 요소에 조건부로 연결하여 모델링할 수 있으며, 여기서 조건은 동적 인덱스가 하위 요소의 정적 인덱스와 같을 때 유지됩니다.

``` firrtl
module MyModule :
  input in: UInt
  input default: UInt[3]
  input n: UInt<2>
  output out: UInt[3]
  out <= default
  when eq(n, UInt(0)) :
    out[0] <= in
  else when eq(n, UInt(1)) :
    out[1] <= in
  else when eq(n, UInt(2)) :
    out[2] <= in
```

다음 예는 `in`{.firrtl} 포트를 `out`{.firrtl} 포트의 n번째 벡터 유형 하위 요소의 m번째 `UInt`{.firrtl} 하위 요소에 연결합니다. `out`{.firrtl} 포트의 다른 모든 하위 요소는 `default`{.firrtl} 포트의 해당 하위 요소에서 연결됩니다.

``` firrtl
module MyModule :
  input in: UInt
  input default: UInt[2][2]
  input n: UInt<1>
  input m: UInt<1>
  output out: UInt[2][2]
  out <= default
  out[n][m] <= in
```

여러 개의 중첩된 하위 액세스 표현식이 포함된 표현식에 대한 연결은 표현식의 모든 하위 요소에 조건부로 연결하여 모델링할 수도 있습니다. 그러나 이 조건은 모든 동적 인덱스가 모든 하위 요소의 정적 인덱스와 같을 때만 유지됩니다.

``` firrtl
module MyModule :
  input in: UInt
  input default: UInt[2][2]
  input n: UInt<1>
  input m: UInt<1>
  output out: UInt[2][2]
  out <= default
  when and(eq(n, UInt(0)), eq(m, UInt(0))) :
    out[0][0] <= in
  else when and(eq(n, UInt(0)), eq(m, UInt(1))) :
    out[0][1] <= in
  else when and(eq(n, UInt(1)), eq(m, UInt(0))) :
    out[1][0] <= in
  else when and(eq(n, UInt(1)), eq(m, UInt(1))) :
    out[1][1] <= in
```

## Multiplexers

멀티플렉서는 부호 없는 선택 신호의 값에 따라 두 가지 입력 표현식 중 하나를 출력합니다.

다음 예는 `a`{.firrtl} 포트와 `b`{.firrtl} 포트 중 하나를 선택한 결과를 `c`{.firrtl} 포트에 연결합니다. `sel`{.firrtl} 신호가 높으면 `a`{.firrtl} 포트가 선택되고, 그렇지 않으면 `b`{.firrtl} 포트가 선택됩니다.

``` firrtl
module MyModule :
  input a: UInt
  input b: UInt
  input sel: UInt<1>
  output c: UInt
  c <= mux(sel, a, b)
```

멀티플렉서 표현식은 다음 조건이 충족되는 경우에만 합법적입니다.

1. 선택 신호의 유형이 부호 없는 정수입니다.

1. 선택 신호의 폭은 다음 중 하나입니다:

    1. 1비트

    1. 지정되지 않았지만 1비트로 유추합니다.

1. 두 입력 표현식의 유형은 동일합니다.

1. 두 입력 표현식의 유형은 패시브입니다([@sec:passive-types] 참조).

## Primitive Operations

접지 타입에 대한 모든 기본 연산은 FIRRTL 프리미티브 연산으로 표현됩니다. 일반적으로 각 연산은 몇 개의 정적 정수 리터럴 매개변수와 함께 몇 개의 인수 표현식을 취합니다.

기본 연산의 일반적인 형태는 다음과 같이 표현됩니다:

``` firrtl
op(arg0, arg1, ..., argn, int0, int1, ..., intm)
```

다음 프리미티브 연산 예제에서는 `a`{.firrtl}와 `b`{.firrtl}라는 두 표현식을 더하고, 표현식 `a`{.firrtl}을 3비트씩 왼쪽으로 이동하고, 네 번째 비트부터 일곱 번째 비트까지 선택하여 `a`{.firrtl} 표현식에 포함시키고, 표현식 `x`{.firrtl}을 시계 타입 신호로 해석하는 것을 보여 줍니다.

``` firrtl
add(a, b)
shl(a, 3)
bits(a, 7, 4)
asClock(x)
```

[@sec:primitive-operations]는 각 프리미티브 연산의 형식과 의미를 설명합니다.

# Primitive Operations {#sec:primitive-operations}

모든 기본 연산의 인수는 기본 유형이 있는 표현식이어야 하며, 매개변수는 정적 정수 리터럴이어야 합니다. 각 특정 연산은 인자와 매개변수의 수와 유형에 추가적인 제한을 둘 수 있습니다.

표기법상 인자 e의 폭은 w~e~로 표시됩니다.

## Add Operation

| Name | Arguments | Parameters | Arg Types     | Result Type | Result Width                |
|------|-----------|------------|---------------|-------------|-----------------------------|
| add  | (e1,e2)   | ()         | (UInt,UInt)   | UInt        | max(w~e1~,w~e2~)+1          |
|      |           |            | (SInt,SInt)   | SInt        | max(w~e1~,w~e2~)+1          |

더하기 연산 결과는 정밀도 손실 없이 e1과 e2의 합이 됩니다.

## Subtract Operation


| Name | Arguments | Parameters | Arg Types     | Result Type | Result Width                |
|------|-----------|------------|---------------|-------------|-----------------------------|
| sub  | (e1,e2)   | ()         | (UInt,UInt)   | UInt        | max(w~e1~,w~e2~)+1          |
|      |           |            | (SInt,SInt)   | SInt        | max(w~e1~,w~e2~)+1          |

빼기 연산 결과는 정밀도 손실 없이 e1에서 e2를 뺀 값입니다.

## Multiply Operation

| Name | Arguments | Parameters | Arg Types     | Result Type | Result Width                |
|------|-----------|------------|---------------|-------------|-----------------------------|
| mul  | (e1,e2)   | ()         | (UInt,UInt)   | UInt        | w~e1~+w~e2~                 |
|      |           |            | (SInt,SInt)   | SInt        | w~e1~+w~e2~                 |

곱하기 연산 결과는 정밀도 손실 없이 e1과 e2의 곱입니다.

## Divide Operation


| Name | Arguments | Parameters | Arg Types   | Result Type | Result Width |
|------|-----------|------------|-------------|-------------|--------------|
| div  | (num,den) | ()         | (UInt,UInt) | UInt        | w~num~       |
|      |           |            | (SInt,SInt) | SInt        | w~num~+1     |

나누기 연산은 num을 den으로 나누어 결과에서 분수 부분을 잘라냅니다. 이는 결과를 0으로 반올림하는 것과 같습니다. den이 0인 나눗셈의 결과는 정의되지 않습니다.

## Modulus Operation

| Name | Arguments | Parameters | Arg Types   | Result Type | Result Width       |
|------|-----------|------------|-------------|-------------|--------------------|
| rem  | (num,den) | ()         | (UInt,UInt) | UInt        | min(w~num~,w~den~) |
|      |           |            | (SInt,SInt) | SInt        | min(w~num~,w~den~) |

모듈러스 연산은 분자의 부호를 유지하면서 num을 den으로 나눈 나머지를 산출합니다. 모듈러스 연산자는 나누기 연산자와 함께 아래 관계를 만족합니다:

    num = add(mul(den,div(num,den)),rem(num,den))}.

## Comparison Operations

| Name   | Arguments | Parameters | Arg Types     | Result Type | Result Width |
|--------|-----------|------------|---------------|-------------|--------------|
| lt,leq |           |            | (UInt,UInt)   | UInt        | 1            |
| gt,geq | (e1,e2)   | ()         | (SInt,SInt)   | UInt        | 1            |

비교 연산은 e1이 (lt)보다 작거나 (leq)보다 작거나 같거나 (gt)보다 크거나 (geq)보다 크거나 같거나 (eq)와 같거나 (neq)와 같지 않은 경우 값 1을 갖는 부호 없는 1비트 신호를 반환합니다.  그렇지 않으면 연산은 0 값을 반환합니다.

## Padding Operations

| Name | Arguments | Parameters | Arg Types | Result Type | Result Width                |
|------|-----------|------------|-----------|-------------|-----------------------------|
| pad  | \(e\)     | \(n\)      | (UInt)    | UInt        | max(w~e~,n)                 |
|      |           |            | (SInt)    | SInt        | max(w~e~,n)                 |


e의 비트 폭이 n보다 작으면 패드 연산은 주어진 폭 n까지 e를 0으로 확장하거나 부호로 확장합니다. 그렇지 않으면 결과는 단순히 e가 됩니다. n은 음수가 아니어야 합니다.

## Interpret As UInt

| Name   | Arguments | Parameters | Arg Types    | Result Type | Result Width |
|--------|-----------|------------|--------------|-------------|--------------|
| asUInt | \(e\)     | ()         | (UInt)       | UInt        | w~e~         |
|        |           |            | (SInt)       | UInt        | w~e~         |
|        |           |            | (Clock)      | UInt        | 1            |
|        |           |            | (Reset)      | UInt        | 1            |
|        |           |            | (AsyncReset) | UInt        | 1            |

Interpret as UInt 연산은 e의 비트를 부호 없는 정수로 재해석합니다.

## Interpret As SInt

| Name   | Arguments | Parameters | Arg Types    | Result Type | Result Width |
|--------|-----------|------------|--------------|-------------|--------------|
| asSInt | \(e\)     | ()         | (UInt)       | SInt        | w~e~         |
|        |           |            | (SInt)       | SInt        | w~e~         |
|        |           |            | (Clock)      | SInt        | 1            |
|        |           |            | (Reset)      | SInt        | 1            |
|        |           |            | (AsyncReset) | SInt        | 1            |

Interpret as SInt 연산은 2의 보수 표현에 따라 e의 비트를 부호 있는 정수로 재해석합니다.

## Interpret as Clock

| Name    | Arguments | Parameters | Arg Types    | Result Type | Result Width |
|---------|-----------|------------|--------------|-------------|--------------|
| asClock | \(e\)     | ()         | (UInt)       | Clock       | n/a          |
|         |           |            | (SInt)       | Clock       | n/a          |
|         |           |            | (Clock)      | Clock       | n/a          |
|         |           |            | (Reset)      | Clock       | n/a          |
|         |           |            | (AsyncReset) | Clock       | n/a          |

클럭 연산으로 해석의 결과는 단일 비트 정수를 클럭 신호로 해석하여 얻은 클럭 유형 신호입니다.

## Interpret as AsyncReset

| Name         | Arguments | Parameters | Arg Types    | Result Type | Result Width |
|--------------|-----------|------------|--------------|-------------|--------------|
| asAsyncReset | \(e\)     | ()         | (AsyncReset) | AsyncReset  | n/a          |
|              |           |            | (UInt)       | AsyncReset  | n/a          |
|              |           |            | (SInt)       | AsyncReset  | n/a          |
|              |           |            | (Interval)   | AsyncReset  | n/a          |
|              |           |            | (Clock)      | AsyncReset  | n/a          |
|              |           |            | (Reset)      | AsyncReset  | n/a          |

비동기 리셋 작업으로 해석한 결과는 비동기 리셋 유형 신호인 AsyncReset입니다.

## Shift Left Operation

| Name | Arguments | Parameters | Arg Types | Result Type | Result Width                |
|------|-----------|------------|-----------|-------------|-----------------------------|
| shl  | \(e\)     | \(n\)      | (UInt)    | UInt        | w~e~+n                      |
|      |           |            | (SInt)    | SInt        | w~e~+n                      |

shift left 연산은 0비트 n개를 e의 최하위 끝으로 연결합니다. n은 음수가 아니어야 합니다.

## Shift Right Operation

| Name | Arguments | Parameters | Arg Types | Result Type | Result Width                |
|------|-----------|------------|-----------|-------------|-----------------------------|
| shr  | \(e\)     | \(n\)      | (UInt)    | UInt        | max(w~e~-n, 1)              |
|      |           |            | (SInt)    | SInt        | max(w~e~-n, 1)              |

shift right 연산은 e에서 최하위 n 비트를 잘라냅니다. n이 e의 비트 폭보다 크거나 같으면 결과 값은 부호 없는 유형은 0이 되고 부호 있는 유형은 부호 비트가 됩니다. n은 음수가 아닌 값이어야 합니다.

## Dynamic Shift Left Operation

| Name | Arguments | Parameters | Arg Types     | Result Type | Result Width                |
|------|-----------|------------|---------------|-------------|-----------------------------|
| dshl | (e1, e2)  | ()         | (UInt, UInt)  | UInt        | w~e1~ + 2`^`w~e2~ - 1       |
|      |           |            | (SInt, UInt)  | SInt        | w~e1~ + 2`^`w~e2~ - 1       |

dynamic shift left 연산은 e1 e2 위치의 비트를 가장 중요한 비트로 이동시키고, e2 0은 가장 중요하지 않은 비트로 이동시킵니다.

## Dynamic Shift Right Operation

| Name | Arguments | Parameters | Arg Types     | Result Type | Result Width                |
|------|-----------|------------|---------------|-------------|-----------------------------|
| dshr | (e1, e2)  | ()         | (UInt, UInt)  | UInt        | w~e1~                       |
|      |           |            | (SInt, UInt)  | SInt        | w~e1~                       |

The dynamic shift right operation shifts the bits in e1 e2 places towards the least significant bit. e2 signed or zeroed bits are shifted in to the most significant bits, and the e2 least significant bits are truncated.

## Arithmetic Convert to Signed Operation

| Name | Arguments | Parameters | Arg Types | Result Type | Result Width |
|------|-----------|------------|-----------|-------------|--------------|
| cvt  | \(e\)     | ()         | (UInt)    | SInt        | w~e~+1       |
|      |           |            | (SInt)    | SInt        | w~e~         |

The result of the arithmetic convert to signed operation is a signed integer representing the same numerical value as e.

## Negate Operation

| Name | Arguments | Parameters | Arg Types | Result Type | Result Width |
|------|-----------|------------|-----------|-------------|--------------|
| neg  | \(e\)     | ()         | (UInt)    | SInt        | w~e~+1       |
|      |           |            | (SInt)    | SInt        | w~e~+1       |

The result of the negate operation is a signed integer representing the negated numerical value of e.

## Bitwise Complement Operation

| Name | Arguments | Parameters | Arg Types | Result Type | Result Width |
|------|-----------|------------|-----------|-------------|--------------|
| not  | \(e\)     | ()         | (UInt)    | UInt        | w~e~         |
|      |           |            | (SInt)    | UInt        | w~e~         |

The bitwise complement operation performs a logical not on each bit in e.

## Binary Bitwise Operations

| Name       | Arguments | Parameters | Arg Types   | Result Type | Result Width     |
|------------|-----------|------------|-------------|-------------|------------------|
| and,or,xor | (e1, e2)  | ()         | (UInt,UInt) | UInt        | max(w~e1~,w~e2~) |
|            |           |            | (SInt,SInt) | UInt        | max(w~e1~,w~e2~) |

The above bitwise operations perform a bitwise and, or, or exclusive or on e1 and e2. The result has the same width as its widest argument, and any narrower arguments are automatically zero-extended or sign-extended to match the width of the result before performing the operation.

## Bitwise Reduction Operations


| Name          | Arguments | Parameters | Arg Types | Result Type | Result Width |
|---------------|-----------|------------|-----------|-------------|--------------|
| andr,orr,xorr | \(e\)     | ()         | (UInt)    | UInt        | 1            |
|               |           |            | (SInt)    | UInt        | 1            |

The bitwise reduction operations correspond to a bitwise and, or, and exclusive or operation, reduced over every bit in e.

In all cases, the reduction incorporates as an inductive base case the "identity value" associated with each operator. This is defined as the value that preserves the value of the other argument: one for and (as $x \wedge 1 = x$), zero for or (as $x \vee 0 = x$), and zero for xor (as $x \oplus 0 = x$). Note that the logical consequence is that the and-reduction of a zero-width expression returns a one, while the or- and xor-reductions of a zero-width expression both return zero.

## Concatenate Operation

| Name | Arguments | Parameters | Arg Types      | Result Type | Result Width |
|------|-----------|------------|----------------|-------------|--------------|
| cat  | (e1,e2)   | ()         | (UInt, UInt)   | UInt        | w~e1~+w~e2~  |
|      |           |            | (SInt, SInt)   | UInt        | w~e1~+w~e2~  |

The result of the concatenate operation is the bits of e1 concatenated to the most significant end of the bits of e2.

## Bit Extraction Operation

| Name | Arguments | Parameters | Arg Types | Result Type | Result Width |
|------|-----------|------------|-----------|-------------|--------------|
| bits | \(e\)     | (hi,lo)    | (UInt)    | UInt        | hi-lo+1      |
|      |           |            | (SInt)    | UInt        | hi-lo+1      |

The result of the bit extraction operation are the bits of e between lo (inclusive) and hi (inclusive). hi must be greater than or equal to lo.  Both hi and lo must be non-negative and strictly less than the bit width of e.

## Head

| Name | Arguments | Parameters | Arg Types | Result Type | Result Width |
|------|-----------|------------|-----------|-------------|--------------|
| head | \(e\)     | \(n\)      | (UInt)    | UInt        | n            |
|      |           |            | (SInt)    | UInt        | n            |

The result of the head operation are the n most significant bits of e. n must be non-negative and less than or equal to the bit width of e.

## Tail

| Name | Arguments | Parameters | Arg Types | Result Type | Result Width |
|------|-----------|------------|-----------|-------------|--------------|
| tail | \(e\)     | \(n\)      | (UInt)    | UInt        | w~e~-n       |
|      |           |            | (SInt)    | UInt        | w~e~-n       |

The tail operation truncates the n most significant bits from e. n must be non-negative and less than or equal to the bit width of e.

# Flows

An expression's flow partially determines the legality of connecting to and from the expression. Every expression is classified as either *source*, *sink*, or *duplex*. For details on connection rules refer back to [@sec:connects].

The flow of a reference to a declared circuit component depends on the kind of circuit component. A reference to an input port, an instance, a memory, and a node, is a source. A reference to an output port is a sink. A reference to a wire or register is duplex.

The flow of a sub-index or sub-access expression is the flow of the vector-typed expression it indexes or accesses.

The flow of a sub-field expression depends upon the orientation of the field. If the field is not flipped, its flow is the same flow as the bundle-typed expression it selects its field from. If the field is flipped, then its flow is the reverse of the flow of the bundle-typed expression it selects its field from. The reverse of source is sink, and vice-versa. The reverse of duplex remains duplex.

The flow of all other expressions are source.

# Width Inference

너비가 지정되지 않은 채 선언된 모든 회로 컴포넌트의 경우, FIRRTL 컴파일러는 들어오는 모든 연결의 적법성을 유지하는 가능한 최소 너비를 추론합니다. 컴포넌트에 들어오는 연결이 없고 너비가 지정되지 않은 경우 너비를 유추할 수 없음을 나타내는 오류가 발생합니다.

너비가 지정되지 않은 모듈 입력 포트의 경우, 유추된 너비는 모듈의 모든 인스턴스에 대한 모든 수신 연결의 적법성을 유지하는 가능한 최소 너비입니다.

접지형 멀티플렉서 표현식의 폭은 해당 입력 폭 두 개 중 최대값입니다. 집계 유형 다중화 표현식의 경우, 각 리프 하위 요소의 결과 너비는 해당 두 입력 리프 하위 요소 너비의 최대값입니다.

각 프리미티브 연산의 폭은 [@sec:primitive-operations]에 자세히 설명되어 있습니다.

정수 리터럴 표현식의 폭은 해당 섹션에 자세히 설명되어 있습니다.

# Combinational Loops

조합 논리는 게이트 사이에 레지스터가 없는 논리 섹션입니다. 조합 루프는 일부 조합 로직의 출력이 개입 레지스터 없이 해당 조합 로직의 입력으로 피드백될 때 존재합니다. 실제 멀티플렉서 선택 값에서 루프가 존재하지 않음을 보여줄 수 있는 경우에도 FIRRTL은 조합 루프를 지원하지 않습니다. 조합 루프는 허용되지 않으며 설계에서 이러한 조합 루프를 제거하거나 중단하기 위해 FIRRTL 변환에 의존해서는 안 됩니다.

모듈 `Foo`에는 조합 루프가 있으며 마지막 연결 의미론에 의해 루프가 제거되더라도 합법적이지 않습니다.
``` firrtl
  module Foo:
    input a: UInt<1>
    output b: UInt<1>
    b <= b
    b <= a
 ```

다음 모듈 `Foo2`는 `n1`과 `n2`가 겹치지 않는다는 것을 증명할 수 있더라도 조합 루프를 가지고 있습니다.
``` firrtl
module Foo2 :
  input n1: UInt<2>
  input n2: UInt<2>
  wire tmp: UInt<1>
  wire vec: UInt<1>[3]
  tmp <= vec[n1]
  vec[n2] <= tmp
```

모듈 `Foo3`은 비트 레벨이 아닌 단어 레벨에만 존재하더라도 불법적인 조합 루프의 또 다른 예입니다.

```firrtl
module Foo3
  wire a : UInt<2>
  wire b : UInt<1>

  a <= cat(b, c)
  b <= bits(a, 0, 0)

```


# Namespaces

회로의 모든 모듈은 동일한 모듈 네임스페이스에 존재하므로 모두 고유한 이름을 가져야 합니다.

각 모듈에는 모든 포트 및 회로 컴포넌트 선언의 이름이 포함된 식별자 네임스페이스가 있습니다. 따라서 모듈 내의 모든 선언은 고유한 이름을 가져야 합니다.

번들 유형 선언 내에서는 모든 필드 이름이 고유해야 합니다.

메모리 선언 내에서는 모든 포트 이름이 고유해야 합니다.

이름에 대한 모든 수정은 네임스페이스 내에서 이름의 고유성을 유지해야 합니다.

# Annotations

어노테이션은 임의의 메타데이터를 인코딩하여 FIRRTL 회로에서 0개 이상의 타깃([@sec:targets])과 연결합니다.

어노테이션은 사전으로 표시되며, 어떤 어노테이션인지 설명하는 "클래스" 필드와 어노테이션이 연결된 IR 개체를 나타내는 "대상" 필드가 있습니다. 어노테이션에는 임의의 추가 필드가 첨부될 수 있습니다. 일부 어노테이션 클래스는 다른 어노테이션을 확장하며, 이는 효과적으로 하위 클래스 어노테이션이 상위 어노테이션의 효과를 암시한다는 것을 의미합니다.

어노테이션은 JSON으로 직렬화할 수 있습니다.

아래는 일부 모듈 `foo`{.firrtl}을 표시하는 데 사용되는 어노테이션의 예입니다:

```json
{
  "class":"myannotationpackage.FooAnnotation",
  "target":"~MyCircuit|MyModule>foo"
}
```

## Targets

회로는 접힌 표현으로 설명, 저장 및 최적화됩니다. 예를 들어, 모듈의 인스턴스가 여러 개 있을 수 있으며, 이 인스턴스는 결국 다이에서 해당 모듈의 여러 물리적 복사본이 될 수 있습니다.

타깃은 FIRRTL 회로에서 모듈의 특정 인스턴스에서 특정 하드웨어를 식별하는 메커니즘입니다.  타깃은 회로, 루트 모듈, 선택적 인스턴스 계층 구조 및 선택적 레퍼런스로 구성됩니다. 타겟은 회로, 모듈, 인스턴스, 레지스터, 와이어 또는 노드와 같은 이름을 가진 하드웨어만 식별할 수 있습니다. 참조는 집계에서 특정 필드 또는 하위 인덱스를 추가로 참조할 수 있습니다. 인스턴스 계층 구조가 없는 대상은 로컬입니다. 인스턴스 계층 구조가 있는 대상은 비로컬입니다.

대상은 다음과 같은 형식의 약식 구문을 사용합니다:

```ebnf
target = “~” , circuit ,
         [ “|” , module , { “/” (instance) “:” (module) } , [ “>” , ref ] ]
```

reference는 모듈 내부의 이름과 (번들의) 하위 필드 또는 (벡터의) 하위 인덱스를 인코딩하는 하나 이상의 적격 토큰입니다:

```ebnf
ref = name , { ( "[" , index , "]" ) | ( "." , field ) }
```

타깃은 접힌 상태, 펼쳐진 상태 또는 부분적으로 접힌 상태의 특정 모듈을 참조할 수 있을 정도로 구체적입니다.

이러한 타깃이 어떻게 보이는지 몇 가지 예를 보여드리기 위해 다음 예제 회로를 살펴보겠습니다. 이것은 모듈 `Baz`의 인스턴스 4개, 모듈 `Bar`의 인스턴스 2개, 모듈 `Foo`의 인스턴스 1개로 구성됩니다:

```firrtl
circuit Foo:
  module Foo:
    inst a of Bar
    inst b of Bar
  module Bar:
    inst c of Baz
    inst d of Baz
  module Baz:
    skip
```

이 회로는 _접힌 상태(folded)_, 완전히 _접히지 않은 상태(unfolded)_ 또는 _부분적으로 접힌 상태(partially folded)_로 표현할 수 있습니다.  그림 [@fig:foo-folded]는 접힌 상태의 표현을 보여줍니다.  그림 [@fig:foo-unfolded]는 각 인스턴스가 자체 모듈로 분리된 완전히 펼쳐진 표현을 보여줍니다.

![A folded representation of circuit Foo](build/img/firrtl-folded-module.eps){#fig:foo-folded width=15%}

![A completely unfolded representation of circuit Foo](build/img/firrtl-unfolded-module.eps){#fig:foo-unfolded}

타깃(또는 여러 타깃)을 사용하면 특정 모듈, 인스턴스 또는 인스턴스 조합을 표현할 수 있습니다. 몇 가지 예는 다음과 같습니다:

Target                   Description
-----------------------  -------------
`~Foo`                   refers to the whole circuit
`~Foo|Foo`               refers to the top module
`~Foo|Bar`               refers to module `Bar` (or both instances of module `Bar`)
`~Foo|Foo/a:Bar`         refers just to one instance of module `Bar`
`~Foo|Foo/b:Bar/c:Baz`   refers to one instance of module `Baz`
`~Foo|Bar/d:Baz`         refers to two instances of module `Baz`

대상에 인스턴스 경로가 포함되어 있지 않으면 _로컬_ 대상입니다.  로컬 타깃은 모듈의 모든 인스턴스를 가리킵니다.  타깃에 인스턴스 경로가 포함되어 있으면 _비로컬_ 타깃입니다.  비로컬 타깃은 모듈의 모든 인스턴스를 가리키지 않을 수도 있습니다.  또한 비로컬 타깃은 동등한 로컬 타깃 표현을 가질 수 있습니다.

## Annotation Storage

어노테이션은 사전 배열 형식을 사용하여 하나 이상의 JSON 파일에 저장할 수 있습니다.  다음은 두 개의 어노테이션이 포함된 유효한 어노테이션 파일을 보여줍니다:

``` json
[
  {
    "class":"hello",
    "target":"~Foo|Bar"
  },
  {
    "class":"world",
    "target":"~Foo|Baz"
  }
]
```

어노테이션을 `%[ ... ]`로 어노테이션 JSON을 감싸서 FIRRTL 회로와 함께 인라인으로 저장할 수도 있습니다.  다음은 인라인으로 저장된 위의 어노테이션 파일을 보여줍니다:

``` firrtl
circuit Foo: %[[
  {
    "class":"hello",
    "target":"~Foo|Bar"
  },
  {
    "class":"world",
    "target":"~Foo|Baz"
  }
]]
  module : Foo
  ; ...
```

모든 합법적인 JSON이 허용되며, 이는 위의 JSON이 모두 한 줄에 "최소화"되어 저장될 수 있음을 의미합니다.

# Semantics of Values

FIRRTL은 2상태 부울 논리에 대해 정의됩니다.  다중 상태 로직이 있는 언어(예: Verilog 또는 VHDL)에서 생성된 회로의 동작은 2상태가 아닌 값이 있을 경우 정의되지 않습니다.  FIRRTL 컴파일러는 회로의 2상태 동작만 존중하면 됩니다.  이는 관찰 가능한 동작의 범위에 대한 제한입니다(즉, ["as-if"](https://en.wikipedia.org/wiki/As-if_rule) 규칙의 완화).

## Indeterminate Values

불확정 값은 알 수 없거나 지정되지 않은 값을 나타냅니다. 불확정 값은 일반적으로 아래에 명시된 제약 조건과 함께 구현에 정의됩니다.  불확정 값은 관찰 가능한 모든 동작이 불확정 값이 항상 특정 값을 취하는 것처럼 보일 경우 구현의 재량에 따라 특정 값(반드시 리터럴일 필요는 없음)으로 간주할 수 있습니다.

이를 통해 다음과 같은 변환이 가능한데, `a`{.firrtl}가 불확정 값을 가질 때 구현은 `v`{.firrtl}의 값을 일관되게 부여하도록 선택합니다.  다른 법적 매핑을 사용하면 구현에서 `42`{.firrtl} 값을 지정할 수 있습니다.  두 경우 모두 구현이 선택한 값에 매핑되지 않은 불확정 값이 있는 경우 `a`{.firrtl}의 가시성이 없습니다.

``` firrtl
module IValue :
  output o : UInt<8>
  input c : UInt<1>
  input v : UInt<8>

  wire a : UInt<8>
  a is invalid
  when c :
    a <= v
  o <= a
```
is transformed to:
``` firrtl
module IValue :
  output o : UInt<8>
  input c : UInt<1>

  o <= v
```

produce도 똑같이 정확합니다:

``` firrtl
module IValue :
  output o : UInt<8>
  input c : UInt<1>

  wire a : UInt<8>
  when c :
    a <= v
  else :
    a <= UInt<3>("h42")
  o <= a
```

불확정 값을 유발하는 구조체의 동작은 다음과 같은 제약 조건으로 정의되어 구현됩니다.

- 레지스터 초기화는 모든 레지스터에 대해 일관된 방식으로 수행됩니다.  일부 레지스터를 무작위로 초기화하는(또는 0으로 채우는 등) 코드가 생성되는 경우 모든 레지스터에 대해 생성되어야 합니다.
- 값이 불확실한 표현식의 고유 인스턴스에 대한 모든 관찰은 런타임에 동일한 값을 표시해야 합니다.  값을 여러 번 읽으면 동일한 런타임 값을 보게 됩니다.
- 상태 저장 요소에 캡처된 불확정 값은 시간에 따라 변하지 않습니다. 불확정 값을 보유하는 레지스터와 같은 시간 인식 구조체는 정상적인 방식으로 값을 변경하지 않는 한 동일한 런타임 값을 반환합니다.  예를 들어 초기화되지 않은 레지스터는 쓰기(또는 리셋)될 때까지 여러 클록 사이클에 걸쳐 동일한 값을 반환합니다.
- 중간값을 생성하는 표현식에 대해 런타임에 생성되는 값은 표현식 입력의 함수일 뿐입니다.  예를 들어, 범위를 벗어난 벡터 액세스는 주어진 범위를 벗어난 인덱스와 벡터 내용에 대해 동일한 값을 생성해야 합니다.
- 값이 불확실한 두 구조체는 값의 동일성에 아무런 제약이 없습니다.  예를 들어, 초기화되지 않은 두 레지스터(따라서 불확정 값을 포함)는 비교 시 동일할 필요가 없습니다.

# Details about Syntax

FIRRTL의 구문은 사람이 읽을 수 있지만 알고리즘적으로 쉽게 구문 분석할 수 있도록 설계되었습니다.

식별자에는 대문자와 소문자, 숫자, `_`{.firrtl} 등의 문자를 사용할 수 있습니다. 식별자는 숫자로 시작할 수 없습니다.

FIRRTL의 정수 리터럴은 다음 중 하나로 시작하며, 여기서 '\#'은 0에서 9 사이의 숫자를 나타냅니다.

- 'h': 16진수를 나타내는 경우, 그 뒤에 선택적 기호를 붙입니다. 나머지 리터럴은 숫자 또는 'A'와 'F' 사이의 문자로 구성되어야 합니다.

- 'o': 8진수 뒤에 선택적 기호를 붙입니다.  나머지 리터럴은 0에서 7 사이의 숫자로 구성되어야 합니다.

- 'b' : 2진수 뒤에 선택적 기호를 붙입니다.  나머지 리터럴은 0 또는 1의 숫자로 구성되어야 합니다.

- '-\#' : 음의 십진수를 나타냅니다. 나머지 리터럴은 0에서 9 사이의 숫자로 구성되어야 합니다.

- '\#' : 양수 10진수를 나타냅니다. 나머지 리터럴은 0에서 9 사이의 숫자로 구성되어야 합니다.

주석은 세미콜론으로 시작하여 줄 끝까지 이어집니다.  쉼표는 공백으로 취급되며 사용자가 원하는 경우 명확성을 위해 쉼표를 사용할 수 있습니다.

FIRRTL에서는 들여쓰기가 중요합니다. 들여쓰기는 공백으로만 구성되어야 하며 탭은 잘못된 문자입니다. FIRRTL IR 문 앞에 나타나는 공백의 수는 * 들여쓰기 수준*을 설정하는 데 사용됩니다. 들여쓰기 수준이 동일한 문은 동일한 컨텍스트를 갖습니다. 회로`{.firrtl} 선언의 들여쓰기 수준은 0이어야 합니다.

특정 구조체(`circuit`{.firrtl}, `module`{.firrtl}, `when`{.firrtl} 및 `else`{.firrtl})는 새 하위 컨텍스트를 생성합니다. 하위 컨텍스트의 첫 번째 줄에 사용된 들여쓰기는 들여쓰기 수준을 설정합니다. 하위 컨텍스트의 들여쓰기 수준은 상위 컨텍스트보다 한 단계 높습니다. 하위 컨텍스트의 모든 문은 같은 수의 공백으로 들여쓰기해야 합니다. 하위 컨텍스트를 종료하려면 한 줄이 상위 컨텍스트의 들여쓰기 수준으로 돌아가야 합니다.

조건문(`when`{.firrtl} 및 `else`{.firrtl})은 중첩될 수 있으므로, 각 조건문의 선행 공백 수는 부모보다 커야 하고 모든 직접 자식 문(더 깊은 조건문의 자식이 아닌 문)간에 일관성이 있어야 하는 들여쓰기 수준의 계층 구조를 만들 수 있습니다.

구체적인 가이드로서 이러한 규칙의 몇 가지 결과를 아래에 요약했습니다:

- 회로`{.firrtl} 키워드는 들여쓰기해서는 안 됩니다.

- 모든 `module`{.firrtl} 키워드는 같은 수의 공백으로 들여쓰기해야 합니다.

- 모듈에서 모든 포트 선언과 모든 문(다른 문의 자식이 아닌 문)은 같은 공백 수만큼 들여쓰기해야 합니다.

- 모듈의 들여쓰기 수준을 구성하는 공백의 수는 각 모듈에 따라 다릅니다.

- 조건문의 분기를 구성하는 문은 같은 수의 공백으로 들여쓰기해야 합니다.

- 중첩된 조건문의 문은 자체적으로 더 깊은 들여쓰기 수준을 설정합니다.

- 각 `when`{.firrtl}과 각 `else`{.firrtl} 컨텍스트는 들여쓰기 수준에서 0이 아닌 공백의 수가 다를 수 있습니다.

이러한 점 중 일부를 설명하는 예로 다음은 합법적인 FIRRTL 회로입니다:

``` firrtl
circuit Foo :
    module Foo :
      skip
    module Bar :
     input a: UInt<1>
     output b: UInt<1>
     when a:
         b <= a
     else:
       b <= not(a)
```

모든 회로, 모듈, 포트 및 문 뒤에는 선택적으로 정보 토큰 `@[fileinfo]`가 올 수 있으며, 여기서 fileinfo는 생성된 소스 파일 정보가 포함된 문자열입니다. '`\n`'(새 줄), '`\t`'(탭), '`]`' 및 '`\`' 자체 등 다음 문자는 선행 '`\`'로 이스케이프 처리해야 합니다.

다음 예시는 포함된 정보 토큰을 보여줍니다:

``` firrtl
circuit Top : @[myfile.txt 14:8]
  module Top : @[myfile.txt 15:2]
    output out: UInt @[myfile.txt 16:3]
    input b: UInt<32> @[myfile.txt 17:3]
    input c: UInt<1> @[myfile.txt 18:3]
    input d: UInt<16> @[myfile.txt 19:3]
    wire a: UInt @[myfile.txt 21:8]
    when c : @[myfile.txt 24:8]
      a <= b @[myfile.txt 27:16]
    else :
      a <= d @[myfile.txt 29:17]
    out <= add(a,a) @[myfile.txt 34:4]
```

# FIRRTL 컴파일러 구현 세부 정보

이 섹션에서는 FIRRTL 컴파일러 _implementation_ 개발자에게 필요한 보조 정보를 제공합니다.  FIRRTL 컴파일러는 FIRRTL 텍스트를 다른 표현(예: Verilog, VHDL, 프로그래밍 언어 또는 바이너리 프로그램)으로 변환하는 프로그램입니다.

## Aggregate Type Lowering (Lower Types)

FIRRTL 컴파일러는 집계 유형을 접지 유형으로 변환하는 "하위 유형" 패스를 제공해야 합니다.

FIRRTL 컴파일러는 이러한 패스를 Verilog/VHDL 표현에서 모든 "공용" 모듈의 포트에 적용해야 합니다.  Public modules are defined as (1) the top-level module and (2) any external modules.

FIRRTL 컴파일러는 이러한 패스를 FIRRTL 회로의 다른 유형에 적용할 수 있습니다.

하위 유형 알고리즘은 다음과 같이 작동합니다:

1. 접지 유형 이름은 수정되지 않습니다.

2. 벡터 유형은 벡터의 i^번째^ 요소에 접미사 `_<i>`를 추가하여 접지 유형으로 변환됩니다.  (`<` and `>` are not included in the suffix.)

3. 번들 타입은 `name`이라는 필드에 접미사 `_<name>`을 추가하여 접지 타입으로 변환합니다.  (`<` and `>` are not included in the suffix.)

하위 유형에 의해 생성된 새 이름은 현재 네임스페이스와 관련하여 고유해야 합니다([@sec:namespaces] 참조).

예를 들어, 다음 와이어를 생각해보자:

``` firrtl
wire a : { b: UInt<1>, c: UInt<2> }[2]
```

이 와이어에 적용된 Lower Types 패스의 결과는 다음과 같습니다:

``` firrtl
wire a_0_b : UInt<1>
wire a_0_c : UInt<2>
wire a_1_b : UInt<1>
wire a_1_c : UInt<2>
```

\clearpage

# FIRRTL 언어 정의

``` ebnf
(* Whitespace definitions *)
indent = " " , { " " } ;
dedent = ? remove one level of indentation ? ;
newline = ? a newline character ? ;

(* Integer literal definitions  *)
digit_bin = "0" | "1" ;
digit_oct = digit_bin | "2" | "3" | "4" | "5" | "6" | "7" ;
digit_dec = digit_oct | "8" | "9" ;
digit_hex = digit_dec
          | "A" | "B" | "C" | "D" | "E" | "F"
          | "a" | "b" | "c" | "d" | "e" | "f" ;

(* An integer *)
int = '"' , "b" , [ "-" ] , { digit_bin } , '"'
    | '"' , "o" , [ "-" ] , { digit_oct } , '"'
    | '"' , "h" , [ "-" ] , { digit_hex } , '"'
    |             [ "-" ] , { digit_bin } ;

(* Identifiers define legal FIRRTL or Verilog names *)
letter = "A" | "B" | "C" | "D" | "E" | "F" | "G"
       | "H" | "I" | "J" | "K" | "L" | "M" | "N"
       | "O" | "P" | "Q" | "R" | "S" | "T" | "U"
       | "V" | "W" | "X" | "Y" | "Z"
       | "a" | "b" | "c" | "d" | "e" | "f" | "g"
       | "h" | "i" | "j" | "k" | "l" | "m" | "n"
       | "o" | "p" | "q" | "r" | "s" | "t" | "u"
       | "v" | "w" | "x" | "y" | "z" ;
id = ( "_" | letter ) , { "_" | letter | digit_dec } ;

(* Fileinfo communicates Chisel source file and line/column info *)
linecol = digit_dec , { digit_dec } , ":" , digit_dec , { digit_dec } ;
info = "@" , "[" , { string , " " , linecol } , "]" ;

(* Type definitions *)
width = "<" , int , ">" ;
type_ground = "Clock" | "Reset" | "AsyncReset"
            | ( "UInt" | "SInt" | "Analog" ) , [ width ] ;
type_aggregate = "{" , field , { field } , "}"
               | type , "[" , int , "]" ;
field = [ "flip" ] , id , ":" , type ;
type = type_ground | type_aggregate ;

(* Primitive operations *)
primop_2expr_keyword =
    "add"  | "sub" | "mul" | "div" | "mod"
  | "lt"   | "leq" | "gt"  | "geq" | "eq" | "neq"
  | "dshl" | "dshr"
  | "and"  | "or"  | "xor" | "cat" ;
primop_2expr =
    primop_2expr_keyword , "(" , expr , "," , expr ")" ;
primop_1expr_keyword =
    "asUInt" | "asSInt" | "asClock" | "cvt"
  | "neg"    | "not"
  | "andr"   | "orr"    | "xorr" ;
primop_1expr =
    primop_1expr_keyword , "(" , expr , ")" ;
primop_1expr1int_keyword =
    "pad" | "shl" | "shr" | "head" | "tail" ;
primop_1expr1int =
    primop_1exrp1int_keyword , "(", expr , "," , int , ")" ;
primop_1expr2int_keyword =
    "bits" ;
primop_1expr2int =
    primop_1expr2int_keyword , "(" , expr , "," , int , "," , int , ")" ;
primop = primop_2expr | primop_1expr | primop_1expr1int | primop_1expr2int ;

(* Expression definitions *)
expr =
    ( "UInt" | "SInt" ) , [ width ] , "(" , ( int ) , ")"
  | reference
  | "mux" , "(" , expr , "," , expr , "," , expr , ")"
  | primop ;
reference = id
          | reference , "." , id
          | reference , "[" , int , "]"
          | reference , "[" , expr , "]" ;

(* Memory *)
ruw = ( "old" | "new" | "undefined" ) ;
memory = "mem" , id , ":" , [ info ] , newline , indent ,
           "data-type" , "=>" , type , newline ,
           "depth" , "=>" , int , newline ,
           "read-latency" , "=>" , int , newline ,
           "write-latency" , "=>" , int , newline ,
           "read-under-write" , "=>" , ruw , newline ,
           { "reader" , "=>" , id , newline } ,
           { "writer" , "=>" , id , newline } ,
           { "readwriter" , "=>" , id , newline } ,
         dedent ;

(* Statements *)
statement = "wire" , id , ":" , type , [ info ]
          | "reg" , id , ":" , type , expr ,
            [ "(with: {reset => (" , expr , "," , expr ")})" ] , [ info ]
          | memory
          | "inst" , id , "of" , id , [ info ]
          | "node" , id , "=" , expr , [ info ]
          | reference , "<=" , expr , [ info ]
          | reference , "is invalid" , [ info ]
          | "attach(" , { reference } , ")" , [ info ]
          | "when" , expr , ":" [ info ] , newline , indent ,
              { statement } ,
            dedent , [ "else" , ":" , indent , { statement } , dedent ]
          | "stop(" , expr , "," , expr , "," , int , ")" , [ info ]
          | "printf(" , expr , "," , expr , "," , string ,
            { expr } , ")" , [ ":" , id ] , [ info ]
          | "skip" , [ info ] ;

(* Module definitions *)
port = ( "input" | "output" ) , id , ":": , type , [ info ] ;
module = "module" , id , ":" , [ info ] , newline , indent ,
           { port , newline } ,
           { statement , newline } ,
         dedent ;
extmodule = "extmodule" , id , ":" , [ info ] , newline , indent ,
              { port , newline } ,
              [ "defname" , "=" , id , newline ] ,
              { "parameter" , "=" , ( string | int ) , newline } ,
            dedent ;

(* In-line Annotations *)
annotations = "%" , "[" , json_array , "]" ;

(* Version definition *)
sem_ver = { digit_dec } , "."  , { digit_dec } , "." , { digit_dec }
version = "FIRRTL" , "version" , sem_ver ;

(* Circuit definition *)
circuit =
  version , newline ,
  "circuit" , id , ":" , [ annotations ] , [ info ] , newline , indent ,
    { module | extmodule } ,
  dedent ;
```


# 이 문서의 버전 관리 체계

이것은 버전 1.0.0 이상에 적용되는 버전 관리 체계입니다.

이 버전 관리 체계는 [시맨틱 버전 관리 2.0.0](https://semver.org/#semantic-versioning-200)을 준수합니다.

구체적으로

패치 숫자는 문법 편집, 추가 예제 및 설명과 같은 비기능적 변경 사항만 포함하는 릴리스 시 범프됩니다.

MINOR 숫자는 사양에 기능이 추가된 경우 범프됩니다.

메이저 숫자는 사양에서 기능이 제거되거나 해석이 변경되거나 새로운 필수 기능이 사양에 추가되는 등 이전 버전과 호환되지 않는 변경 사항의 경우 범프됩니다.

즉, `x.y.z`와 호환되던 모든 `.fir` 파일은 `x.Y.Z`와 호환되며, 여기서 `Y >= y`, `z` 및 `Z`는 임의의 숫자가 될 수 있습니다.
