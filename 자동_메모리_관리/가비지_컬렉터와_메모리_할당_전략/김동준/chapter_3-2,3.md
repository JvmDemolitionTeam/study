가비지 컬렉터가 힙에 저장된 객체를 정소하려면 죽은 객체를 판단해야 한다. 즉, 어떤 식으로도 더는 사용될 수 없는 객체를 찾아야 한다. 개념 위주로 정리하되, 알고리즘과 관련해서는 요건에 해당하는 객체인지를 판별하는 알고리즘들을 이해하고 필요시 커스터마이징하면서 직접 테스트를 통해 이해할 예정.

## 서론

참고로 GC는 자바만의 특징이 아닌, 다른 프로그래밍 언어(자바스크립트, 파이썬 등)도 가지고 있는 특징이다. 또한 후술할 알고리즘들의 베이스 역시 다른 언어들의 GC에서도 똑같이 통용되는 편이다. 그럼에도 불구하고 자바, 코틀린 즉 JVM의 GC에서 도드라지는 특징들은 아래와 같다.

1. 세대별 가비지 컬렉션을 통해 객체 수명에 맞춘 효율적 메모리 관리
2. 다양한 GC 알고리즘 제공(Serial, Parallel, G1, ZGC 등)으로 애플리케이션 요구사항에 맞는 선택 가능
3. Stop-The-World 시간 최소화를 목표로 한 동시성 GC의 발전
4. GC 튜닝 및 핫스팟 최적화를 통해 동적 성능 개선

위의 특징들을 바탕으로 다양한 GC 알고리즘 및 동작 원리를 중점적으로 공부할 예정.

## 알고리즘 개념

### 1. 참조 카운팅 알고리즘

참조 카운팅 알고리즘은 가장 단순하게 구현된 알고리즘이다. 이 알고리즘은 **어떤 다른 객체도 참조하지 않는 객체**를 더 이상 필요없는 객체로 여기고 이를 가비지라 부른다. 참조 개수를 카운팅해서 참조가 하나도 없으면(0이면) 가비지로 판단한다. 즉, 메모리 주소가 참조될 때마다 카운트를 1 증가시키고, 참조를 끊을 땐 1 감소시킴으로써 카운트를 체크했을 때 0이면 메모리를 해제하는 방식이다.

다만 이 알고리즘은 **순환 참조** 문제를 해결하지 못한다. 아래의 코드를 GC 로그가 뜨도록 파라미터를 부여해서 실행해본다.
```java
public class ReferenceCountingGC {
    public Object instance = null;
    private static final int MB = 1_024 * 1_024;
    private byte[] bigSize = new byte[2 * MB];

    public static void main(String[] args) {
        test();
    }

    public static void test() {
        ReferenceCountingGC a = new ReferenceCountingGC();
        ReferenceCountingGC b = new ReferenceCountingGC();

        // 상호 참조
        a.instance = b;
        b.instance = a;

        a = null; // a 변수 더 이상 아무 것도 참조 x
        b = null; // b 변수 더 이상 아무 것도 참조 x

        /**
         * 변수가 가리키는 방향만 달라졌을 뿐, 기존 a, b에 할당됐던 객체는 여전히 남아있음
         */

        // 가비지 컬렉터 수행
        /**
         * 참조 카운팅 알고리즘대로라면 사실 동작할 수가 없음. 여전히 객체 2개는 살아있고 여전히 상호 참조 중이므로 
         */
        System.gc();
    }
}
```
<img width="60%" alt="스크린샷 2024-12-13 오후 9 00 45" src="https://github.com/user-attachments/assets/e42a1877-be1a-43bc-9ad9-5c634e37ed2a" />

보면 확인되지만 애시당초 참조 카운팅 알고리즘을 사용하는 옵션 자체를 제공하지 않는다. 왜냐하면 코드에서도 확인할 수 있듯, 상호 참조가 남아있는 객체가 잔존한 이상 참조 카운팅 알고리즘이 동작할 수 없다.

즉, 참조 카운팅 알고리즘은 GC를 수행하기 위한 알고리즘이 될 수 없다.

### 2. 도달 가능성 분석 알고리즘

통상 객체의 생사 판단에는 **루트 객체로부터 거슬러 올라가 닿을 수 있는가**를 기준으로 판단하게 된다. 여기서 쓰이는 알고리즘이 그래프에서 길 찾기를 할 때 주로 쓰이는 **BFS** 혹은 **DFS**가 주로 쓰인다.

<img width="70%" src="https://github.com/user-attachments/assets/909fd9e1-da9a-4e3b-9b3c-217654402564" />

그 중에서도 주로 많이 쓰이는 건 **DFS**다. 길찾기 기반이며 시간 복잡도가 `O(n)`으로 같지만, 메모리 효율성이 더 뛰어나다. 그 이유는 객체의 참조 구조는 보통 그래프에서도 **유사 트리 구조**를 기반으로 형성되는데, DFS는 너비와 상관없이 루트 노드로부터 추적하는 현재 경로상의 노드만을 기억하면 되는 반면, BFS는 현재 위치한 깊이의 전체 너비의 노드를 기억해야 하기 때문이다. 

또한, 구조 자체가 객체의 참조를 재귀적으로 호출해서 파악하는 방식이므로 재귀를 기반으로 구현한 DFS가 직관적으로 적용되기 쉬운 편이다. 위의 과정이 곧 밑에서 나오는 마크 시리즈의 **마킹 단계**에 적용되는 알고리즘이며 이 과정을 통해 도달 가능성을 분석하고 도달이 불가능한 객체는 곧 GC의 대상이 된다.

#### (1) 마크 스윕(Mark & Sweep)

<img width="70%" alt="스크린샷 2024-12-16 오전 12 02 46" src="https://github.com/user-attachments/assets/eacf23b5-0c38-4d8d-b104-fe22f8750c29" />


마킹 이후, 스윕 단계에서는 단순하게 비활성 객체를 해제하게 된다. 즉 마킹 단계에서 이미 도달 가능성에 따른 생사여부가 결정됐기 때문에 죽었다고 판정된 객체를 단순히 제거되기 때문에 특별한 기법이 요구되진 않는다. 그렇기 때문에 후술할 다른 알고리즘들에 비해 구현이 상당히 간단하다. 

이 과정은 Young Gen, Old Gen 모두에서 이뤄지며 각자의 기법은 다르지만 공통적으로 죽은 판정을 받은 객체들이 제거된다. 다만 이 과정에서 제거된 객체의 자리에 남는 빈 공간이 낭비되면서 큰 규모의 메모리 공간이 확보되지 못하는 이른바 **메모리 단편화** 이슈가 발생할 수 있어서 후술할 카피 단계와 컴팩트 단계가 같이 쓰이는 편이다.

#### (2) 마크 카피(Mark & Copy)

<img width="70%" alt="스크린샷 2024-12-16 오전 12 03 49" src="https://github.com/user-attachments/assets/99b919a7-3801-4591-913f-bdb03d29ab09" />

객체가 존재하는 힙 공간을 **From** 공간과 **To** 공간으로 나눈다. From 공간에서 객체 할당 이후, GC가 작동해서 마킹이 마무리되면 죽은 객체를 처리하고 살아있는 객체를 **To** 공간으로 복제하게 된다. 이로써 From 공간은 텅비게 되므로 새로운 메모리 할당 영역으로 활용이 가능해진다. 복사된 곳에서는 자연스럽게 압축된 것마냥 배치되기 때문에 메모리 단편화 이슈를 해결할 수 있다.

다만, 이 방법은 결국 메모리 가용 공간이 절반으로 줄어드는(항시 To 공간을 마련해야 하므로) 단점이 발생된다. 또한 복사라는 것은 결국 객체의 이동이기 때문에 그 과정에서 드는 리소스 비용 역시 고려해야 된다. 이런 특성 때문에 객체 밀도가 낮은 경우에 효율성을 발휘할 수 있는 GC 기법이라 할 수 있다.

#### (3) 마크 컴팩트(Mark & Compact)

<img width="70%" alt="스크린샷 2024-12-16 오전 12 41 32" src="https://github.com/user-attachments/assets/d542aeb2-9fc2-4ab6-beb0-cf0724bd7f08" />

마크 단계에서 죽은 객체를 제거하고 생존 판정을 받은 남은 객체들을 메모리의 한쪽 끝으로 밀어붙이는, 이른바 **압축(Compact)** 과정을 거쳐서 메모리 연속 공간을 확보하여 단편화 이슈를 해결한다. 설명이 단순하지만, 마크 카피에 비해 구현도가 복잡하고 객체 이동 비용이 마크 카피에 비해 상대적으로 더 비싼 편에 속한다. 마크 카피의 경우는 참조 주소를 업데이트하는 방식으로 복사, 즉 **순간이동**처럼 이뤄지지만, 마크 컴팩트는 실제로 객체를 **움직여 이동**하는 방식으로 압축이 이뤄지기 때문이다.

## JVM의 GC

### 1. 서론

#### (1) Stop the world

객체의 할당은 힙에서 관리되기 때문에 곧 죽은 객체의 제거 역시 힙에서 이뤄지게 된다. 이 과정에서 일시적으로 모든 애플리케이션의 동작이 멈춰야 하는 상황이 오는데, 이것을 **Stop the world**라고 한다. 이것이 왜 발생하냐면 메모리의 일관성과 안정성을 유지, 확보하기 위해서다. 좀 더 정확히 말하자면, GC가 작동하는 스레드에서의 객체 제거를 위해 다른 모든 스레드들이 일시적으로 작동을 멈추게 된다. 이후 GC 작업이 완료되면 그제서야 모든 스레드가 작업을 재개하게 된다.

Stop the world로 인해 애플리케이션을 일시정지함으로써 안전하게 마킹된 객체들만 삭제될 수 있으나 대용량 트래픽이 요구되는 상황에서 애플리케이션이 일시중지되는 것은 심각한 성능 저하 이슈로도 받아들일 수 있다. 이것은 어떤 GC 구현체든 상관없이 전부 가지고 있는 개념이다.

#### (2) 힙의 구조

![image](https://github.com/user-attachments/assets/86a269ce-c02b-4e6f-8ea5-4c770298c34e)

객체를 저장하는 힙은 크게 **Young Gen**과 **Old Gen**으로 나눌 수 있다. 일부 GC에서는 메모리가 더 세분화될 수 있지만 보편적은 힙 구조는 위의 표준적인 세대 분리로 구현되고 동작하게 된다. 이는 **Region**이라는 개념이 같이 등장하게 되는데 이것은 후술할 GC 구현체 종류 정리에서 좀 더 알아본다.

- Young(New) Generation

새로 생성한 객체가 사용하는 영역으로 생명 주기가 짧은 객체를 GC 대상(Minor GC)으로 한다. 대부분의 객체가 금방 Unreachable 상태, 즉 사망판정을 받기 때문에, 많은 객체가 Young 영역에서 생성되었다가 사라진다. Young Gen을 구성하는 영역은 아래로 이뤄져있다.

>- Eden : new를 통해 새로 생성된 객체가 위치. 정기적인 죽은 객체 수집 후 살아남은 객체들은 Survivor로 이동한다.
>- Survivor 0,1 : S0에서 살아남은 객체는 S1으로 이동하고, S1에서 살아남은 객체는 Old 영역으로 이동한다.

- Old Generation

생명 주기가 긴 객체를 GC 대상(Major GC or Full GC)으로 하는 영역으로 Young Gen에서 살아남은 객체가 복사되는 영역으로 Young 영역보다 크게 할당되며, 영역의 크기가 큰 만큼 가비지는 적게 발생한다. 정적인 정보(static)들이 주로 해당 영역에 위치하게 된다.

세대를 거치면서 해당 위치에 따라 객체들은 다양한 GC 과정을 거쳐 정리된다. 그게 곧 위에서 언급한 Minor GC, Major GC이며 전체 영역에 대하여도 GC가 발생할 수도 있다. 참고로 Minor GC에 비해 Major GC와 Full GC 의 작업 시간이 길기 때문에 Major GC와 Full GC 가 자주 일어나게 되면 장애가 발생할 위험이 높으므로 주의하여 설계해야 한다.

>- Minor GC : Young 영역에서 실행되는 GC. 주로 0.x 초 단위로 작업이 종료된다.
>- Major GC : Old 영역에서 실행되는 GC. 수초에서 수십초까지 작업이 진행된다.
>- Full GC : Young, Old 전체 영역에 대한 GC

### 2. 종류

#### (1) G1 GC (Garbage First)

<img width="70%" alt="스크린샷 2024-12-17 오후 3 50 19" src="https://github.com/user-attachments/assets/a0466911-9e9e-4c60-aa02-93939b5deb30" />

현재(JDK 21 시점) 자바 애플리케이션을 실행할 때 동작하는 디폴트 GC 구현체다. JDK 버전 9부터 도입되었으며 **Region**이라는 개념이 적용되는 GC 구현체이기도 하다. Reigon의 의의는, 세대의 위치가 고정되어있지 않는다는 것이다.

<img src="https://github.com/user-attachments/assets/faac5b76-7a3f-4a5c-b6ac-2ccb9cd5c90c" width="50%" />

위의 이미지처럼 Region이 블록처럼 분포되어 있으며, 동적으로 위치가 바뀐다. 참고로 하나의 region 크기에도 저장되지 못할 정도로 큰 객체들(humongous objects)은 처음 객체가 저장될 때에도 Young 영역이 아닌, Old 영역으로 바로 저장된다.

![image](https://github.com/user-attachments/assets/b982e714-dc64-4edb-80b4-9626181674e5)

G1 GC에서의 동작 과정은 크게 **Young Only** 과정과 **Space Reclamation** 과정이 존재하는데, 간단히 말해서 Young Only 과정은 Old 영역으로 객체를 할당하는 것이고 Space Reclamtion은 Young 영역과 Old 영역에서의 객체를 회수해가는 과정이다. Young Only는 Stop the world 상태에서 병렬로 수행되고 Young Gen만 처리하기 때문에 STW 시간이 짧다. Space Reclamation은 힙 전체를 수집하는 Full GC를 피하고, 부분적인 공간 회수만 수행하면서 애플리케이션 실행과 병렬로 처리된다.

>#### Young Only 순서도
>1. Initial Mark : Old Region에 남아있는 객체들이 참조하고 있는 Survivor Region을 찾는다. 이때 Stop The World가 발생한다.
>2. Root Region Scan : Initial Mark 단계에서 찾은 Survivor Region에서 GC 작업 대상이 있는지 확인한다.
>3. Concurrent Mark : Heap 영역에서 전체 Region을 스캔하며 GC 대상 객체가 발견되지 않은 Region은 다음 단계에서 제외되도록 한다.
>4. Remark : 스캔이 끝나면 GC 대상에서 제외될 객체를 식별한다. 이때 Stop The World가 발생한다.
>5. Clean up : 살아있는 객체가 가장 적은 Region 부터 사용되지 않은 객체를 제거한다. 이때 Stop The World가 발생한다. 완전히 비운 Region은 재사용 가능한 형태로 동작한다.
>6. Copy : GC 대상이였지만 완전히 비워지지 않은 Region의 살아남은 객체들은 새로운 Region에 복사하여 Compaction 과정을 수행한다. 이때 Stop The World가 발생한다.

#### (2) Serial GC (직렬 GC)

<img width="613" alt="스크린샷 2024-12-17 오후 4 48 59" src="https://github.com/user-attachments/assets/fd83d49b-9d0e-4aae-85ec-39cec6766d63" />

직렬 GC는 마크-스윕 + 컴팩트 알고리즘을 활용한다. 싱글 스레드로 동작하기 때문에 작업시간이 상당히 짧은 편이며 JDK 5 ~ 6에서 디폴트로 쓰인 GC 구현체다. 다만 단일 스레드에서 동작하기 때문에 병렬 처리가 불가능하고 멀티 CPU 코어를 활용할 수 없어서 성능이 제한적이다.

실행 명령어 파라미터는 `-XX:+UseSerialGC`이다.

<img width="70%" alt="스크린샷 2024-12-17 오후 4 47 43" src="https://github.com/user-attachments/assets/4ff3b59c-ba06-4103-9e46-f07193d2124f" />

#### (3) Parellel GC (병렬 GC)

<img width="577" alt="스크린샷 2024-12-17 오후 4 52 04" src="https://github.com/user-attachments/assets/7869572a-e66f-4645-81d1-7e9f6eef5ff6" />

JDK 8까지 디폴트로 사용됐던 GC 구현체이며, 단일 스레드에서 동작하던 직렬 GC를 멀티 스레드 환경으로 옮긴 것으로 보면 된다. 동작 원리 및 알고리즘은 직렬 GC와 동일하며, 통상 Old 영역에서는 싱글 스레드로 수행되고 Young 영역에서는 멀티 스레드로 수행된다.

실행 명령어 파라미터는 `-XX:+UseParallelGC`이다.

<img width="70%" alt="스크린샷 2024-12-17 오후 4 54 22" src="https://github.com/user-attachments/assets/41b98b0e-64dc-4e5b-be34-063cc3472583" />

첫 번째 로그는 Young GC 로그를 나타내며, 두 번째 로그는 Full GC 로그를 나타낸다. 직렬 GC보다는 낫지만 멀티 스레드 특성상, 많은 스레드 수가 할당되면 병목현상이 발생할 가능성이 높아지고 메모리 소비가 오히려 과해질 가능성도 존재한다.

#### (4) CMS GC (Concurrent Mark - Sweep)

<img width="568" alt="스크린샷 2024-12-17 오후 4 57 43" src="https://github.com/user-attachments/assets/02630a4c-1aa0-4cf4-88c7-74d6c7ab80a7" />

병렬 GC가 직렬 GC에서 병렬 처리를 위한 발전 형태였다면 CMS GC는 STW의 기간을 줄이기 위해 도입된 GC 구현체이다. 기존의 직렬 GC가 직선적인 STW를 가짐으로써 애플리케이션 실행 중간의 큰 공백으로 차지하는 것을 막고자 도입된 개념이다. 다만 오히려 STW 예측이 불분명해지고 Full GC의 빈번함으로 인한 메모리 단편화 문제를 해결하지 못했기 때문에 JDK 9부터 deprecated 처리되고 JDK 14에서는 완전히 삭제됐다.

>#### CMS GC 순서도
>1. Initial Mark : Old Region에 남아있는 객체들이 참조하고 있는 Survivor Region을 찾는다. 이때 Stop The World가 발생한다.
>2. Concurrent Mark : Heap 영역에서 전체 Region을 스캔하며 GC 대상 객체가 발견되지 않은 Region은 다음 단계에서 제외되도록 한다.
>3. Remark : 스캔이 끝나면 GC 대상에서 제외될 객체를 식별한다. 이때 Stop The World가 발생한다.
>4. Concurrent Sweep : 삭제할 객체들을 처리하며, 다른 스레드와 병렬로 처리된다.

별도의 실행 명령어는 완전히 삭제됐기 때문에 존재하지 않는다. 기존 구식인 직렬 GC와 병렬 GC는 여전히 선택 가능한 옵션으로 남았음에도, CMS GC는 그 자체의 불안정한 성능과 Full GC 발생 가능성, 그리고 메모리 단편화 문제가 중요한 이슈로 받아들여진 모양이다.

<img width="70%" alt="스크린샷 2024-12-17 오후 5 06 00" src="https://github.com/user-attachments/assets/ea5fbde2-46fd-4eaa-a60b-a5e4f3d79541" />

#### (5) ZGC

<img width="50%" src="https://github.com/user-attachments/assets/06b7b905-8a8b-44c6-8038-c8256808c6ba" />

JDK 11부터 도입된 low latency 응답시간 GC 구현체로, 매우 짧은 STW를 통해 실시간 애플리케이션 등에 최적화된 GC다. ZGC의 압축 방식은, 삭제되고 난 후의 빈 곳을 찾아 채워넣는 것이 아닌 새로운 영역을 생성해서 생존 객체들을 채운다. 즉, G1 GC처럼 기존의 Region을 찾는 것이 아닌 GC가 실행될 때 마킹된 객체들을 새로운 Region을 동적으로 만들어 할당하고 죽은 객체가 차지한 Region은 GC가 수행되어 삭제된다. 다소 높은 메모리 사용량과 상대적으로 긴 초기화 시간이 단점으로 꼽힌다.

실행 명령어 파라미터는 `-XX:+UseZGC`이다.

<img width="70%" alt="스크린샷 2024-12-17 오후 5 17 51" src="https://github.com/user-attachments/assets/f1c417ff-a245-43e5-870a-4e88cb770023" />


---

*참조*<br />
*https://abiasforaction.net/understanding-jvm-garbage-collection-part-1/*<br />
*https://abiasforaction.net/understanding-jvm-garbage-collection-part-2/*<br />
*https://velog.io/@mirrorkyh/GC-%EC%A2%85%EB%A5%98%EC%99%80-%ED%8A%B9%EC%A7%95*<br />
*https://www.baeldung.com/jvm-garbage-collectors*
