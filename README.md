spring-data-mongodb-encrypt - @Field @Encrypted
-----------------------------------------------

Features
--------

- integrates transparently into `spring-data-mongo`
- supports sub-documents
- supports List, Map @Fields and nested beans
- high performance encryption
- high performance operation (no reflection at runtime)
- key versioning (to help migrating to new key without need to convert data)
- supports 256-bit AES out of the box
- supports any encryption available in Java (JCE)
- simple (500 lines of code)
- tested throughly

For the impatient
-----------------

Add dependency:

```xml
        <dependency>
            <groupId>com.bol</groupId>
            <artifactId>spring-data-mongodb-encrypt</artifactId>
            <version>1.0</version>
        </dependency>
```

Configure spring:

```java
@Configuration
public class MongoDBConfiguration extends AbstractMongoConfiguration {

    private static final byte[] secretKey = Base64.getDecoder().decode("hqHKBLV83LpCqzKpf8OvutbCs+O5wX5BPu3btWpEvXA=");

    @Override
    protected String getDatabaseName() {
        return "test";
    }

    @Override
    @Bean
    public Mongo mongo() throws Exception {
        return new MongoClient();
    }

    @Bean
    public CryptVault cryptVault() {
        return new CryptVault()
                .with256BitAesCbcPkcs5PaddingAnd128BitSaltKey(0, secretKey)
                .withDefaultKeyVersion(0);
    }

    @Bean
    public EncryptionEventListener encryptionEventListener(CryptVault cryptVault) {
        return new EncryptionEventListener(cryptVault);
    }
}
```

Example usage:

```java
@Document
public class MyBean {
    @Id
    public String id;

    // not encrypted
    @Field
    public String nonSensitiveData;

    // encrypted primitive types
    @Field
    @Encrypted
    public String secretString;

    @Field
    @Encrypted
    public Long secretLong;

    // encrypted sub-document (MySubBean is serialized, encrypted and stored as byte[])
    @Field
    @Encrypted
    public MySubBean secretSubBean;

    // encrypted collection (list is serialized, encrypted and stored as byte[])
    @Field
    @Encrypted
    public List<String> secretStringList;

    // values containing @Encrypted fields are encrypted
    @Field
    public MySubBean nonSensitiveSubBean;

    // values containing @Encrypted fields are encrypted
    @Field
    public List<MySubBean> nonSensitiveSubBeanList;

    // encrypted map (values containing @Encrypted fields are replaced by encrypted byte[])
    @Field
    public Map<String, MySubBean> publicMapWithSecretParts;
}

public class MySubBean {
    @Field
    public String nonSensitiveData;

    @Field
    @Encrypted
    public String secretString;
}
```

The result in mongodb:

```
> db.mybean.find().pretty()
{
	"_id" : ObjectId("59ea0fb902da8d61252b9988"),
	"_class" : "com.bol.secure.MyBean",
	"nonSensitiveSubBeanList" : [
		{
			"nonSensitiveData" : "sky is blue",
			"secretString" : BinData(0,"gJNJl3Eij5hX/dJeVgJ/eATIQqahYfUxg89wtKjZL1zxL5h4PTqGqjjn4HbBXbAibw==")
		},
		{
			"nonSensitiveData" : "grass is green",
			"secretString" : BinData(0,"gL+HVZ/OtbESNtL5yWgEYVv0rhT4gdOwYFs7zKx6WGEr1dq3jj84Sq+VhQKl4EthJg==")
		}
	]
}
```