# 2.2

## 런타임 데이터 영역(Java Runtime Data Area)

JVM은 자바 프로그램을 실행하는 동안 **필요한 메모리를 몇 개의 데이터 영역**으로 나누는데, 각각 영역별로 생명주기, 목적에서 차이점이 있다.

![image](https://github.com/user-attachments/assets/b2aef4f0-958a-437d-933b-9f5c2476d66f)

위의 사진은 JVM의 구성 요소를 표현한 것인데, 정리하자면

>- JVM = 클래스 로더 + 런타임 데이터 영역 + 실행 엔진
>- 런타임 데이터 영역 = 메소드 영역 + 힙 영역 + 스택 영역 + 프로그램 카운터 레지스터 + 네이티브 메소드 스택

![image](https://github.com/user-attachments/assets/d93fe54e-9689-4e4c-9cfb-a7bda6242b87)

런타임 데이터 영역은 프로그램 생명주기에서는 클래스 정보와 실행에 필요한 데이터를 저장하고, 객체 생명주기에서는 객체의 생성, 사용, 소멸에 필요한 메모리 영역(힙, 스택 등)을 관리하게 된다. 간단히 정리하자면 **자바 애플리케이션이 실행될 떄 사용되는 데이터들이 저장되는 메모리 공간**이라고 볼 수 있다. 이 중에서 **스레드**가 공유하는 영역은 힙 영역과 메소드 영역이다.

### 1. 프로그램 카운터 레지스터

멀티스레딩에 있어 스레드가 각자의 메소드를 실행하게 될 텐데, 각각의 스레드에게 동시 실행 환경이 보장되어야 한다. 이때, 단위 코어가 한 스레드의 명령어만 실행할 수 있고 교대로 스레드를 번갈아가며 사용되는 방식이어서 **스레드 전환 후, 이전에 실행하다 멈춘 지점을 정확하게 복원할 필요성**이 있다. 이 복원을 위해 스레드 각각에 **프로그램 카운터**가 필요하다.

조금 더 정확히 말하자면 최근에 자바 메소드를 실행 중일 때, 실행 중인 바이트코드 명령어의 주소가 프로그램 카운터에 기록되는 것이며 이것이 기록되는 공간이 **프로그램 카운터 레지스터**다. 만약 실행했던 메소드가 네이티브(C나 C++로 실행)하다면, 해당 명령어의 위치를 알 수 없기 때문에 프로그램 카운터 레지스터에 `undefined` 값을 기록하게 된다.

### 2. 네이티브 메소드 스택

프로그램 카운터 레지스터에서 네이티브 메소드가 실행될 때는 기록이 되지 않으므로 이를 위한 네이티브 메소드 스택이 필요하다. 후술할 스택 영역과 매우 비슷한 역할을 하며 각 스레드 별로 생명주기를 가지게 된다.

### 3. 스택 영역

각 스레드별로 **메소드 호출 시 지역변수, 파라미터, 호출 내역 등, 스레드의 메소드 호출 정보를 저장**하는 스택 영역을 가진다. 메소드 호출 시에 생성되었다가 호출이 종료되면 소멸된다. 대표적이고 중요한 특징들을 정리하자면 아래와 같다.

- 스레드별 생명주기를 가지며 스레드 간에는 배타적이다.
- 그렇기 때문에 동기화 이슈가 발생하지 않는다(지역변수를 생각해 볼 것).
- 스택은 애플리케이션 실행 시에 **스택 프레임**이라는 데이터 단위에 메소드 호출 데이터들을 저장한다.

JVM은 스택 프레임을 스택 영역에 푸시했다가 제거하는 작업만을 맡는다. 자바 애플리케이션이 동작할 때 `kill -3 <pid>` 명령어를 사용해서 얻는 스레드 덤프에서 확인할 수 있는 스택 트레이스가 바로 스택 프레임의 핵심 정보들을 담고 있는 것이다.

<img width="867" alt="스레드 덤프" src="https://github.com/user-attachments/assets/e6e0039f-9560-4711-89f0-0edb457b5bc2">

스택 프레임의 크기는 가변이 아니다. 메소드 내의 변수, 연산, 반환 값 타입 등은 이미 자바 소스 코드에서 결정되기 때문에 컴파일 시점에 스택 프레임의 크기가 미리 결정될 수 있다. JVM은 스택 프레임을 JVM 스택에 푸시해서 메소드를 실행하게 된다. 여기서 왜 이름이 스택이냐면 실제로 **스택 자료구조** 형태를 따르기 때문이다. 메소드 내부에 메소드가 있을 때, 어차피 먼저 시작(호출)되는 메소드는 외부의 메소드이고 먼저 종료되는 메소드가 내부 메소드일 테니까.

```java
void method1() { // 먼저 호출
    method2(); // 나중에 호출되고 먼저 종료
} // 나중에 종료
```

스택 프레임은 **Local Variables**, **Operand Stack**, **Frame Data**로 구성된다.

#### (1) 지역변수 영역(Local Variable Section)

메소드의 파라미터 변수와 지역변수들이 저장되는 곳이다. 이 지역변수 영역은 배열로 구성되어 있다. 아래의 코드를 디버깅해보면 다음과 같이 나온다.

```java
public class Test1 {
    public int method(int a, char b, long c, float d, boolean e, Object f, String g, Integer h) {
        // 스택 트레이스를 출력하여 호출 스택을 확인
        Thread.dumpStack();
        return 0;
    }

    public static void main(String[] args) {
        Test1 test = new Test1();
        test.method(1, 'b', 123L, 3.14f, true, new Object(),"ABCDE", 12345);
    }
}
```

<img width="1071" alt="스크린샷 2024-12-09 오후 7 20 15" src="https://github.com/user-attachments/assets/45531863-789a-4af9-8ecd-10ffb73421f7">

자바 문법 중, 원시 타입과 참조 타입을 알고 있으면 좋다. 파라미터 `a`부터 `e`까지는 원시 타입인데 보면 정해진 바이트 혹은 비트(예를 들어 int는 4바이트, char은 2바이트, boolean은 1비트 등)로 저장될 수 있다. 의아한 점은 참조 타입이다. `f`부터 `h`까지는 참조 타입인데, 그 크기가 정해져있지 않다. 그럼에도 위에서 말했듯 스택 프레임의 크기가 컴파일 시점에서 이미 정할 수 있다고 했는데 어떻게 가능한 것일까.

그 이유는 **참조**라는 키워드에 정답이 있다. `int`와 `Integer` 파라미터를 비교하면 알겠지만, `Integer` 타입은 객체의 **주소**를 통해 넘어가서 실제 값을 확인할 뿐 파라미터 입장에서는 단순 참조값만을 알고 있을 뿐이다. 즉 스택 프레임으 구성요소로써 저장되는 것은 **객체의 크기가 아닌, 참조 주소의 크기이며 이 참조 주소의 크기는 미리 정해져있다**(64비트 JVM에서는 8바이트, 32비트 JVM에서는 4바이트)

실제 객체의 크기는 후술할 힙 메모리에 저장되며 여담이지만, 위의 이유로 성능 상에서는 원시 타입이 더 좋은 편이긴 하다. 스택에서 힙으로 넘어가는 것 또한 리소스 소모이기 때문이다.

>- `boolean`을 제외하고 모든 원시 타입은 JVM이 직접적으로 지원하는 형태이다.
>- 그렇지만 `byte`, `short`, `char`는 지역변수 영역이나 연산 스택에서는 `int` 형으로 저장되고 힙 등 다른 곳에서는 원래의 형으로 원복하여 저장된다.
>- `boolean`은 원복하지 않는다. JVM 내에서는 그저 숫자에 불과하다(코드 레벨과는 다름).

#### (2) 연산 스택(Operand Stack)

JVM의 **작업 공간**을 맡는다. 프로그램이 실행되면서 수행하는 연산의 중간 결과값들을 일시적으로 저장하고 조작하는 공간이 바로 연산 스택이다. 명령어 실행과 관련된 중간 결과를 처리하는 데에 사용된다. 연산 스택 역시 지역변수 영역처럼 배열로 구성되어 있으나, 연산 스택은 자료구조가 **스택**이란 점에서 인덱스를 사용하지 않는다.

아래 코드를 연산해서 컴파일하여 바이트코드를 추출해본다.

```java
public class Test2 {
    public void operandStack(){
        int a, b, c;
        a = 5;
        b = 6;
        c = a + b;
    }
}
```

<img width="578" alt="스크린샷 2024-12-09 오후 8 31 00" src="https://github.com/user-attachments/assets/ffb8fb85-6277-4866-a4a5-0738ee1e30ba">

`operandStack()` 메소드의 바이트코드를 해석해보면 아래와 같다. 해당 내용은 지역변수 영역과 연산 스택 간의 상호작용을 표현하고 있다.

```bash
  public void operandStack();
    Code:
       0: iconst_5        # 상수 5를 연산 스택에 푸시
       1: istore_1        # 연산 스택값(5)를 지역변수 1에 저장
       2: bipush        6 # 6(-128에서 127까지의 바이트 값 범위내)을 스택에 푸시
       4: istore_2        # 스택에서 6을 꺼내 지역변수 2에 저장
       5: iload_1         # 지역변수 1에 저장된 5가 연산 스택에 푸시
       6: iload_2         # 지역변수 2에 저장된 6이 연산 스택에 푸시
       7: iadd            # 연산 스택에 있는 값들(5와 6)을 꺼내서 합산
       8: istore_3        # 합산한 값(11)을 지역변수 3에 저장
       9: return          # 메소드 종료
```
![image](https://github.com/user-attachments/assets/c324d596-409f-4bc3-bb5b-8e6a1ad9d970)

#### (3) 스택 프레임 정보(Frame Data)

스택 프레임 데이터라는 별도의 영역이 있는 건 아니고, 지역변수 영역과 연산 스택 이외의 스택 구성 요소를 뭉뚱그려서 프레임 데이터라고 부를 수 있다. 여기에는 상수 풀 정보와 메소드의 정상 종료 시의 정보와 비정상 종료 시의 예외 정보 등을 저장한다. 아까 위에서 봤던 `Test2` 클래스의 `operandStack()` 메소드에 `try - catch` 구문을 추가해서 다시 컴파일 후, 바이트코드를 추출해보자.

```java
public class Test2 {
    public void operandStack(){
        int a, b, c;
        a = 5;
        b = 6;

        try {
            c = a + b;
        } catch (IllegalArgumentException e) {
            c = 0;
        }
    }
}
```

<img width="621" alt="스크린샷 2024-12-09 오후 9 08 40" src="https://github.com/user-attachments/assets/0064e460-8d1e-45db-b8dd-50770998366a">

추가된 정보들을 확인하면 다음과 같다.

>- `goto 16`: 예외가 없으면 이 명령어로 프로그램 흐름이 16번 줄로 가서 메소드 종료
>- `astore 4`: 예외가 발생하면, 예외 객체를 지역변수 4에 저장하고 iconst_0로 값을 0으로 설정
>- 예외 테이블(Exception Table) 추가
>  - `from 5 to 9`(각 숫자는 바이트코드 엔트리 넘버)까지의 범위에서 `IllegalArgumentException`이 발생할 수 있다.
>  - 예외가 발생하면 바이트코드 엔트리 넘버 12로 넘어간다.

이 예외 테이블이 기타 프레임 데이터의 일종이라고 볼 수 있다.

### 4. 메소드 영역
*스레드 공유*

여기서부터는 스레드가 공유하는 영역이다. 참고로 밑의 내용은 자바 8 이하에 해당하는 내용이며 이후는 후술한다. 메소드 영역은 JVM과 생명주기를 같이하며 **클래스 로딩과 관련된 정보**를 다루고 런타임 시에 클래스 지원 구조를 결정하게 된다. 여기에는 클래스 로더에 의해 로드된 클래스 관련 정보들을 저장하는데 구체적인 내용들은 아래와 같다.

>- 상수 풀(Constant Pool)
>- 클래스 및 인터페이스의 필드
>- 메소드
>- 생성자
>- 정적 변수
>- 정적 메소드

JVM 벤더에 따라 다르지만, 보통 힙에 주로 관여하는 가비지 컬렉터(GC)가 메소드 영역에도 관여할 수 있다. 메소드 영역에 저장되어 있는 클래스의 메타데이터가 힙에 있는 객체 인스턴스와 연결되어 있지 않다면, GC가 힙에서 객체의 메모리를 회수하면서 메소드 영역에서도 클래스의 메타데이터가 삭제될 수 있다. 다만 말했듯이 JVM 벤더에 따라 다르다.

전술했듯 위의 내용은 자바 8 이하에 해당하는 내용이었고, 현재 위의 기능들 대부분은 **메타스페이스**로 이전됐다. 현재 메소드 영역은 클래스 로딩과 관련돼서 상당히 제한적인 기능만으로 동작하고 있다. 메타스페이스는 네이티브 메모리라는 OS에 의해 관리되는 영역에서 관리하나 기존 메소드 영역에서 관리하던 **문자열 풀, 정적 변수값**은 힙에서 관리하게 됐다. 이것의 의의는 GC의 관리 대상에 정적 변수값 역시 포함됐다는 점이다. 메타스페이스 내용은 힙 설명에서 추가 후술한다.

### 5. 힙 영역
*스레드 공유*

자바의 보편적인 이슈 중 하나는 메모리 문제다. 그래서 메모리 해제 역시 중요 개념이며 이는 전술한 GC가 맡게 된다. 이 GC가 직접적으로 관여하는 부분이 바로 힙이다. 힙의 특징을 정리하자면 아래와 같다.

- 오로지 **객체**와 **배열**만 저장된다.
- 동일한 객체 인스턴스는 스레드 간에 공유되기 때문에 동기화 이슈가 발생할 수 있다.
- JVM이 실행 중인 시스템 메모리 중 가장 큰 영역을 차지하게 된다.
- 힙은 `Young` 영역과 `Old` 영역으로 나뉘며 이는 GC에서 봤던 내용과 동일하다.

다만 **힙의 구조는 정형적이지 않다.** 이는 `-Xmx`, `-Xms` 명령어를 통해 알 수 있듯이 힙은 확장성 있게 구현되어 있다. 또한 GC 알고리즘에 따라 구조가 달라질 수도 있다. 아래처럼 `-Xms` 명령어로 초기 메모리 확보량을 지정하고 `-Xmx` 명령어로 최대 메모리 크기를 지정할 수 있다.

<img width="417" alt="스크린샷 2024-12-09 오후 10 43 55" src="https://github.com/user-attachments/assets/90c77398-e87a-4e4b-a10a-4a3beb8c4b65">

앞서 메타스페이스에 대해 간략히 봤는데 JDK 7까지는 힙에 `PermGen` 이라는 영구 세대 영역이 존재했다. 여기서 클래스 메타데이터, 문자열 상수와 정적 변수를 관리했으나 이것과 관련해 `OutOfMemoryError`가 자주 발생했고 JDK 8 이후에 등장하는 메타스페이스에 클래스 메타데이터와 런타임 상수 풀을 관리시키고 힙에는 정적 변수와 문자열 풀을 관리시켰다.

인텔리제이에서 힙 관련 설정 정보들(JVM 및 GC 관련)을 로그로 출력하려면 `Run Edit Configuration`을 통해 아래와 같은 명령어를 추가한다.

<img width="1031" alt="스크린샷 2024-12-09 오후 11 07 36" src="https://github.com/user-attachments/assets/2b93f17d-bb75-4ab7-92a2-8121b3e4588e">
<img width="1396" alt="스크린샷 2024-12-09 오후 11 08 13" src="https://github.com/user-attachments/assets/2d9864d1-e3c8-4f51-b569-0093ab01e41d">

>#### 1. JVM 설정
>- **JVM 버전**: `21.0.2+13-58 (release)`
>- **CPU**: 8개 (총 8개 사용 가능)
>- **메모리**: 16GB (`Memory: 16384M`)
>#### 2. 힙 메모리 설정
>- **힙 최소 크기 (`-Xms`)**: 128MB
>- **힙 최대 크기 (`-Xmx`)**: 512MB
>#### 3. GC 알고리즘
>- **사용 중인 GC**: `G1 Garbage Collector`
>#### 4. Metaspace 관련 설정
>- **CDS Archive Mapped Address**: `0x0000009000000000-0x0000009000ca4000`
>- **Compressed Class Space 주소 범위**: `0x0000009001000000-0x0000009041000000`
>- **Compressed Oops**: 활성화
>#### 5. 기타 설정
>- **Heap Region Size**: 1MB
>- **Parallel Workers**: 8
>- **Concurrent Workers**: 2
>- **Concurrent Refinement Workers**: 8
>- **Large Page Support**: 비활성화
>- **NUMA Support**: 비활성화


---
*참조*<br />
*https://velog.io/@ddangle/Java-%EB%9F%B0%ED%83%80%EC%9E%84-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EC%98%81%EC%97%ADRuntime-Data-Area%EC%97%90-%EB%8C%80%ED%95%B4*<br />
*https://loosie.tistory.com/847#Runtime_Data_Areas*
