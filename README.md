# spring-transaction-propagation-db2


# 0. 프로젝트 설정

### **비즈니스 요구사항**

회원을 등록하고 조회하는 기능을 가집니다.

회원에 대한 변경 이력을 추적할 수 있도록 회원 데이터가 변경될 때 변경 이력을 DB LOG 테이블에 남겨야 합니다.

- 예제를 단순화 하기 위해 회원 등록시에만 DB LOG 테이블에 남기도록 하겠습니다.

`Member`

```java
@Entity
@Getter @Setter
public class Member {
  
    @Id
    @GeneratedValue
    private Long id;
    private String username;
  
    public Member() {
    }
  
    public Member(String username) {
        this.username = username;
    }
}
```

JPA를 통해 관리하는 회원 엔티티입니다.

`MemberRepository`

```java
@Slf4j
@Repository
@RequiredArgsConstructor
public class MemberRepository {
  
    private final EntityManager em;
  
    @Transactional
    public void save(Member member) {
        log.info("member 저장");
        em.persist(member);
    }
  
    public Optional<Member> find(String username) {
        return em.createQuery("select m from Member m where         m.username=:username", Member.class)
            .setParameter("username", username)
            .getResultList().stream().findAny();
    }
}
```

JPA를 사용하는 회원 리포지토리입니다. 저장과 조회 기능을 제공합니다.

`Log`

```java
@Entity
@Getter @Setter
public class Log {
    @Id
    @GeneratedValue
    private Long id;
    private String message;
  
    public Log() {
    }
  
    public Log(String message) {
        this.message = message;
    }
}
```

JPA를 통해 관리하는 로그 엔티티입니다.

`LogRepository`

```java
@Slf4j
@Repository
@RequiredArgsConstructor
public class LogRepository {
  
    private final EntityManager em;
  
    @Transactional
    public void save(Log logMessage) {
        log.info("log 저장");
        em.persist(logMessage);
      
        if (logMessage.getMessage().contains("로그예외")) {
            log.info("log 저장시 예외 발생");
            throw new RuntimeException("예외 발생");
        }
    }
  
    public Optional<Log> find(String message) {
        return em.createQuery("select l from Log l where l.message = :message", Log.class)
            .setParameter("message", message)
            .getResultList().stream().findAny();
    }
}
```

JPA를 사용하는 로그 리포지토리입니다. 저장과 조회 기능을 제공합니다.

중간에 예외 상황을 재현하기 위해 `로그예외`라고 입력하는 경우 예외를 발생시키도록 만들었습니다.

`MemberService`

```java
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class MemberService {
  
    private final MemberRepository memberRepository;
    private final LogRepository logRepository;
  
    public void joinV1(String username) {
        Member member = new Member(username);
        Log logMessage = new Log(username);
      
        log.info("== memberRepository 호출 시작 ==");
        memberRepository.save(member);
        log.info("== memberRepository 호출 종료 ==");
      
        log.info("== logRepository 호출 시작 ==");
        logRepository.save(logMessage);
        log.info("== logRepository 호출 종료 ==");
    }
  
    public void joinV2(String username) {
        Member member = new Member(username);
        Log logMessage = new Log(username);
      
        log.info("== memberRepository 호출 시작 ==");
        memberRepository.save(member);
        log.info("== memberRepository 호출 종료 ==");
      
        log.info("== logRepository 호출 시작 ==");
        try {
            logRepository.save(logMessage);
        } catch (RuntimeException e) {
            log.info("log 저장에 실패했습니다. logMessage={}", logMessage.getMessage());
            log.info("정상 흐름 변환");
        }
        log.info("== logRepository 호출 종료 ==");
    }
}
```

회원을 등록하면서 동시에 회원 등록에 대한 DB 로그도 함께 남깁니다.

`joinV1()`

회원과 DB로그를 함께 남기는 비즈니스 로직입니다.

현재 별도의 트랜잭션은 설정하지 않은 상태입니다.

`joinV2()`

`joinV1()` 과 같은 기능을 수행합니다.

DB로그 저장 시 예외가 발생하면 예외를 복구합니다.

현재 별도의 트랜잭션은 설정하지 않습니다.

`MemberServiceTest`

```java
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.junit.jupiter.api.Assertions.assertTrue;

@Slf4j
@SpringBootTest
class MemberServiceTest {
    @Autowired
    MemberService memberService;
    @Autowired
    MemberRepository memberRepository;
    @Autowired
    LogRepository logRepository;
  
    /**
    * MemberService @Transactional:OFF
    * MemberRepository @Transactional:ON
    * LogRepository @Transactional:ON
    */
    @Test
    void outerTxOff_success() {
        //given
        String username = "outerTxOff_success";
        //when
        memberService.joinV1(username);
        //then: 모든 데이터가 정상 저장된다.
        assertTrue(memberRepository.find(username).isPresent());
        assertTrue(logRepository.find(username).isPresent());
    }
}
```

**참고**

JPA의 구현체인 하이버네이트가 테이블을 자동으로 생성합니다.

메모리 DB이기 때문에 모든 테스트가 완료된 이후에 DB는 사라집니다.

여기서는 각각의 테스트가 완료된 시점에 데이터를 삭제하지 않으므로 `username`은 테스트별로 각각 다르게 설정해야 합니다. 그렇지 않으면 다음 테스트에 영향을 줍니다. (모든 테스트가 완료되어야 DB가 사라짐)

**JPA와 데이터 변경**

JPA를 통한 모든 데이터 변경(등록, 수정, 삭제)에는 트랜잭션이 필요합니다. (조회는 트랜잭션 없이 가능)

현재 코드에서 서비스 계층에 트랜잭션이 없기 때문에 리포지토리에 트랜잭션이 있습니다.


# 1. 커밋, 롤백

### **서비스 계층에 트랜잭션이 없을 때 - 커밋**

**상황**

- 서비스 계층에 트랜잭션이 없음
- 회원, 로그 리포지토리가 각각 트랜잭션을 가지고 있음
- 회원, 로그 리포지토리 둘다 커밋에 성공

`outerTxOff_success`

```java
/**
* MemberService @Transactional:OFF
* MemberRepository @Transactional:ON
* LogRepository @Transactional:ON
*/
@Test
void outerTxOff_success() {
    //given
    String username = "outerTxOff_success";
  
    //when
    memberService.joinV1(username);
  
    //then: 모든 데이터가 정상 저장된다.
    assertTrue(memberRepository.find(username).isPresent());
    assertTrue(logRepository.find(username).isPresent());
}
```

![https://user-images.githubusercontent.com/52024566/209253746-78d82f6b-1b70-4f2c-8e49-a5074f1f4ab8.png](https://user-images.githubusercontent.com/52024566/209253746-78d82f6b-1b70-4f2c-8e49-a5074f1f4ab8.png)

![https://user-images.githubusercontent.com/52024566/209253749-797b7067-58ff-40ae-a68b-d6b4b4a3b421.png](https://user-images.githubusercontent.com/52024566/209253749-797b7067-58ff-40ae-a68b-d6b4b4a3b421.png)

 1. `MemberService`에서 `MemberRepository`를 호출합니다. `MemberRepository`에는 `@Transactional` 애노테이션이 있으므로 트랜잭션 AOP가 작동합니다. 여기서 트랜잭션 매니저를 통해 트랜잭션을 시작합니다.

- 그림에서는 생략했지만, 트랜잭션 매니저에 트랜잭션을 요청하면 데이터소스를 통해 커넥션 `con1`을 획득하고, 해당 커넥션을 수동 커밋 모드로 변경해서 트랜잭션을 시작합니다.
- 그리고 트랜잭션 동기화 매니저를 통해 트랜잭션을 시작한 커넥션을 보관합니다.
- 트랜잭션 매니저의 호출 결과로 `status`를 반환합니다. 여기서는 신규 트랜잭션 여부가 `true` 입니다.

 2. `MemberRepository`는 JPA를 통해 회원을 저장하는데, 이때 JPA는 트랜잭션이 시작된 `con1`을 사용해서 회원을 저장합니다.

 3. `MemberRepository` 가 정상 응답을 반환했기 때문에 트랜잭션 AOP는 트랜잭션 매니저에 커밋을 요청합니다.

 4. 트랜잭션 매니저는 `con1` 을 통해 물리 트랜잭션을 커밋합니다.

- 물론 이 시점에 앞서 설명한 신규 트랜잭션 여부, `rollbackOnly` 여부를 모두 체크합니다.

이렇게 해서 `MemberRepository`와 관련된 모든 데이터는 정상 커밋되고, 트랜잭션B는 완전히 종료됩니다. 

이후에 `LogRepository`를 통해 트랜잭션C를 시작하고, 정상 커밋합니다. 

결과적으로 둘다 커밋되었으므로 `Member`, `Log` 모두 안전하게 저장됩니다.

**@Transactional과 REQUIRED**

트랜잭션 전파의 기본 값은 `REQUIRED`입니다. 따라서 아래 두 코드는 같습니다.

- `@Transactional(propagation = Propagation.REQUIRED)`
- `@Transactional`

`REQUIRED`는 기존 트랜잭션이 없으면 새로운 트랜잭션을 만들고, 기존 트랜잭션이 있으면 참여합니다.

### **서비스 계층에 트랜잭션이 없을 때 - 롤백**

**아래와 같은 상황이라고 가정합시다.**

- 서비스 계층에 트랜잭션이 없음
- 회원, 로그 리포지토리가 각각 트랜잭션을 가지고 있음.
- 회원 리포지토리는 정상 동작하지만 로그 리포지토리에서 예외가 발생

`outerTxOff_fail`

```java
/**
* MemberService @Transactional:OFF
* MemberRepository @Transactional:ON
* LogRepository @Transactional:ON Exception
*/
@Test
void outerTxOff_fail() {
    //given
    String username = "로그예외_outerTxOff_fail";
  
    //when
    assertThatThrownBy(() -> memberService.joinV1(username))
        .isInstanceOf(RuntimeException.class);
  
    //then: 완전히 롤백되지 않고, member 데이터가 남아서 저장된다.
    assertTrue(memberRepository.find(username).isPresent());
    assertTrue(logRepository.find(username).isEmpty());
}
```

사용자 이름에 `로그예외`라는 단어가 포함되어 있으면 `LogRepository`에서 런타임 예외가 발생합니다.

트랜잭션 AOP는 해당 런타임 예외를 확인하고 롤백 처리합니다.

**로그예외 로직**

```java
if (logMessage.getMessage().contains("로그예외")) {
    log.info("log 저장시 예외 발생");
    throw new RuntimeException("예외 발생");
}
```

![https://user-images.githubusercontent.com/52024566/209253753-b78cba3b-a7b3-438e-97ea-18ec49088da4.png](https://user-images.githubusercontent.com/52024566/209253753-b78cba3b-a7b3-438e-97ea-18ec49088da4.png)

![https://user-images.githubusercontent.com/52024566/209253754-548d98af-6eb6-4e1f-bfd1-a053f504fdf8.png](https://user-images.githubusercontent.com/52024566/209253754-548d98af-6eb6-4e1f-bfd1-a053f504fdf8.png)

`MemberService`에서 `MemberRepository`를 호출하는 부분은 앞서 설명한 내용과 같습니다. 트랜잭션이 정상 커밋되고, 회원 데이터도 DB에 정상 반영됩니다.

`MemberService`에서 `LogRepository`를 호출하는데, 로그예외 라는 이름을 전달합니다. 이 과정에서 새로운 트랜잭션 C가 생성됩니다.

**LogRepository 응답 로직**

 1. `LogRepository`는 트랜잭션C와 관련된 `con2`를 사용합니다.

 2. `로그예외`라는 이름을 전달해서 `LogRepository`에 런타임 예외가 발생합니다.

 3. `LogRepository` 는 해당 예외를 밖으로 던집니다. 이 경우 트랜잭션 AOP가 예외를 받습니다.

 4. 런타임 예외가 발생해서 트랜잭션 AOP는 트랜잭션 매니저에 롤백을 호출합니다.

 5. 트랜잭션 매니저는 이 트랜잭션이 **신규 트랜잭션이므로 물리 롤백을 호출**합니다.

> **참고** - 트랜잭션 AOP도 결국 내부에서는 트랜잭션 매니저를 사용합니다.
이 경우 회원은 저장되지만, 회원 이력 로그는 롤백됨. 따라서 데이터 정합성에 문제가 발생할 수 있음.
>

# 2. 단일 트랜잭션

### **트랜잭션 하나만 사용하기**

회원 리포지토리와 로그 리포지토리를 하나의 트랜잭션으로 묶는 가장 간단한 방법은 **이 둘을 호출하는 회원 서비스에만 트랜잭션을 사용**하는 것입니다.

그에 맞게 코드를 수정하고 Test 메서드를 수정합시다.

`MemberService - joinV1()`

```java
@Transactional //추가
public void joinV1(String username)
```

`MemberRepository - save()`

```java
//@Transactional //제거
public void save(Member member)
```

`LogRepository - save()`

```java
//@Transactional //제거
public void save(Log logMessage)
```

`MemberRepository`, `LogRepository`의 `@Transactional` 코드를 제거

`MemberService`에만 `@Transactional` 코드를 추가

`singleTx`

```java
/**
* MemberService @Transactional:ON
* MemberRepository @Transactional:OFF
* LogRepository @Transactional:OFF
*/
@Test
void singleTx() {
    //given
    String username = "singleTx";
  
    //when
    memberService.joinV1(username);
  
    //then: 모든 데이터가 정상 저장된다.
    assertTrue(memberRepository.find(username).isPresent());
    assertTrue(logRepository.find(username).isPresent());
}
```

![https://user-images.githubusercontent.com/52024566/209555139-c96d9341-01f2-4abd-8fe0-afae51e814c5.png](https://user-images.githubusercontent.com/52024566/209555139-c96d9341-01f2-4abd-8fe0-afae51e814c5.png)

이렇게 하면 `MemberService`를 시작할 때 부터 종료할 때 까지의 모든 로직을 하나의 트랜잭션으로 묶을 수 있습니다.

- 물론 `MemberService`가 `MemberRepository`, `LogRepository`를 호출하므로 이 로직들은 같은 트랜잭션을 사용합니다.

`MemberService`만 트랜잭션을 처리하기 때문에 앞서 배운 논리 트랜잭션, 물리 트랜잭션, 외부 트랜잭션, 내부 트랜잭션, `rollbackOnly`, 신규 트랜잭션, 트랜잭션 전파와 같은 복잡한 것을 고민할 필요가 없습니다.

이렇게 아주 단순하고 깔끔하게 트랜잭션을 묶을 수 있습니다.

![https://user-images.githubusercontent.com/52024566/209555144-549adabe-0ac3-40ae-a8f7-1c81745ba580.png](https://user-images.githubusercontent.com/52024566/209555144-549adabe-0ac3-40ae-a8f7-1c81745ba580.png)

`@Transactional`이 `MemberService`에만 붙어있기 때문에 **MemberService만 트랜잭션 AOP가 적용**됩니다.

`MemberRepository`, `LogRepository`는 트랜잭션 AOP가 적용되지 않습니다.

`MemberService`의 시작부터 끝까지, 관련 로직은 해당 트랜잭션이 생성한 커넥션을 사용합니다.

`MemberService`가 호출하는 `MemberRepository`, `LogRepository`도 같은 커넥션을 사용하면서 자연스럽게 트랜잭션 범위에 포함합니다.

> **참고 -** 같은 쓰레드를 사용하면 트랜잭션 동기화 매니저는 같은 커넥션을 반환
> 

### **각각 트랜잭션이 필요한 상황**

하지만 만약 아래 그림처럼 각각 트랜잭션이 필요하면 어떻게 할까요?

![https://user-images.githubusercontent.com/52024566/209555145-ac22fee8-7d72-4e6c-866b-6882a8fadc92.png](https://user-images.githubusercontent.com/52024566/209555145-ac22fee8-7d72-4e6c-866b-6882a8fadc92.png)

**트랜잭션 적용 범위**

![https://user-images.githubusercontent.com/52024566/209555149-4d21f889-93be-40cd-831b-6d80ce322c95.png](https://user-images.githubusercontent.com/52024566/209555149-4d21f889-93be-40cd-831b-6d80ce322c95.png)

클라이언트 A는 `MemberService`부터 `MemberRepository` , `LogRepository`를 모두 하나의 트랜잭션으로 묶고 싶습니다.

클라이언트 B는 `MemberRepository`만 호출하고 여기에만 트랜잭션을 사용하고 싶습니다.

클라이언트 C는 `LogRepository`만 호출하고 여기에만 트랜잭션을 사용하고 싶습니다.

클라이언트 A만 생각하면 `MemberService`에 트랜잭션 코드를 남기고, `MemberRepository`, `LogRepository`의 트랜잭션 코드를 제거하면 앞서 배운 것 처럼 깔끔하게 하나의 트랜잭션을 적용할 수 있습니다.

하지만 이렇게 되면 클라이언트 B, C가 호출하는 `MemberRepository`, `LogRepository`에는 트랜잭션을 적용할 수 없습니다.

트랜잭션 전파 없이 이런 문제를 해결하려면 아마 **트랜잭션이 있는 메서드와 트랜잭션이 없는 메서드를 각각 만들어야** 합니다. 이 경우에는 **더 복잡**하게 다음과 같은 상황이 발생할 수도 있습니다.

![https://user-images.githubusercontent.com/52024566/209555303-251aaddc-88f8-449f-a043-4f3fb89ed5ab.png](https://user-images.githubusercontent.com/52024566/209555303-251aaddc-88f8-449f-a043-4f3fb89ed5ab.png)

클라이언트 Z가 호출하는 `OrderService`에서도 트랜잭션을 시작할 수 있어야 하고, 클라이언트A가 호출하는 `MemberService`에서도 트랜잭션을 시작할 수 있어야 합니다.

결국 **이런 문제를 해결하기 위해 트랜잭션 전파가 필요하게 됩니다.**

# 3. 전파 커밋

스프링은 `@Transactional`이 적용되어 있으면 기본으로 `REQUIRED`라는 전파 옵션을 사용합니다. 

이 옵션은 이전 글에서도 설명했듯 기존 트랜잭션이 없으면 트랜잭션을 생성하고, 기존 트랜잭션이 있으면 기존 트랜잭션에 참여합니다. 참여한다는 뜻은 해당 트랜잭션을 그대로 따른다는 뜻이고, 동시에 같은 동기화 커넥션을 사용한다는 의미입니다.

![https://user-images.githubusercontent.com/52024566/209814530-dddac598-76ef-4f44-b482-9e76ad47a4cf.png](https://user-images.githubusercontent.com/52024566/209814530-dddac598-76ef-4f44-b482-9e76ad47a4cf.png)

이렇게 둘 이상의 트랜잭션이 하나의 물리 트랜잭션에 묶이게 되면 둘을 구분하기 위해 논리 트랜잭션과 물리 트랜잭션으로 구분합니다.

**신규 트랜잭션**

![https://user-images.githubusercontent.com/52024566/209814539-fa02aa35-e2ea-4f8d-a900-01eb328e445f.png](https://user-images.githubusercontent.com/52024566/209814539-fa02aa35-e2ea-4f8d-a900-01eb328e445f.png)

이 경우 외부에 있는 신규 트랜잭션만 실제 물리 트랜잭션을 시작하고 커밋합니다.

내부에 있는 트랜잭션은 물리 트랜잭션 시작하거나 커밋하지 않습니다.

**모든 논리 트랜잭션 커밋**

![https://user-images.githubusercontent.com/52024566/209814542-8b9338f0-457a-4402-98ab-c7422e5ab399.png](https://user-images.githubusercontent.com/52024566/209814542-8b9338f0-457a-4402-98ab-c7422e5ab399.png)

모든 논리 트랜잭션을 커밋해야 물리 트랜잭션도 커밋됩니다. 하나라도 롤백되면 물리 트랜잭션은 롤백됩니다.

### 모든 논리 트랜잭션이 정상 커밋되는 경우

`outerTxOn_success`

```java
/**
* MemberService @Transactional:ON
* MemberRepository @Transactional:ON
* LogRepository @Transactional:ON
*/
@Test
void outerTxOn_success() {
    //given
    String username = "outerTxOn_success";
  
    //when
    memberService.joinV1(username);
  
    //then: 모든 데이터가 정상 저장된다.
    assertTrue(memberRepository.find(username).isPresent());
    assertTrue(logRepository.find(username).isPresent());
}
```

![https://user-images.githubusercontent.com/52024566/209814544-0f3a5df4-d9da-49ce-80cd-0f192c9ce31e.png](https://user-images.githubusercontent.com/52024566/209814544-0f3a5df4-d9da-49ce-80cd-0f192c9ce31e.png)

클라이언트A(여기서는 테스트 코드)가 `MemberService`를 호출하면서 트랜잭션 AOP 을 호출합니다.

여기서 신규 트랜잭션이 생성되고, 물리 트랜잭션도 시작됩니다.

`MemberRepository`를 호출하면서 트랜잭션 AOP 호출

- 이미 트랜잭션이 있으므로 기존 트랜잭션에 참여합니다.
- `MemberRepository`의 로직 호출이 끝나고 정상 응답하면 트랜잭션 AOP 을 호출합니다.
- 트랜잭션 AOP는 정상 응답이므로 트랜잭션 매니저에 커밋을 요청합니다. 이 경우 신규 트랜잭션이 아니므로 실제 커밋은 호출하지 않습니다.

`LogRepository`를 호출하면서 트랜잭션 AOP 호출

- 이미 트랜잭션이 있으므로 기존 트랜잭션에 참여합니다.
- `LogRepository`의 로직 호출이 끝나고 정상 응답하면 트랜잭션 AOP 을 호출합니다.
- 트랜잭션 AOP는 정상 응답이므로 트랜잭션 매니저에 커밋을 요청합니다. 이 경우 신규 트랜잭션이 아니므로 실제 커밋(물리 커밋)을 호출하지 않습니다.

`MemberService`의 로직 호출이 끝나고 정상 응답하면 트랜잭션 AOP 을 호출합니다.

- 트랜잭션 AOP는 정상 응답이므로 트랜잭션 매니저에 커밋을 요청합니다. 이 경우 신규 트랜잭션이므로 물리 커밋을 호출합니다.

# 4. 전파 롤백

**로그 리포지토리에서 예외가 발생해서 전체 트랜잭션이 롤백되는 경우**

![https://user-images.githubusercontent.com/52024566/209965432-1c44e100-3162-4d09-863a-70368fc39b9b.png](https://user-images.githubusercontent.com/52024566/209965432-1c44e100-3162-4d09-863a-70368fc39b9b.png)

`outerTxOn_fail`

```java
/**
* MemberService @Transactional:ON
* MemberRepository @Transactional:ON
* LogRepository @Transactional:ON Exception
*/
@Test
void outerTxOn_fail() {
    //given
    String username = "로그예외_outerTxOn_fail";
  
    //when
    assertThatThrownBy(() -> memberService.joinV1(username))
				    .isInstanceOf(RuntimeException.class);
  
    //then: 모든 데이터가 롤백된다.
    assertTrue(memberRepository.find(username).isEmpty());
    assertTrue(logRepository.find(username).isEmpty());
}
```

여기서는 `로그예외`라고 넘겼기 때문에 `LogRepository`에서 런타임 예외가 발생합니다.

**흐름**

![https://user-images.githubusercontent.com/52024566/209965438-0e381d89-a030-4f69-abf5-402ea6be9c1a.png](https://user-images.githubusercontent.com/52024566/209965438-0e381d89-a030-4f69-abf5-402ea6be9c1a.png)

클라이언트A가 `MemberService`를 호출하면서 트랜잭션 AOP 호출

- 여기서 신규 트랜잭션이 생성되고, 물리 트랜잭션도 시작

`MemberRepository`를 호출하면서 트랜잭션 AOP 호출

- 이미 트랜잭션이 있으므로 기존 트랜잭션에 참여

`MemberRepository`의 로직 호출이 끝나고 정상 응답하면 트랜잭션 AOP 호출

- 트랜잭션 AOP는 정상 응답이므로 트랜잭션 매니저에 커밋을 요청. 이 경우 신규 트랜잭션이 아니므로 실제 커밋을 호출하지 않음

`LogRepository`를 호출하면서 트랜잭션 AOP 호출

- 이미 트랜잭션이 있으므로 기존 트랜잭션에 참여
- `LogRepository` 로직에서 런타임 예외가 발생. 예외를 던지면 트랜잭션 AOP가 해당 예외를 받음
    - 트랜잭션 AOP는 런타임 예외가 발생했으므로 트랜잭션 매니저에 롤백을 요청. 이 경우 신규 트랜잭션이 아니므로 물리 롤백을 호출하지는 않고 `rollbackOnly`를 설정
    - `LogRepository`가 예외를 던졌기 때문에 트랜잭션 AOP도 해당 예외를 그대로 밖으로 던짐

`MemberService`에서도 런타임 예외를 받게 되는데, 여기 로직에서는 해당 런타임 예외를 처리하지 않고 밖으로 던짐.

- 트랜잭션 AOP는 런타임 예외가 발생했으므로 트랜잭션 매니저에 롤백을 요청. 이 경우 신규 트랜잭션이므로 물리 롤백을 호출
- 참고로 이 경우 어차피 롤백이 되었기 때문에, `rollbackOnly` 설정은 참고하지 않음

`MemberService`가 예외를 던졌기 때문에 트랜잭션 AOP도 해당 예외를 그대로 밖으로 던짐

- 이렇게 클라이언트A는 `LogRepository`부터 넘어온 런타임 예외를 받게 됩니다.

### **정리**

회원과 회원 이력 로그를 처리하는 부분을 하나의 트랜잭션으로 묶은 덕분에 문제가 발생했을 때 회원과 회원 이력 로그가 모두 함께 롤백됩니다. 따라서 데이터 정합성에 문제가 발생하지 않습니다.

# 5. 복구 REQUIRED

앞서 회원과 로그를 하나의 트랜잭션으로 묶어서 데이터 정합성 문제를 깔끔하게 해결하였습니다. 

그런데 회원 이력 로그를 DB에 남기는 작업에 가끔 문제가 발생해서 회원 가입 자체가 안되는 경우가 가끔 발생합니다. 그래서 사용자들이 회원 가입에 실패해서 이탈하는 문제가 발생할 수 있죠. 

회원 이력 로그의 경우 여러가지 방법으로 추후에 복구가 가능할 것으로 보입니다. 그래서 비즈니스 요구사항이 변경되었습니다. **회원 가입을 시도한 로그를 남기는데 실패하더라도 회원 가입은 유지되어야 합니다.**

![https://user-images.githubusercontent.com/52024566/210072954-0ea00036-38a0-4218-89a3-327475b31e5d.png](https://user-images.githubusercontent.com/52024566/210072954-0ea00036-38a0-4218-89a3-327475b31e5d.png)

단순하게 생각해보면 `LogRepository`에서 예외가 발생하면 그것을 `MemberService`에서 예외를 잡아서 처리하면 될 것 같습니다.

이렇게 하면 `MemberService`에서 정상 흐름으로 바꿀 수 있기 때문에 `MemberService`의 트랜잭션 AOP에서 커밋을 수행할 수 있을 것으로 보입니다.

.

그런데 이 방법은 실패합니다. 이 방법이 왜 실패하는지 예제를 통해서 확인해봅시다.

`recoverException_fail`

```java
/**
* MemberService @Transactional:ON
* MemberRepository @Transactional:ON
* LogRepository @Transactional:ON Exception
*/
@Test
void recoverException_fail() {
    //given
    String username = "로그예외_recoverException_fail";
  
    //when
    assertThatThrownBy(() -> memberService.joinV2(username))
        .isInstanceOf(UnexpectedRollbackException.class);
  
    //then: 모든 데이터가 롤백된다.
    assertTrue(memberRepository.find(username).isEmpty());
    assertTrue(logRepository.find(username).isEmpty());
}
```

여기서 `memberService.joinV2()`를 호출하는 부분을 주의해야 합니다. `joinV2()`에는 예외를 잡아서 정상 흐름으로 변환하는 로직이 추가됩니다.

```java
try {
    logRepository.save(logMessage);
} catch (RuntimeException e) {
    log.info("log 저장에 실패했습니다. logMessage={}", logMessage);
    log.info("정상 흐름 변환");
}
```

![https://user-images.githubusercontent.com/52024566/210072960-9546f7b5-d68d-4c2d-92a9-7499354adcaf.png](https://user-images.githubusercontent.com/52024566/210072960-9546f7b5-d68d-4c2d-92a9-7499354adcaf.png)

내부 트랜잭션에서 `rollbackOnly`를 설정하기 때문에 결과적으로 정상 흐름 처리를 해서 외부 트랜잭션에서 커밋을 호출해도 물리 트랜잭션은 롤백됩니다.

그리고 `UnexpectedRollbackException` 이 던져집니다.

### **전체 흐름**

![https://user-images.githubusercontent.com/52024566/210072963-c8cffbd9-1ecb-41d0-b3a9-4ce79a84d54c.png](https://user-images.githubusercontent.com/52024566/210072963-c8cffbd9-1ecb-41d0-b3a9-4ce79a84d54c.png)

`LogRepository`에서 예외가 발생합니다. 예외를 던지면 `LogRepository`의 트랜잭션 AOP가 해당 예외를 받습니다.

- 신규 트랜잭션이 아니므로 물리 트랜잭션을 롤백하지는 않고, 트랜잭션 동기화 매니저에 `rollbackOnly`를 표시

이후 트랜잭션 AOP는 전달 받은 예외를 밖으로 던집니다.

- 예외가 `MemberService`에 던져지고, `MemberService`는 해당 예외를 복구한다. 그리고 정상적으로 리턴.

정상 흐름이 되었으므로 `MemberService`의 트랜잭션 AOP는 커밋을 호출합니다.

- 커밋을 호출할 때 신규 트랜잭션이므로 실제 물리 트랜잭션을 커밋해야 함. 이때 `rollbackOnly`를 체크
- `rollbackOnly`가 체크 되어 있으므로 물리 트랜잭션을 롤백
- 트랜잭션 매니저는 `UnexpectedRollbackException` 예외를 던짐
- 트랜잭션 AOP도 전달받은 `UnexpectedRollbackException`을 클라이언트에 던짐

### **정리**

논리 트랜잭션 중 하나라도 롤백되면 전체 트랜잭션은 롤백됩니다.

내부 트랜잭션이 롤백 되었는데, 외부 트랜잭션이 커밋되면 `UnexpectedRollbackException` 예외가 발생합니다. 

`rollbackOnly` 상황에서 커밋이 발생하면 `UnexpectedRollbackException` 예외가 발생한다는 것과 같은 의미입니다.