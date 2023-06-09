# 인프런 스프링 DB 2편 - 데이터 접근 활용 기술

공부 및 실습 정리

[![Velog's GitHub stats](https://velog-readme-stats.vercel.app/api?name=2jooin1207&slug=스프링-db-1)](https://velog.io/@2jooin1207/%EC%8A%A4%ED%94%84%EB%A7%81-db-1)

# JDBC

> 자바에서 데이터베이스에 접속할 수 있게 하는 자바 API

애플리케이션 서버와 db는 다음과 같은 과정을 거친다.

1. 커넥션 연결
2. SQL 전달
3. 결과응답

그러나 db마다 커넥션을 연결하는 법, SQL을 전달받는 방법 등이 다르다.
이것을 해결하기위해 등장한 것이 jdbc이다.

jbdc는 자바 개발자라면 필수적으로 알아두어야 하는 기술이다.

## 기능

다음 3가지 기능을 표준 인터페이스로 정의하여 제공한다.
- java.sql.Connection -> 연결
- java.sql.Statement -> SQL을 담은 내용
- java.sql.ResultSet -> SQL 요청 응답

JDBC로 인해 데이터베이스를 변경하더라도 애플리케이션 서버의 사용 코드를 그대로 사용할수 있으며, db마다 다른 접속 방법을 공부할 필요가 없어졌다.

## 예시

### DBConnectionutil

~~~
@Slf4j
public class DBConnectionUtil {

    public static Connection getConnection() {
        try {
        	//DriverManager.getConnection() <- JDBC가 제공하는 커넥션 연결
            Connection connection = DriverManager.getConnection(ConnectionConst.URL, ConnectionConst.USERNAME, ConnectionConst.PASSWORD);

            log.info("get Connection={}, class={}", connection, connection.getClass());
            return connection;
        } catch (SQLException e) {
            throw new IllegalStateException(e);
        }
    }
}


~~~

DriverManager는 라이브러리에 등록된 DB 드라이버들을 관리하고, 커넥션을 획득하는 기능을 제공.

하는 일들
- 라이브러리에 등록된 드라이버들을 자동으로 인식
- 드라이버에 URL, 이름, PASSWORD 등 접속에 필요한 추가 정보를 전달하여 커넥션을 획득할 수 있는 지 확인
- 이때 드라이버는 URL정보를 확인하여 본인이 처리할 수 있는 요청인지 확인 후, 처리할 수 있다면 커넥션 구현체를 클라이언트에게 반환

con.prepareStatement(sql)
- 데이터베이스에 전달할 SQL과 파라미터로 전달할 데이터들을 준비한다
- ?를 통한 파라미터 바인딩을 가능하게 해준다

pstmt.executeUpdate()
- sql을 커넥션을 통해 실제 db로 전달
- int를 반환하는데 영향받은 DB row의 수를 반환
- 데이터 등록, 수정, 삭제시 .executeUpdate()를 사용

리소스 정리
- 쿼리를 실행한 후에는 리소스를 '역순으로' 정리해주어야 한다.
- 정리하지 않을 경우 커넥션이 끊어지지 않고 계속 유지되는 문제가 발생할 수 있음
- 따라서 finaly를 이용한 리소스 정리는 필수

ResultSet
- select 쿼리의 결과가 순서대로 들어간다.

# 커넥션 폴과 데이터 소스

> 커넥션을 관리하는 폴

- 커넥션을 새로 만드는 것은 상당히 복잡하며, 시간도 오래 걸린다.
- 따라서 미리 커넥션을 만들어 둔후, 필요시 연결한다.
- 사용다한 커넥션은 종료하는 것이 아닌 살아있는 채로 반환한다.

스프링에서는 HikariCP를 기본 커넥션풀로 사용한다.

javax.sql.DataSource
- 커넥션을 획득하는 방법을 추상화한 인터페이스
- 핵심기능으로 '커넥션 조회'가 있다.
- 개발시 DataSource 인터페이스에 의존하도록 로직을 구성해야 한다.

~~~
public interface DataSource {
 Connection getConnection() throws SQLException;
}
~~~

## DataSource를 사용한 커넥션 예제

~~~
@Slf4j
public class ConnectionTest {

	//드라이브 매니저를 통한 커넥션 연결
    @Test
    void driverManager() throws SQLException {
        Connection con1 = DriverManager.getConnection(URL, USERNAME, PASSWORD);
        Connection con2 = DriverManager.getConnection(URL, USERNAME, PASSWORD);

        log.info("connection={}, class={} ", con1, con1.getClass());
        log.info("connection={}, class={} ", con2, con2.getClass());
    }

	//데이터 소스를 사용한 커넥션 연결
    @Test
    void dataSourceDriverManager() throws SQLException {
    	//항상 새로운 커넥션을 획득
        //스프링이 제공하는 코드
        DataSource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD);
        useDataSource(dataSource);
    }

    @Test
    void dataSourceConnectionPool() throws SQLException, InterruptedException {
        //HikariCp를 사용한 커넥션 풀링
        //스프링에서 자동으로 제공하는 커넥션 풀
        HikariDataSource dataSource = new HikariDataSource();

        dataSource.setJdbcUrl(URL);
        dataSource.setUsername(USERNAME);
        dataSource.setPassword(PASSWORD);
        dataSource.setMaximumPoolSize(10);  //커넥션 풀 사이즈 크기: 10
        dataSource.setPoolName("MyPool");	//커넥션 이름: Mypool

        useDataSource(dataSource);
        Thread.sleep(1000);
    }

    private void useDataSource(DataSource dataSource) throws SQLException{
        Connection con1 = DriverManager.getConnection(URL, USERNAME, PASSWORD);
        Connection con2 = DriverManager.getConnection(URL, USERNAME, PASSWORD);

        log.info("connection={}, class={} ", con1, con1.getClass());
        log.info("connection={}, class={} ", con2, con2.getClass());
    }
}
~~~

DriverManager
- 커넥션을 획득할때마다 URL, USERNAME, PASSWORD 등의 파라미터를 꼐속 전달해야 한다.


DataSource
- 처음 객체를 생성할 때만 필요한 파라미터를 넘긴다.
- 커넥션을 획득시 단순히 dataSource.getConnection()만 호출.
- 필요한 데이터를 dataSource가 생성되는 시점에 모두 넣어두게 되면, DataSource만 주입받아 getconnection()만 호출하면 되므로 편하다.

## DaatSource를 적용한 코드

~~~
@Slf4j
public class MemberRepositoryV1 {

    //dataSource 의존관계 주입
    private final DataSource dataSource;

    public MemberRepositoryV1(DataSource dataSource) {
        this.dataSource = dataSource;
    }


    public Member save(Member member) throws SQLException {

        String sql = "insert into member(member_id, money) values(?, ?)";
        Connection con = null;
        PreparedStatement pstmt = null;
        try {
            con = getConnection();
            pstmt = con.prepareStatement(sql);
            pstmt.setString(1, member.getMemberId());
            pstmt.setInt(2, member.getMoney());
            pstmt.executeUpdate();
            return member;
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(con, pstmt, null);
        }
    }
    ...
    
    }
~~~

- DataSource는 표준 인터페이스이므로 DriverManagerDataSource에서 HikariDataSource로 변경되어도 해당 코드의 변경이 필요없다.
- JdbcUtils를 사용하면 커넥션을 좀 더 편리하게 닫을 수 있다.

# 트랙젝션

> 하나의 거래를 안전하게 처리할 수 있도록 보장해주는 것

1)A의 계좌를 5000원 만큼 감소 시키고, 2)B의 계좌를 5000원 만큼 증가시킬때
2번 작업에서 에러가 날경우 A의 계좌만 5000원 만큼 감소하는 일이 일어날 수 있다.

1, 2 작업 모두 성공해야 저장되며 하나라도 실패할 경우 롤백시키는 기능이 바로 트랜젝션이다.

커밋(commit)
- 모든 작업이 성공해서 DB에 정상 반영하는 것
롤백(RoolBack)
- 작업중 실패해서 거래 이전으로 데이터를 되돌리는 것

## 트랜젝션의 ACID

다음과 같은 항목을 보장해야 한다

1) 원자성
- 트랜젝션의 모든 작업은 모두 성공하거나 모두 실패해야 한다

2) 일관성
- 모든 ㅌ트랜젝션은 일관성 있는 데이터베이스 상태를 유지해야 한다.

3) 격리성
- 동시에 실행되는 트랜젝션이 서로에게 영향을 미치지 못해게 격리해야 한다.
- 다만 격리성을 완전히 보장하려면 동시처리성능이 나빠지므로, 격리성의 수준을 4단계로 나계어 정의한다.

4)지속성
- 트랜젝션이 성공적으로 끝날경우 그 결과가 항상 기록되어야 하며, 문제가 생겨도 로그 등을 통해 그 내용을 복구해야 한다.


## 서버 커넥션과 DB세션

1. 클라이언트는 DB 서버에 연결을 요청한 후 커넥션을 맺는다.
2. DB서버는 내부에 세션을 생성한다.
3. 이후 해당 커넥션을 통한 모든 요청은 이 세션을 통해 실행된다.

-> 클라이언트를 통해 SQL을 전달하면 현재 커넥션에 연결된 세션이 SQL을 실행된다.

## 트랜젝션 예제

기본 데이터 셋팅
~~~
set autocommit true;
delete from member;
insert into member(member_id, money) values ('memberA',10000);
insert into member(member_id, money) values ('memberB',10000);
~~~

계좌이체 실행
~~~
set autocommit false;	//수동 커밋으로 변경
//A의 계좌를 2000만큼 감소
update member set money=10000 - 2000 where member_id = 'memberA'; 
//B의 계좌를 2000만큼 증가
update member set money=10000 + 2000 where member_id = 'memberB'; 
~~~

결과
~~~
//정상 실행시
Commit();
//오류 발생시
RollBack();
~~~

### DB락이란

세션 1이 트랜젝션을 시작하고 데이터를 수정하는 동안 아직 커밋이 되어있지 않았는데도 세션 2에서 동시에 같은 데이터를 수정하게 되면 트랜잭션의 원자성이 깨지게 된다.

따라서 한 세션에서 데이터를 수정하는 경우 다른 세션에서 수정하지 못하게 막아야한다.

>DB락을 획득한 세션만이 데이터를 수정할 수 있다.

락 타임 아웃
- SET LOCK_TIMEOUT 60000 : 락 획득 시간을 60초로 설정한다. 60초 안에 락을 얻지 못하면 예외가 발생한다.

조회락
- ~select for update
- 해당데이터를 다른 곳에서 변경하지 못하도록 강제로 막아야 할때 사용

## 트랜잭션의 추상화

애플리케이션의 구조는 3가지 계층으로 나누어 진다.

프레젠테이션 계층
- UI와 관련된 처리 담당

서비스 계층
- 비지니스 로직 담당
- 가급적 순수 자바코드로 작성

데이터 접근 계층
- 실제 데이터베이스에 접근

따라서 서비스 계층에 트랜젝션과 비지니스 로직을 함께 작성할 경우 유지보수가 쉽지 않다. 

이런 경우 트랜젝션 추상화를 사용한다.
- 대부분의 데이터 접근 기술을 만들어두어 코드의 수정을 최소화 시킨다.

### 트랜젝션 동기화 매니저와 AOP

트랜젝션 동기화 매니저 - 커넥션을 안전하게 동기화시키는 방법

1. 트랜젝션 매니저는 커넥션을 만든 후 트랜젝션을 시작한다
2. 트랜젝션이 시작된 커넥션은 트랜젝션 동기화 매니저에 보관한다.
3. 리포지토리는 트랜젝션 동기화 매니저에 보관된 커넥션을 꺼내 사용한다.
4. 트랜젝션이 종료되면 커넥션을 종료후 닫는다.

탬플릿 콜백 패턴
- 트랜잭션을 시작하고, 비즈니스 로직을 실행하고, 성공하면 커밋하고, 예외가 발생해서 실패하면 롤백한다.
-트랜잭션 템플릿을 사용하여 try - catch문을 생략할 수 있다.


트랜잭션 템플릿의 기본 동작은 다음과 같다.
- 비즈니스 로직이 정상 수행되면 커밋한다.
- 언체크 예외가 발생하면 롤백한다. 그 외의 경우 커밋한다.

</br>


![](https://velog.velcdn.com/images/2jooin1207/post/49e92358-19c1-4f62-8441-0759127702f5/image.PNG)


트랜젝션 AOP

> @Transactional를 사용하면 스프링이 AOP를 사용해서 트랜젝션을 편리하게 처리해준다.

트랜젝션 프록시는 트랜젝션 처리 로직을 모두 가져간 후, 트랜젝션을 시작한 후 실제 서비스를 대신 호출한다.
-> 서비스 계층에 순수한 비지니스 코드만 남길 수 있다.

필요한 클래스/메소드에 @Transactional 어노테이션을 붙여 사용한다.


~~~
//MemberServiceV3_3

@Slf4j
public class MemberServiceV3_3 {
    private final MemberRepositoryV3 memberRepository;

    public MemberServiceV3_3(MemberRepositoryV3 memberRepository) {
        this.memberRepository = memberRepository;
    }

    //트랜잭션 어노테이션
    @Transactional
    public void accountTransfer(String fromId, String toId, int money) throws SQLException {
        //비즈니스 로직
        bizLogic(fromId, toId, money);
    }

    private void bizLogic(String fromId, String toId, int money) throws
            SQLException {
        Member fromMember = memberRepository.findById(fromId);
        Member toMember = memberRepository.findById(toId);
        memberRepository.update(fromId, fromMember.getMoney() - money);
        validation(toMember);
        memberRepository.update(toId, toMember.getMoney() + money);
    }

    private void validation(Member toMember) {
        if (toMember.getMemberId().equals("ex")) {
            throw new IllegalStateException("이체중 예외 발생");
        }
    }
}
~~~

선언적 트랜잭션 관리
- @Transactional을 선언하여 편리하게 트랜젝션을 적용하는 것.
- 필요한 것에 추가하면 나머지는 스프링 트랜젝션 AOP가 자동으로 처리해준다.

</br>

테스트 코드
~~~
@Slf4j
@SpringBootTest	//스프링 AOP를 적용하기 위해 스프링 컨테이너가 필요하다
class MemberServiceV3Test {
    public static final String MEMBER_A = "memberA";
    public static final String MEMBER_B = "memberB";
    public static final String MEMBER_EX = "ex";

    @Autowired
    private MemberRepositoryV3 memberRepository;
    @Autowired
    private MemberServiceV3_3 memberService;

	//필요한 빈을 등록할 수 있다.
    @TestConfiguration
    static class TEstConfig {
        private final DataSource dataSource;
        public TEstConfig(DataSource dataSource) {
            this.dataSource = dataSource;
        }
        @Bean
        MemberRepositoryV3 memberRepositoryV3(){
            return new MemberRepositoryV3(dataSource);
        }
        @Bean
        MemberServiceV3_3 memberServiceV3_3(){
            return new MemberServiceV3_3(memberRepositoryV3());
        }
    }

    @AfterEach
    void after() throws SQLException {
        memberRepository.delete(MEMBER_A);
        memberRepository.delete(MEMBER_B);
        memberRepository.delete(MEMBER_EX);
    }

    @Test
    @DisplayName("정상 이체")
    void accountTransfer() throws SQLException {
        //given
        Member memberA = new Member(MEMBER_A, 10000);
        Member memberB = new Member(MEMBER_B, 10000);
        memberRepository.save(memberA);
        memberRepository.save(memberB);
        //when
        memberService.accountTransfer(memberA.getMemberId(), memberB.getMemberId(), 2000);
        //then
        Member findMemberA = memberRepository.findById(memberA.getMemberId());
        Member findMemberB = memberRepository.findById(memberB.getMemberId());
        assertThat(findMemberA.getMoney()).isEqualTo(8000);
        assertThat(findMemberB.getMoney()).isEqualTo(12000);
    }

    @Test
    @DisplayName("이체중 예외 발생")
    void accountTransferEx() throws SQLException {
        //given
        Member memberA = new Member(MEMBER_A, 10000);
        Member memberEx = new Member(MEMBER_EX, 10000);
        memberRepository.save(memberA);
        memberRepository.save(memberEx);
        //when
        assertThatThrownBy(() ->
                memberService.accountTransfer(memberA.getMemberId(), memberEx.getMemberId(),
                        2000))
                .isInstanceOf(IllegalStateException.class);
        //then
        Member findMemberA = memberRepository.findById(memberA.getMemberId());
        Member findMemberEx =
                memberRepository.findById(memberEx.getMemberId());
        //memberA의 돈이 롤백 되어야함
        assertThat(findMemberA.getMoney()).isEqualTo(10000);
        assertThat(findMemberEx.getMoney()).isEqualTo(10000);
    }
}
~~~

스프링 부트가 자동으로 데이터소스와 트랜젝션 매니저를 자동으로 등록해준다.
- 데이터소스는 dataSource라는 이름으로 스프링 빈을 등록해준다.
- application.properties에 있는 속성을 사용해서 DataSource를 생성한다.
다음과 같이 설정한다.
~~~
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa
spring.datasource.password=
~~~
- 스프링부트는 HikariDataSource를 기본으로 사용한다.



# 마이바티스 기초 설정

grade에 추가
~~~
//마이바티스 추가
	implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:2.2.0'
~~~

application.properties에 추가
~~~
#패키지 위치 생략 가능
mybatis.type-aliases-package=hello.itemservice.domain

#객체 이름을 낙타표시법으로 바꿔줌
mybatis.configuration.map-underscore-to-camel-case=true
logging.level.hello.itemservice.repository.mybatis=trace
~~~

# ItemMapper와 XML 파일

## ItemMapper 등록
~~~
@Mapper
public interface ItemMapper {
    void save(Item item);

    //파라미터가 두개인 경우 @Param을 사용해 주어야함
    void update(@Param("id") Long id, @Param("updateParam")ItemUpdateDto updateParam);

    Optional<Item> findById(Long id);

    List<Item> findAll(ItemSearchCond itemSearch);
}
~~~

- @Mapper 어노테이션이 있어야 마이바티스에서 인식을 할 수 있다.

- 이 인터페이스의 메소드를 호출하면, xml의 해당 SQL을 실행한후 결과값을 반환한다.
따라서 인터페이스 메소드를 만들때 리턴 값과 메소드타입을 맞춰주어야 한다.

## ItemMapper.xml

~~~
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!--ItemMapper의 위치를 적어줌-->
<mapper namespace="hello.itemservice.repository.mybatis.ItemMapper">

    <!--useGeneratedKeys는 자동으로 id값을 설정해주는 역활-->
    <insert id="save" useGeneratedKeys="true" keyProperty="id">
        <!--아이템 객체의 속성 값들을 넣어줌-->
        insert into item(item_name, price, quantity)
        values (#{itemName}, #{price}, #{quantity})
    </insert>

    <update id="update">
        update item
        set item_name=#{updateParam.itemName},
            price=#{updateParam.price},
            quantity=#{updateParam.quantity}
        where id = #{id}
    </update>

    <!--resultType: 조회하는 item의 경로를 적어주어야 하나, application.properties에 경로를 넣어준만큼 생략가능-->
    <select id="findById" resultType="Item">
        select id, item_name, price, quantity
        from item
        where id = #{id}
    </select>

    <select id="findAll" resultType="Item">

        select id, item_name, price, quantity
        from item
        <where>
            <if test="itemName != null and itemName != ''">
                and item_name like concat('%', #{itemName}, '%')
            </if>
            <if test="maxPrice != null">
                and price &lt;= #{maxprice}
            </if>

        </where>
    </select>

</mapper>
~~~
- mapper namespace="매퍼 인터페이스의 경로"

- sql의 'id'와 mapper 인터페이스의 메소드 이름이 같아야 한다.

- useGeneratedKeys="true" keyProperty="id" 같은 경우, id의 값을 자동으로 설정해준다.

- resultType="Item" 같은 경우 반환 타입을 적어주는 역활. 

- 원래 select문을 쓸때 Item 객체의 패키지 위치를 다 적어주어야 하나, application.properties에 경로를 적어준 만큼 생략이 가능하다.

- xml 파일의 위치는 Mapper 인터페이스가 있는 폴더의 위치와 동일하게 맞춰 주어야한다. 
xml 파일을 원하는 위치에 두고 싶으면 application.properties 에 다음과 같이 설정하면 된다.
> mybatis.mapper-locations=classpath:mapper/	&#42;	&#42;/	&#42;.xml

- 이러면 resources/mapper 를 포함한 그 하위 폴더에 있는 xml을 xml 매핑 파일로 인식하며, 파일 이름도 자유롭게 지정해도 된다.
- 마이바이트에서 동적쿼리는 <!where>, <!if>등의 동적쿼리 문법을 통해 구현한다.

# 기타 설정들

## MybatisItemRepository

~~~
@Repository
@RequiredArgsConstructor
public class MybatisItemRepository implements ItemRepository {

    //mapper의 의존관계 구현체를 주입
    private final ItemMapper itemMapper;

    @Override
    public Item save(Item item) {
        itemMapper.save(item);
        return item;
    }

    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        itemMapper.update(itemId, updateParam);
    }

    @Override
    public Optional<Item> findById(Long id) {
        return itemMapper.findById(id);
    }

    @Override
    public List<Item> findAll(ItemSearchCond itemSearch) {
        return itemMapper.findAll(itemSearch);
    }
}
~~~

- MybatisItemRepository는 ItemMapper에 기능을 위임한다

## MyBatisConfig

~~~
@Configuration
@RequiredArgsConstructor
public class MyBatisConfig {
    private final ItemMapper itemMapper;

    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }
	//itemMapper를 주입받은 후, 필요한 의존 관계를 만든다.
    @Bean
    public ItemRepository itemRepository() {
        return new MybatisItemRepository(itemMapper);
    }
}
~~~

## ItemServiceApplication

~~~
@Import(MyBatisConfig.class)	//메모리 컨픽을 임포트
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")
public class ItemServiceApplication {

	public static void main(String[] args) {
		SpringApplication.run(ItemServiceApplication.class, args);
	}

	@Bean
	@Profile("local")	//운영환경에 따라 설정정보를 변환하기 위해 등록한 프로필
	public TestDataInit testDataInit(ItemRepository itemRepository) {
		return new TestDataInit(itemRepository);
	}
}
~~~

- @Import(MyBatisConfig.class)	//메모리 컨픽을 임포트

# 마이바티스 분석

![](https://velog.velcdn.com/images/2jooin1207/post/f6cec079-e520-41fd-8d93-27cc1a945cbc/image.PNG)


1. 애플리케이션 로딩 시점에 MyBatis 스프링 연동 모듈은 @Mapper 가 붙어있는 인터페이스를 조사한다.
2. 해당 인터페이스가 발견되면 동적 프록시 기술을 사용해서 ItemMapper 인터페이스의 구현체를 만든다.
3. 생성된 구현체를 스프링 빈으로 등록한다.

-> 따라서 ItemMapper 매퍼 인터페이스의 구현체가 없어도 xml을 편리하게 호출하고 구현할 수 있다.
