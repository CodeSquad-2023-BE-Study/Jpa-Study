# 컨버터 `@Converter`

- 컨버터를 사용해서 엔티티의 데이터를 변환해서 데이터베이스에 저장합니다.
- JPA 3.0에서 도입되어 사용할 수 있습니다.

- `JpaAttributeConverter` 의 구현체 모습입니다. 관계형 값을 도메인 값으로 변환해주고 있습니다.

```java
package org.hibernate.metamodel.model.convert.internal;

public class JpaAttributeConverterImpl<O,R> implements JpaAttributeConverter<O,R> {
	private final ManagedBean<AttributeConverter<O,R>> attributeConverterBean;
	// ..

	@Override
	public O toDomainValue(R relationalForm) { // DB -> Object
		return attributeConverterBean.getBeanInstance().convertToEntityAttribute( relationalForm );
	}

	@Override
	public R toRelationalValue(O domainForm) { // Object -> DB
		return attributeConverterBean.getBeanInstance().convertToDatabaseColumn( domainForm );
	}
```

## 사용 예시

- 리스트를 직렬화하여 컬럼 1개에 저장하는 데 사용했습니다.

```java
@Entity @Getter @Validated
@Table(name = "item_contents")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@SQLDelete(sql = "UPDATE item_contents SET is_deleted = true WHERE id = ?")
public class ItemContents {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String contents;

    @Size(min = 1, max = 10)
    @Convert(converter = ImageUrlConverter.class) // 직렬화하여 한 컬럼에 넣고 싶어서
    private List<ItemDetailImage> detailImageUrl = new ArrayList<>();
}
```

```java
public class ImageUrlConverter implements AttributeConverter<List<ItemDetailImage>, String> {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public String convertToDatabaseColumn(List<ItemDetailImage> attribute) {
        try {
            return objectMapper.writeValueAsString(attribute);
        } catch (JsonProcessingException e) {
            throw new ValidationException("이미지 직렬화에 실패했습니다.");
        }
    }

    @Override
    @SneakyThrows
    public List<ItemDetailImage> convertToEntityAttribute(String dbData) {
        try {
            return objectMapper.readValue(dbData, new TypeReference<List<ItemDetailImage>>() {});
        } catch (JsonProcessingException e) {
            throw new ValidationException("이미지 URL 형식이 다릅니다.");
        }
    }
}
```

- 데이터를 암호화할 때 편하게 사용할수 있습니다.

```jsx
@Converter(autoApply = true) // global 설정했습니다
@RequiredArgsConstructor
public class AccountNumberConverter implements AttributeConverter<AccountNumber, String> {

    private final AesBytesEncryptor aesBytesEncryptor;

    // 기타 메서드
}
```

```java
@Converter
@RequiredArgsConstructor
public class TwoWayEncryptor implements AttributeConverter<String, String> {

    private final AesBytesEncryptor aesBytesEncryptor;

    @Override
    public String convertToDatabaseColumn(String attribute) {
        byte[] encrypt = aesBytesEncryptor.encrypt(attribute.getBytes(StandardCharsets.UTF_8));
        return byteArrayToString(encrypt);
    }

    @Override
    public String convertToEntityAttribute(String dbData) {
        byte[] decryptBytes = stringToByteArray(dbData);
        byte[] decrypt = aesBytesEncryptor.decrypt(decryptBytes);
        return new String(decrypt, StandardCharsets.UTF_8);
    }
		// 기타 메서드
}
```

```java
+---------+-------+---------------------------------------------------------+----------------------------------------------------------------------------------------------------------+
|member_id|bank_id|account                                                  |bankbook_path                                                                                             |
+---------+-------+---------------------------------------------------------+----------------------------------------------------------------------------------------------------------+
|2        |3      |36%-84%28%-106%55%-92%60%-26%75%-14%-112%-22%84%0%107%83%|https://labeling-tool.s3.ap-northeast-2.amazonaws.com/application/members/2-newpow11%40gmail.com/bankbook/|
+---------+-------+---------------------------------------------------------+----------------------------------------------------------------------------------------------------------+
```

## 주의사항

- class level에서 적용할 때는 반드시 `attributeName` 을 지정합니다.

```jsx
@Convert(converter = BooleanToYNConverter.class, attributeName = "vip")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Member {

    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;

    private String name;

    private boolean vip;

}
```

- `@Id` 어노테이션과 같이 사용할 수 없는 것 같습니다.

---

# Refs.

- **자바 ORM 표준 JPA 프로그래밍**
- https://www.baeldung.com/jpa-attribute-converters
- https://namocom.tistory.com/892
