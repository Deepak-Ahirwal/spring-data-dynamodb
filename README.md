[![codecov.io](https://codecov.io/github/derjust/spring-data-dynamodb/coverage.svg?branch=master)](https://codecov.io/github/derjust/spring-data-dynamodb?branch=master) [![Build Status](https://travis-ci.org/derjust/spring-data-dynamodb.svg?branch=master)](https://travis-ci.org/derjust/spring-data-dynamodb) 
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.github.derjust/spring-data-dynamodb/badge.svg)](https://search.maven.org/search?q=g:com.github.derjust)
[![Developer Workspace](https://codenvy.io/factory/resources/codenvy-contribute.svg)](https://codenvy.io/f?user=userrzonfqnofgrmfxjx&amp;name=spring-data-dynamodb)
[![Donation badge](https://img.shields.io/badge/Donate-%F0%9F%92%B8-DAA520.svg)](DONATION.md)



# Spring  Data DynamoDB #

<img align="left" src="https://derjust.github.io/spring-data-dynamodb/banner/spring-data-dynamodb.png" />

The primary goal of the [Spring® Data](https://projects.spring.io/spring-data/) project is to make it easier to build Spring-powered applications that use data access technologies.

This module deals with enhanced support for a data access layer built on [AWS DynamoDB](https://aws.amazon.com/dynamodb/).

Technical infos can be found on the [project page](https://derjust.github.io/spring-data-dynamodb/).

## Supported Features ##

* Implementation of [CRUD methods for](https://docs.spring.io/spring-data/commons/docs/current/reference/html/#repositories.definition) DynamoDB Entities
* Dynamic query generation from [query method names](https://docs.spring.io/spring-data/commons/docs/current/reference/html/#repositories.query-methods.query-creation) ([Supported keywords and comparison operators](https://github.com/derjust/spring-data-dynamodb/wiki/Supported-Spring-Data-Comparison-Operators))
* [Projections](https://github.com/derjust/spring-data-dynamodb/wiki/Projections)
* Possibility to integrate [custom repository code](https://github.com/derjust/spring-data-dynamodb/wiki/Custom-repository-implementations)
* Easy Spring annotation based integration
* [REST support](https://github.com/derjust/spring-data-dynamodb-examples/blob/master/README-rest.md) via [spring-data-rest](https://projects.spring.io/spring-data-rest/)

## Demo application ##

For a demo of spring-data-dynamodb, using spring-data-rest to showcase DynamoDB repositories exposed with REST,
please see [spring-data-dynamodb-examples](https://github.com/derjust/spring-data-dynamodb-examples).

## Quick Start ##

Download the JAR though [Maven Central](http://mvnrepository.com/artifact/com.github.derjust/spring-data-dynamodb) ([`SNAPSHOT` builds](https://oss.sonatype.org/content/repositories/snapshots/com/github/derjust/spring-data-dynamodb/) are available via the [OSSRH snapshot repository](https://stackoverflow.com/a/7717234/25332) ):

```xml
<dependency>
  <groupId>com.github.derjust</groupId>
  <artifactId>spring-data-dynamodb</artifactId>
  <version>5.0.3</version>
</dependency>
```

Setup DynamoDB configuration as well as enabling Spring-Data DynamoDB repository support via Annotation ([XML-based configuration](wiki/Quick-Start---XML-based-configuration))

```java
@Configuration
@EnableDynamoDBRepositories(basePackages = "com.acme.repositories")
public class DynamoDBConfig {

	@Value("${amazon.dynamodb.endpoint}")
	private String amazonDynamoDBEndpoint;

	@Value("${amazon.aws.accesskey}")
	private String amazonAWSAccessKey;

	@Value("${amazon.aws.secretkey}")
	private String amazonAWSSecretKey;

	@Bean
	public AmazonDynamoDB amazonDynamoDB(AWSCredentials amazonAWSCredentials) {
		AmazonDynamoDB amazonDynamoDB = new AmazonDynamoDBClient(amazonAWSCredentials);

		if (StringUtils.isNotEmpty(amazonDynamoDBEndpoint)) {
			amazonDynamoDB.setEndpoint(amazonDynamoDBEndpoint);
		}
		return amazonDynamoDB;
	}

	@Bean
	public AWSCredentials amazonAWSCredentials() {
	    // Or use an AWSCredentialsProvider/AWSCredentialsProviderChain
		return new BasicAWSCredentials(amazonAWSAccessKey, amazonAWSSecretKey);
	}

}
```

Create a DynamoDB entity for this table:

```java
@DynamoDBTable(tableName = "User")
public class User {

  private String id;
  private String firstName;
  private String lastName;

  @DynamoDBHashKey
  @DynamoDBAutoGeneratedKey 
  public String getId() {
	return id;
  }

  @DynamoDBAttribute
  public String getFirstName() {
	return firstName;
  }

  @DynamoDBAttribute
  public String getLastName() {
	return lastName;
  }
       
  public void setId(String id) {
    this.id = id;
  }
  
  public void setFirstName(String firstName) {
    this.firstName = firstName;
  }

  public void setLastName(String lastName) {
    this.lastName = lastName;
  }
}
```

Create a CRUD repository interface in `com.acme.repositories`:

```java
package com.acme.repositories;

@EnableScan
public interface UserRepository extends CrudRepository<User, String> {
  List<User> findByLastName(String lastName);
}
```

or for paging and sorting...

```java
package com.acme.repositories;

public interface UserRepository extends PagingAndSortingRepository<User, String> {
  Page<User> findByLastName(String lastName,Pageable pageable);
  
  @EnableScan 
  @EnableScanCount
  Page<User> findAll(Pageable pageable);
}
```

And finally write a test client

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = { 
    PropertyPlaceholderAutoConfiguration.class, DynamoDBConfig.class})
    public class UserRepositoryIntegrationTest {
       
    @Autowired
    UserRepository repository;
        
    @Test
    public void sampleTestCase() {
        User dave = new User("Dave", "Matthews");
        repository.save(dave);
    
        User carter = new User("Carter", "Beauford");
        repository.save(carter);
    
        List<User> result = repository.findByLastName("Matthews");
        Assert.assertThat(result.size(), is(1));
        Assert.assertThat(result, hasItem(dave));
    }
    
    private static final long CAPACITY = 5L;
    
    @Autowired
    private AmazonDynamoDB amazonDynamoDB;
    
    @Before
    public void init() throws Exception {
        // Delete User table in case it exists
        amazonDynamoDB.listTables().getTableNames().stream().
                filter(tableName -> tableName.equals(User.TABLE_NAME)).forEach(tableName -> {
            amazonDynamoDB.deleteTable(tableName);
        });
	
	//Create User table
        amazonDynamoDB.createTable(new DynamoDBMapper(amazonDynamoDB)
            .generateCreateTableRequest(User.class)
	    .withProvisionedThroughput(new ProvisionedThroughput(CAPACITY, CAPACITY)));
    }
    
}
```


## More
More sample code can be found in the [spring-data-dynamodb-examples](https://github.com/derjust/spring-data-dynamodb-examples) project.

Advanced topics can be found in the [wiki](https://github.com/derjust/spring-data-dynamodb/wiki).


## Version & Spring Framework compatibility ##

The major and minor number of this library refers to the compatible Spring framework version. The build number is used as specified by SEMVER.

API changes will follow SEMVER and loosly the Spring Framework releases.

| `spring-data-dynamodb` version  | Spring Boot compatibility      |Spring Framework compatibility  | Spring Data compatibility |
| ------------------------------- | ------------------------------ | ------------------------------ | ------------------------- |
| 1.0.x                           |                                | >= 3.1 && < 4.2                |                           |
| 4.2.x                           | >= 1.3.0 && < 1.4.0            | >= 4.2 && < 4.3                | Gosling-SR1               |
| 4.3.x                           | >= 1.4.0 < 2.0                 | >= 4.3 && < 5.0                | Gosling-SR1               |
| 4.4.x                           | >= 1.4.0 < 2.0                 | >= 4.3 && < 5.0                | Hopper-SR2                |
| 4.5.x                           | >= 1.4.0 < 2.0                 | >= 4.3 && < 5.0                | Ingalls                   |
| 5.0.x                           | >= 2.0                         | >= 5.0                         | Kay-SR1                   |

`spring-data-dynamodb` depends directly on `spring-data` as also `spring-context`, `spring-data` and `spring-tx`.

`compile` and `runtime` dependencies are kept to a minimum to allow easy integartion, for example into 
Spring-Boot projects.

## History
The code base has some history already in it - let's clarify it a bit:
* The code base was established under [github.com/michaellavelle/spring-data-dynamodb)](https://github.com/michaellavelle/spring-data-dynamodb)
* It was forked and further maintained under [github.com/derjust/spring-data-dynamodb)](https://github.com/derjust/spring-data-dynamodb) 
    * Available in Maven Central under [`com.github.derjust:spring-data-dynamodb`](http://central.maven.org/maven2/com/github/derjust/spring-data-dynamodb/)

The Java package name/XSD namespace never changed from `org.socialsignin.spring.data.dynamodb`.
But the XSD is now also available at [`https://derjust.github.io/spring-data-dynamodb/spring-dynamodb-1.0.xsd`](https://derjust.github.io/spring-data-dynamodb/spring-dynamodb-1.0.xsd).
