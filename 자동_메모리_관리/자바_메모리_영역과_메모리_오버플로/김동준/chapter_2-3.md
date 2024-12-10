# 2.3

## JVM에서의 객체

### 1. 객체 생성

다음 코드를 기반으로 JVM에서의 객체 생성 과정을 확인한다. 현재 `psvm` 실행 전이기 때문에 런타임 데이터 영역에 아무 것도 없는 상태다.

```java
public class Main {
    public static void main(String[] args) {
        Customer customer;
        customer = new Customer();

        customer.name = "KIM";
        customer.age = 10;
        customer.address = "Seoul";
    }
}

class Customer {
    String name;
    int age;
    String address;

    public void sayHi() {
    }
}
```

#### (1) Main 클래스, main 정적 메소드

```java
public class Main {
    public static void main( // ...
```

자바는 동적 로딩을 하는 언어이기 때문에 런타임 시점에서 클래스를 로드한다. 따라서 컴파일 단계에서 자바 바이트 코드를 생성한 뒤에, JVM에서 인터프리터를 통해 줄 단위로 실행하게 된다. 실행 시점에서 `main` 메소드를 찾아서 실행해야 하므로 이를 갖는 클래스인 `Main` 클래스를 로드하면서 시작된다. 각각 클래스 정보와 정적 메소드이기 때문에 JVM의 메타스페이스에 저장된다.

<img src="https://github.com/user-attachments/assets/3a36837b-6e35-444f-8a30-b5a922454ebc" width="50%" />

#### (2) args[] 메소드 파라미터

```java
    public static void main(String[] args) { // ...
```

`main` 정적 메소드는 `String[]` 타입의 파라미터가 존재한다. 즉 메소드의 파라미터 정보가 있기 때문에 이 시점에서 `main` 메소드의 스택 프레임(정확히는 지역변수 영역)이 할당된다. 스택 프레임에 `main` 메소드의 파라미터 정보가 저장된다. 여기까지의 JVM 런타임 데이터 영역을 살펴보면 아래와 같다.

<img src="https://github.com/user-attachments/assets/14bfdbb6-55e5-4d00-bb34-1a92aa6d0fc3" width="50%" />

#### (3) Human 변수 및 인스턴스

```java
        Customer customer;
        customer = new Customer();
        // ...
```

`customer` 변수는 메소드의 지역 변수이기 때문에 스택의 지역변수 영역에 저장된다. 즉, 아까 파라미터 정보가 저장된 스택 프레임에 `customer` 변수도 같이 저장된다. 이제 `customer` 변수에 `Customer` 클래스의 생성자 메소드가 호출된 것이기 때문에, 스택에 또 다른 스택 프레임이 할당된다. 즉, `new` 키워드는 새로운 메소드를 생성하는 것과 똑같다. 또한 파라미터가 없기 때문에 별도의 지역변수 영역으로 할당되지는 않는다.

`new` 키워드와 동시에 `Customer` 클래스가 생성되어야 하므로, `Customer` 클래스 정보가 역시 메타스페이스에 저장된다. 여기까지의 JVM 런타임 데이터 영역을 들여보면 아래와 같을 것이다.

<img src="https://github.com/user-attachments/assets/032317c2-5ae4-4376-bb59-e37aef0b0792" width="50%" />


`Customer`라는 클래스 데이터가 메타스페이스에서 동적으로 로딩될 때, `Customer` 클래스 내의 정보(필드, 메소드)들 역시 메타데이터로써 포함되고, 여기서 필드의 실제 값은 힙에 할당되게 된다. 그리고 코드 상에서는 초기화 값을 별도로 지정하지 않았기 때문에 힙에서는 해당 변수들이 `null`로 초기 할당이 이뤄지게 된다. 단, `age`는 원시 타입이기 때문에 0으로 초기화가 이뤄지면서 생성된 해당 인스턴스의 참조 주소값이 `new` 스택 프레임의 `this`에 저장된다.

<img src="https://github.com/user-attachments/assets/b2d9d88e-0eef-489a-8397-88f84bd7da08" width="50%" />

#### (4) 인스턴스 필드 변경

```java
        customer.name = "KIM";
        customer.age = 10;
        customer.address = "Seoul";
```

이제 인스턴스가 생성됐기 때문에 해당 참조 주소값을 `main` 스택 프레임의 지역변수인 `consumer`가 바라볼 수 있도록 할당 후, `new`는 생성자로써의 역할이 끝난(즉, 호출이 종료된) 상태이므로 현재 스택에서 사라지게 된다. 그리고 기존의 0으로 초기화된 원시 타입의 변수(`age`)는 새로운 값이 들어오면 해당 값으로 대체되고, `null`이 할당됐던 래퍼 타입들의 변수가 바라보는 힙의 공간에 해당 타입의 새로운 값으로 대체된다.

<img src="https://github.com/user-attachments/assets/f0da84c9-31d4-4901-b27b-36ca023ba97a" width="50%" />

### 2. 문자열 객체의 취급

#### (1) JVM 문자열 풀

`String` 역시 대표적인 자바의 래퍼 타입 클래스다. 근데 클래스의 인스턴스를 생성함에 있어 항상 생성자를 썼는데, 문자열은 조금 다르게 취급된다.

```java
public class StringTest {
    public static void main(String[] args) {
        String string1 = new String("이것은 문자열입니다");
        String string2 = new String("이것은 문자열입니다");
        String string3 = "이것은 문자열입니다";
        String string4 = "이것은 문자열입니다";

        // equals() 메소드는 문자열의 '내용'을 확인하는 것이므로 사용 x
        System.out.println("string1, string2 비교: " + (string1 == string2));
        System.out.println("string1, string3 비교: " + (string1 == string3));
        System.out.println("string3, string4 비교: " + (string3 == string4));
    }
}
```

위의 코드는 아래와 같이 JVM에 배치될 것이다.

>- 스택: `string1` 변수, `string2` 변수, `string3` 변수, `string4` 변수
>- 힙: 해당 변수들의 값

테스트 코드를 돌려서 각 객체들의 해시코드를 비교해보면 아래와 같은 결과가 나온다.

<img width="70%" alt="스크린샷 2024-12-11 오전 1 15 47" src="https://github.com/user-attachments/assets/0b7cadbb-2e91-495b-9960-5b84829dafc9">

`string1`처럼 생성된 방식은 우리가 흔히 아는 **생성자**로 생성된 것이다. 그리고 `string3` 역시 새롭게 문자열 객체가 생성됐으나 생성자가 아닌 **리터럴**로 생성됐다. 리터럴은 쉽게 말해서 큰따옴표 쌍이라고 생각하면 된다.

자바에서는 문자열을 생성자가 아닌 리터럴로 생성하게 되면, 컴파일 단게에서 문자열 객체 생성 + **interning**이라는 단계를 거친다(이하 인터닝). 인터닝은 JVM의 힙에 존재하는 문자열 풀(JDK 버전 9부터 메소드 영역에서 힙으로 관리 책임이 이전)에 **기존에 저장됐던 문자열 값**을 재사용하는 메모리 최적화 기법이다. 즉, 위에서 `string1` 변수는 리터럴로 생성됐기 때문에 힙의 문자열 풀에 저장되고, 여기서 `string2` 역시 리터럴로 생성되면서 해당 변수에 할당된 문자열 객체의 값이 문자열 풀에 존재하는지를 확인한다.

문자열 풀에 이미 `string2`의 문자열 객체 값이 `string1`으로 인해 존재하기 때문에 해당 문자열 객체를 반환해주면서 결과적으로 `string1` 문자열 객체와 `string2` 문자열 객체가 동일하게 되는 것이다. 반면 생성자를 통해 생성된 문자열은 힙에 새로운 객체가 지속적으로 생성될 뿐, 문자열 풀에 저장되지 않으므로 다른 참조 주소값을 가지게 되므로 위의 테스트 코드에서 `false`가 나오게 된 것이다.

<img width="70%" alt="스크린샷 2024-12-11 오전 1 15 47" src="https://github.com/user-attachments/assets/bcca33bb-0cd0-4e7a-93c8-82a21494f617">

#### (2) 인터닝

`String` 클래스를 뒤져보면 `intern()` 이라는 네이티브 메소드가 보인다.

<img width="70%" alt="스크린샷 2024-12-11 오전 1 54 30" src="https://github.com/user-attachments/assets/2205c520-9375-45fd-9cc0-789a523e2c1b">

`intern()` 메소드를 호출해서 할당하면 문자열의 인터닝 과정을 수동으로 수행할 수 있다. 실제로 테스트 코드를 수정해서 수행하면 아래처럼 나온다.

```java
public class StringTest {
    public static void main(String[] args) {
        String string1 = new String("이것은 문자열입니다");
        String string2 = new String("이것은 문자열입니다");
        String string3 = "이것은 문자열입니다";  // 문자열 풀에 존재

        string1 = string1.intern();  // 문자열 풀에 존재

        System.out.println("인터닝 이후의 string1, string2 비교: " + (string1 == string2));
        System.out.println("인터닝 이후의 string1, string3 비교: " + (string1 == string3));
    }
}
```

<img width="70%" alt="스크린샷 2024-12-11 오전 1 58 59" src="https://github.com/user-attachments/assets/616a3f67-ed9b-4d6b-ad81-0a899148dba4">

`string1`에 대해 인터닝을 수동 실행하면서 `string1`의 객체 역시 문자열 풀에 포함되었고 그 과정에서 이미 `string3` 객체가 문자열 풀에 존재하기 때문에 대체됨으로써 동일한 객체가 된다. 그렇기 때문에 `string1`과 `string3`를 해시코드를 비교하면 `true`가 나오게 되는 것이다. 반면 `string2`는 여전히 생성자로만 생성된 문자열이기 때문에 문자열 풀에 없으므로 여전히 비교했을 때 `false`가 나온다.

다만 수동으로 저 네이티브 메소드를 굳이굳이 호출할 필요는 거의 없을 것이다. 문자열을 리터럴로 생성되면 자연스럽게 인터닝 과정을 거치므로 최적화 및 효율성을 추구할 수 있기 때문이다. 여담으로 문자열 풀이 메소드 영역에서 힙으로 넘어옴에 따라, 가바지 컬렉팅의 대상이 됐다는 점에서 의의를 가진다.

### 3. 객체 메모리 레이아웃

#### (1) 헤더(Header)
- 객체 정보를 포함 (예: 클래스 메타데이터, GC 관련 정보, 동기화 정보)  
- JVM 구현에 따라 크기가 다를 수 있지만 보통 12~16바이트
#### (2) 인스턴스 데이터(Instance Data)
- 실제 객체의 필드값이 저장되는 영역
- 객체의 데이터 구조에 따라 메모리 크기 결정  
#### (3) 패딩(Padding)
- JVM이 메모리 정렬(alignment)을 유지하기 위해 추가하는 공간
- 보통 8바이트 배수로 정렬

### 4. 객체 접근방식

#### (1) 참조 vs offset
- 객체는 JVM Heap에 저장되고, 참조(reference)는 메모리 주소를 가리킴
- 필드 접근 시 JVM은 참조를 기반으로 필드의 메모리 offset을 계산하여 접근
#### (2) Unsafe 클래스의 역할
- 자바 내부 API로 메모리와 객체를 직접 조작할 수 있음
- 필드 offset 계산, 메모리 할당, 직접 접근 등에 사용
- 성능 최적화와 저수준 작업에 활용되지만, 안전하지 않아 사용에 주의 필요
