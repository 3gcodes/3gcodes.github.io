+++
date = '2025-01-09T14:11:31-06:00'
draft = false
title = 'Testing Spring Data JDBC Repositories with Test Containers'
+++

Recently, for a side project, I've gone through the motion of Spring Data JPA to JDBC and then found a middle-ground with Spring Data JDBC. I've come to loathe ORM's but I don't need to hand wire SQL statements into Row Mappers for basic CRUD operations. Spring Data JDBC seems to be as close to an ORM (it's essentially a Row Mapper) as I want to get. Plus, I can always revert down to JDBC with `JdbcTempate` if I want.

While I generally discourage writing tests that essentially test a popular framework (that should already be heavily tested), there will be some custom query methods in my repositories so I wanted to go ahead and wire up the testing infrastructure. 

Some things to note:
* I'm not going to go into detail on creating a Spring Boot app from scratch and adding all the bits. I'm going to assume you have a working app and want to add some tests using Testcontainers.
* I'm using UUID's for my @Id on my Entities. This requires some special care since Spring Data JDBC doesn't have generators like JPA does.
* I'm using Testcontainers with Postgres to mimic closely to what my app will use in production.

Let's start with basic code so we know what we're dealing with. Note that I will be omitting getters/setters for brevity, but you will need them. 

I have an `EntityBase` class that all my proper Entities extend.

```java
public class EntityBase<T> {

	@Id
	private UUID id;
	@CreatedDate
	private LocalDateTime createdAt;
	@LastModifiedDate
	private LocalDateTime updatedAt;
	private LocalDateTime deletedAt;
	private boolean deleted = false;
}
```

Now, I'll define my `OrganizationEntity` like so, again keeping things simple.

```java
@Table(name = "organization")
public class OrganizationEntity extends EntityBase<OrganizationEntity> {
	private String name;
}
```

I name all my entities with an `Entity` suffix because I use Java records as DTO's. If you name it `Organization`, there's no need for the `@Table` annotation.

Here is the repository.

```java
@Repository
public interface OrganizationRepository extends CrudRepository<OrganizationEntity, UUID> {
    Optional<OrganizationEntity> findByIdAndDeletedIsFalse(UUID id);
}
```

Because I have soft deletes, I needed to create a `findById` method that only finds non-deleted records.

The next thing we need is a way for the `OrganizationEntity` to generate a UUID on an insert. We need to create a new component.

```java
@Component
public class UUIDGenCallback implements BeforeConvertCallback<EntityBase> {

	@Override
	public EntityBase onBeforeConvert(EntityBase entity) {
	    if (entity.getId() == null) {
		entity.setId(UUID.randomUUID());
	    }
	    return entity;
	}
}
```

I'm using the `EntityBase` class because that's what contains our `@Id`.

Foundation-ally, if you have services and controllers and you inject this repository,  and assuming you have a working connection to the database, all should be fine. Again, there are hundreds of videos and tutorials on the web to get you that far. I don't want to re-hash it here.

In order to work with Testcontainers and Postgres, you'll need some additions to your dependencies. I'm using maven, so adjust if you're using Gradle on your own.

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-testcontainers</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>postgresql</artifactId>
  <scope>test</scope>
</dependency>
```

On to testing. First thing, if you have any specific properties you need for testing, make sure you create a `application-test.yaml` (or `.properties` if that's your thing) and include the there. Here's mine, which is very minimal. I just needed to make sure Flyway was enabled; only necessary if you're using Flyway. If you're not using a migration library, you'll need a way to make sure your schema gets created before your tests run. That sounds like some good homework for you. 

```yaml
spring:
  application:
    name: myapp-web
  flyway:
    enabled: true
```

Because I don't want to repeat a bunch of annotations/config I always have a base class for different test types. Here's the `JdbcTestsBase`.

```java
@Testcontainers
@DataJdbcTest
@ActiveProfiles("test")
@ComponentScan("com.some.package")
@Import(UUIDGenCallback.class)
public class JdbcTestsBase {
}
```

Note that `@ComponentScan("com.some.package")` may not be necessary. I needed it because of where my `UUIDGenCallback` lives. It's also why the `@Import` exists. Because `@DataJdbcTests` is considered a slice, the entire Spring app context doesn't get wired up. You'll see this in the test class as well because I need to component scan the repository class.

```java
@ComponentScan("com.some.package.organization.repository")
class OrganizationRepositoryTests extends JdbcTestsBase {

	@Container
	@ServiceConnection
	static final PostgreSQLContainer<?> POSTGRESQL_CONTAINER = new PostgreSQLContainer<>("postgres:16-alpine");

	@Autowired
	private OrganizationRepository repository;

	@Test
	void givenCreateOrganization_thenSuccess() {
		final var org = repository.save(new OrganizationEntity().withName("Test Org 001"));
		assertThat(org, notNullValue());
		assertThat(org.getId(), notNullValue());
	}

	@Test
	void givenFindByIdAndDeletedIsFalse_thenReturnOrganization() {
		final var org = repository.save(new OrganizationEntity().withName("Test Org 001"));
		assertThat(org, notNullValue());
		final var maybeOrg = repository.findByIdAndDeletedIsFalse(org.getId());
		
		assertThat(maybeOrg.isPresent(), equalTo(true));
		assertThat(maybeOrg.get().getName(), equalTo("Test Org 001"));
	}
}
```

What Testcontainers does here is download an image, `postgres:16-alpine`, and wires up the Datasource to connect to it automatically. There's a lot of ways to customize this with your own configuration. I'll link to all the docs below.

If you want to ask me any questions, best way is to reach out to me on X.

https://x.com/greggbolinger

## Testcontainers Links

https://docs.spring.io/spring-boot/reference/testing/testcontainers.html

https://testcontainers.com/guides/testing-spring-boot-rest-api-using-testcontainers/