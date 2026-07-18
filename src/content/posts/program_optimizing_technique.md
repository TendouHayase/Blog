---
title: 프로그램 최적화 방법 소개
published: 2025-03-16
description: 프로그램을 컴파일러 친화적으로 더 빠르게 동작하도록 최적화하는 방법
tags: ["Computer Science"]
category: Knowledge
draft: false
---
# 프로그래머가 최적화를 알아야 하는 이유

현대의 컴파일러는 과거보다 훨씬 더 빠르게 프로그램을 작동하도록 최적화를 시킵니다.
심지어 초보 프로그래머가 최대한 최적화한 코드보다 컴파일러가 *-O1*정도의 수준으로 최적화한 코드가 더 빠르게 실행될 수도 있습니다.
하지만 컴파일러가 프로그램을 최적화하는데는 한계가 있습니다.
컴파일러가 과도하게 최적화를 하려다가 언어의 표준이 보장하는 동작을 보장하지 못할 경우가 생길 수 있기에 실제로는 코드 논리 구조와 상관없는 최적화지만 최적화하진 못하는 경우가 생길 수 있기 때문입니다.  

# 최적화 정도 측정 단위

코드의 속도를 측정하는 단위는 여러가지가 있겠지만 여기서는 CPE(Cycle Per Element)라는 단위를 사용할 것 입니다.
CPE는 CPU가 동작하는 클럭이 한 원소를 처리하는데 얼마나 걸리는지를 나타낸 지표이다.
예를 들어 루프에서 한 변수가 다음 루프까지 처리되는 사이클이 5번이면 CPE는 5.0인 것이다.

# 최적화 방법

## 루프 비효율성 제거

```cpp
// 데이터 타입을 data_t라고 가정
void func1(data_t* src, data_t* dest)
{
 long i;

 *dest = INITIAL_VALUE; // 초기값 설정

 for (i = 0; i < data_length(src); i++)
 {
  data_t val;
  get_data_from_container(src, i, &val); // 컨테이너로부터 데이터 가져오기
  *dest = *dest + val;
 }
}

void func2(data_t* src, data_t* dest)
{
 long i;
 long length = data_length(src);

 *dest = INITIAL_VALUE; // 초기값 설정

 for (i = 0; i < length; i++)
 {
  data_t val;
  get_data_from_container(src, i, &val); 
  *dest = *dest + val;
 }
}
```

위 코드의 두 함수의 차이는 반복문의 검사를 할 때 쓸 변수를 반복문 내에서 호출하느냐, 아니면 그 전에 초기화하느냐의 차이만 존재합니다.
저와 같은 초보 프로그래머는 두 함수의 속도가 얼마나 차이나느냐고 물을 수 있습니다.
하지만 만약 `data_length`함수의 시간 복잡도가 상수가 아니라면 이야기가 달라집니다.
대표적으로 문자열과 같은 것의 길이 검사는 보통 $O(n)$의 시간 복잡도가 걸립니다.
즉, 그렇다면 위 `func1`과 `func2`의 시간복잡도는 $O(n^2)$과 $O(n)$으로 시간복잡도가 매우 크게 차이나게 됩니다.
이런 경우, 컴파일러가 최적화를 해야하느냐 할 수도 있지만 GCC 같은 경우는 반복문 안에서 src가 변할 수도 있기에 최적화를 수행하지 않습니다.
그렇기에, 이런 점을 프로그래머가 잘 캐치하여 최적화하여야 합니다.

## 프로시저 호출 줄이기

```cpp
void get_data_from_container(data_t* src, int i, data_t* dest)
{
 if (i < 0 || i > src->len()) // data_t가 len이란 메소드를 가지고 있다 가정
  throw std::out_of_range();
 *dest = src[i];
}

void func2(data_t* src, data_t* dest)
{
 long i;
 long length = data_length(src);

 *dest = INITIAL_VALUE; // 초기값 설정

 for (i = 0; i < length; i++)
 {
  data_t val;
  get_data_from_container(src, i, &val); 
  *dest = *dest + val;
 }
}

void func3(data_t* src, data_t* dest)
{
 long i;
 long length = data_length(src);
 
 for (i = 0; i < length; i++)
  *dest = *dest + data[i];
}
```

위 코드에서 두 함수는 함수 호출에 따른 오버헤드도 있겠지만 주요 차이점은 인덱스 검사를 하는가 여부입니다.
이 코드는 최신 프로세서들에서는 분기예측같은 기술을 통해 성능저하가 거의 없지만
다른 코드에서 프로시저를 호출하면 호출할수록 속도가 느려집니다.

## 불필요한 메모리 참조의 제거

```cpp
void func3(data_t* src, data_t* dest)
{
 long i;
 long length = data_length(src);
 
 for (i = 0; i < length; i++)
  *dest = *dest + data[i];
}

void func4(data_t* src, data_t* dest)
{
 long i;
 long length = data_length(src);
 long sum = *dest;
 
 for (i = 0; i < length; i++)
  sum = sum + data[i];

 *dest = sum;
}
```

위 코드에서의 두 함수의 차이는 반복문 안에서 메모리 참조 여부입니다.
메모리 참조는 레지스터를 바로 사용못하고 메모리에 접근을 해야하기에 사이클을 많이 잡아먹는 작업입니다.
그래서 위의 `func3`을 `func4`처럼 바꾸면 기계어로 변환되었을때 `sum`이 레지스터에서 작업되어 사이클을 적게 먹게 됩니다.

## 루프 풀기

루프풀기는 반복문에서 매 반복실행마다 계산되는 원소의 수를 증가시켜 루프의 실행횟수를 줄이는 방법입니다.  
예시로 $2*1$ 루프풀기는 매 반복실행마다 계산되는 원소를 2배로 늘려 루프의 실행횟수를 절반으로 낮추고 루프 인덱스 계산과
조건부 분기같은 연산의 횟수를 줄입니다.  

## 병렬성 높이기

병렬성 높이기의 대표적인 예시로는 1-100까지 모두 더할때 1-50과 51-100까지 더하는 변수를 따로 만들어 CPU 파이프라인을 더 효과적으로 쓸 수 있게합니다.

## 재결합 변환

재결합 변환은 `acc = (acc + data[i]) + data[i+1];`같은 코드를 `acc = acc + (data[i] + data[i+1]);`같은 코드로 변환시켜 데이터 의존성을 줄이는 방법입니다.  

## SIMD 활용

최신 프로세서들은 SIMD(Single Instruction, Multi Data)라고 하는 SSE(Streaming SIMD Extension), AVX(Advanced Vector Extension)같은 연산을 지원합니다. 이 명령어들의 관련된 레지스터는 아키텍처마다 다르지만 32비트 수 8개 또는 64비트 수 4개를 보관할 수 있으며 이 레지스터에 한번의 인스트럭션을 사용해 모두 연산할 수 있습니다. 컴파일러가 AVX를 잘 활용할 수 있게하고 int형(32비트) 사용시 루프풀기의 인자를 8로 하여 코드를 변환하면 매우 빠른 성능을 보이게됩니다.  

## 레지스터 넘기기

레지스터 넘기기는 지금까지와는 다르게 사용되면 성능이 떨어집니다. 이것은 CPU에 있는 레지스터 수를 넘어가는 루프 변수들이 있다면 초과한 변수들을 스택에 할당하고 다음 사이클에 메모리에 접근하여 계산하기 때문입니다.  

## 분기예측

현대 CPU들은 분기예측이라는 기술로 조건문에서 결과를 미리 계산하고 틀릴 경우에만 속도가 느려지는 그런 아키텍처를 가지고 있습니다.
따라서 CPU가 예측하기 쉬운 분기들은 속도를 거의 떨어뜨리지 않기에 코드 논리상 1번의 다른 결과만 나오는 범위 검사같은 코드는 성능을 떨어뜨리지 않습니다.
하지만 분기예측은 규칙적인 패턴에 대해서만 안정적이기에 예측하기 힘든 로직이 되면 좋지 않은 성능을 보이게 됩니다.
이를 최대한 해결하기위해 코드 스타일을 명령형 스타일(프로그램의 상태를 선택적으로 갱신하기 위해 조건문 사용)에서 기능적 스타일(값을 계산하고 이 값들로 프로그램의 상태를 갱신하기 위해 조건부 연산 사용)로 바꾸어 조건부 제어이동대신 조건부 데이터 이동을 사용하도록 할 수 있습니다. 코드예시로는 `if (a[i] > b[i]) {long t = a[i]; a[i] = b[i]; b[i] = t;}` 같은 코드를 `long min = a[i] < b[i] ? a[i] : b[i]; long max = a[i] < b[i] ? b[i] : a[i]; a[i] = min; b[i]= max;`와 같은 코드로 변환할 수 있습니다.

---

> 참조 : Randel E Bryant, David R O'Hallaron의 컴퓨터 시스템
