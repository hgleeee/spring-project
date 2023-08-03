# @Transactional 주의할 점
## Inner Method에서의 동작
```java
public class BooksImpl implements Books {

        public void addBooks(List<String> bookNames) {
                bookNames.forEach(bookName -> this.addBook(bookName));
        }

        @Transactional
        public void addBook(String bookName) {
                Book book = new Book(bookName);
                bookRepository.save(book);
                book.setFlag(true);
        }
}
```

- 위 코드의 문제점은 addBook() 메소드에 @Transactional이 적용되지 않는다는 것이다.
  - 따라서 해당 코드를 실행해도 변경 감지가 동작하지 않아서 DB에는 저장된 book 정보의 Flag 컬럼이 정상적으로 업데이트되지 않는다.
- 프록시를 적용되면 클라이언트는 프록시를 타겟 객체로 생각하고 프록시 메소드를 호출하게 된다.
  - 프록시는 클라이언트로부터 요청을 받으면 타겟 객체의 메소드로 위임하고, 경우에 따라 부가 작업을 추가한다.
- Trasaction AOP에 의해 추가된 프록시라면, 타겟 객체 메소드 호출 전에 트랜잭션을 시작하고, 호출 후에 트랜잭션을 커밋하거나 롤백을 한다.
  - 즉, 프록시는 클라이언트가 타겟 객체를 호출하는 과정에만 동작하며, 타겟 객체의 메소드가 자기 자신의 다른 메소드를 호출할 때는 프록시가 동작하지 않는다.
- 위 예제에서 addBook() 메소드는 프록시로 감싸진 메소드가 아니므로 @Transactional 어노테이션이 동작하지 않게 된다.
 

## Private Method에서의 동작
- @Transactional 어노테이션을 붙이면, 트랜잭션 처리를 위해 빈 객체에 대한 프록시 객체를 생성한다.
  - 이 때, 프록시는 타겟 클래스를 상속하여 생성된다.
- 따라서 상속이 불가능한 private 메소드의 경우 @Transactional 어노테이션을 붙여도 트랜잭션이 동작하지 않는다.

## Spring TransactionTemplate
- DBMS 종류에 따라 DB Lock 지속 시간이나 Read Consistency의 차이, 그리고 이로 인한 서비스 동시성 문제 등을 생각하면 메소드 단위로 경계가 설정되는 AOP 방식의 트랜잭션이 비효율적일 수 있다.
- 예를 들어, 실행 시간이 상당한 메소드에 AOP로 트랜잭션을 붙였다고 생각해 보자.
  - 불필요하게 DB 커넥션을 점유하거나 DB Lock이 유지되는 시간이 길어질 수 있다.

```java
public class TransactionInvoker {

        private final A1Dao a1dao;
        private final A2Dao a2dao;

        @Transactional
        public void invoke() {
                // 매우 긴 Business Logic ...
                doInternalTransaction();
        }

        public void doInternalTransaction() {
                a1dao.insertA1();
                a2dao.insertA2();
        }
}
```

- 예를 들어, 위와 같은 상황에서는 비즈니스 로직이 트랜잭션에 포함되는 비효율이 발생할 수 있다.
- 이러한 경우에 개발자가 직접 트랜잭션의 경계를 설정할 필요가 있고, 이때 TransactionTemplate가 사용된다.

```java
public class TransactionInvoker {

        private final A1Dao a1dao;
        private final A2Dao a2dao;
        private final TransactionTemplate transactionTemplate;

        public void setTransactionManager(PlatformTransactionManager transactionManager){
                this.transactionTemplate = new TransactionTemplate(transactionManager);
                this.transactionTemplate.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        }

        public void invoke() throws Exception{
                // Business Logic ...
                doInternalTransaction();
        }

        private void doInternalTransaction() throws Exception{
                transactionTemplate.execute(new TransactionCallbackWithoutResult(){
                    public void doInTransactionWithoutResult(TransactionStatus status){
                            try{
                                    a1dao.insertA1();
                                    a2dao.insertA2(); 
                            }
                            catch(Exception e){
                                    status.setRollbackOnly();
                            }
                            return;
                    }
                });
        }
}
```

- Spring에서 setter를 통해 TransactionTemplate를 주입받는다.
- 그리고 TransactionTemplate을 생성 및 Trasaction 속성을 설정한다. (물론 생성자 주입을 선택해도 된다.)


## 질문
### @Transactional를 스프링 Bean의 메소드 A에 적용하였고, 해당 Bean의 메소드 B가 호출되었을 때, B 메소드 내부에서 A 메소드를 호출하면 어떤 요청 흐름이 발생하는지 설명하라.
- 프록시는 클라이언트가 타겟 객체를 호출하는 과정에만 동작하며, 타겟 객체의 메소드가 자기 자신의 다른 메소드를 호출할 때는 프록시가 동작하지 않는다.
- 즉 A, 메소드는 프록시로 감싸진 메소드가 아니므로 트랜잭션이 적용되지 않는 일반 코드가 수행된다.

### A라는 Service 객체의 메소드가 존재하고, 그 메소드 내부에서 로컬 트랜잭션 3개(다른 Service 객체의 트랜잭션 메소드를 호출했다는 의미)가 존재한다고 할 때, @Transactional을 A 메소드에 적용하면 어떤 요청 흐름이 발생하는지 설명하라.
- 트랜잭션 전파 수준에 따라 달라진다.
- 만약 기본 옵션인 REQUIRED를 가져간다면 로컬 트랜잭션 3개가 모두 부모 트랜잭션인 A에 합류하여 수행된다.
- 그래서 부모 트랜잭션이나 로컬 트랜잭션 3개나 모두 같은 트랜잭션이므로 어느 하나의 로직에서든 문제가 발생하면 전부 롤백이 된다.

