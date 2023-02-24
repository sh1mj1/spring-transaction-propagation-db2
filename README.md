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