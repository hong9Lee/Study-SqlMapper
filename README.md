# Study-SqlMapper

인터넷 강의를 통해 SqlMapper를 통한 데이터 접근 기술 학습
- Java, SpringBoot, h2, MyBatis, JdbcTemplate, QueryDSL

##
JDBC를 사용할 때 복잡하고 반복적인 작업을 JdbcTemplate을 통해 해결  
JdbcTemplate은 별도의 설정 없이 사용 가능하지만, 동적 쿼리 사용에 어려움이 있다.  
약간의 설정이 필요하지만 MyBatis를 통해 동적 쿼리를 해결할 수 있다.

따라서 프로젝트에 동적 쿼리와 복잡한 쿼리가 많다면 MyBatis, 단순한 쿼리들이 많다면 JdbcTemplate을 선택해 사용하면 될 것 같다.
```
-- JdbcTemplate

String sql = "select id, item_name, price, quantity from item";
  if (StringUtils.hasText(itemName) || maxPrice != null) {
      sql += " where";
}

boolean andFlag = false;
if (StringUtils.hasText(itemName)) {
    sql += " item_name like concat('%',:itemName,'%')";
    andFlag = true;
}

if (maxPrice != null) {
    if (andFlag) {
      sql += " and";
    }
    sql += " price <= :maxPrice";
}
log.info("sql={}", sql);
return template.query(sql, param, itemRowMapper());


-- MyBatis

<select id="findAll" resultType="Item">
      select id, item_name, price, quantity from item
    <where>
      <if test="itemName != null and itemName != ''">
        and item_name like concat('%',#{itemName},'%')
      </if>
               
      <if test="maxPrice != null">
        and price &lt;= #{maxPrice}
      </if>
    </where>
</select>
```

- QueryDSL  
  쿼리를 Java 코드로 type-safe하게 개발할 수 있게 지원하는 프레임워크 (컴파일 단계에서 오류 검출)  
  이와 비슷한 Criteria API의 경우 소스가 너무 복잡하다.

```
public List<Item> findAll(ItemSearchCond cond) {

String itemName = cond.getItemName();
Integer maxPrice = cond.getMaxPrice();

return query
        .select(item)
        .from(item)
        .where(likeItemName(itemName), maxPrice(maxPrice))
        .fetch();
}

private BooleanExpression maxPrice(Integer maxPrice) {
    if (maxPrice != null) {
        return item.price.loe(maxPrice); // 작거나 같다.
    }
  return null;
}

private BooleanExpression likeItemName(String itemName) {
    if(StringUtils.hasText(itemName)) {
      return item.itemName.like("%" + itemName + "%");
    }
  return null;
}


```



## Service 로직 작성
```
-- JdbcTemplate
public Item save(Item item) {
  SqlParameterSource param = new BeanPropertySqlParameterSource(item); // 객체를 Map으로 변환하여 파라미터 바인딩
  Number key = jdbcInsert.executeAndReturnKey(param); // Insert 후 객체의 key 반환
  item.setId(key.longValue());
  return item;
}
...

-- MyBatis
<insert id="save" useGeneratedKeys="true" keyProperty="id">
  insert into item(item_name, price, quantity)
  values (#{itemName}, #{price}, #{quantity})
</insert>
...

```


## Test Code를 통한 검증
```
@Test
void save() {
  //given
  Item item = new Item("itemA", 10000, 10);

  //when
  Item savedItem = itemRepository.save(item);

  //then
  Item findItem = itemRepository.findById(item.getId()).get();
  assertThat(findItem).isEqualTo(savedItem);
}


@Test
void findItems() {
  //given
  Item item1 = new Item("itemA-1", 10000, 10);
  Item item2 = new Item("itemA-2", 20000, 20);
  Item item3 = new Item("itemB-1", 30000, 30);

  itemRepository.save(item1);
  itemRepository.save(item2);
  itemRepository.save(item3);

  //둘 다 없음 검증
  test(null, null, item1, item2, item3);
  test("", null, item1, item2, item3);

  //itemName 검증
  test("itemA", null, item1, item2);
  test("temA", null, item1, item2);
  test("itemB", null, item3);

  //maxPrice 검증
  test(null, 10000, item1);

  //둘 다 있음 검증
  test("itemA", 10000, item1);
}

void test(String itemName, Integer maxPrice, Item... items) {
  List<Item> result = itemRepository.findAll(new ItemSearchCond(itemName, maxPrice));
  assertThat(result).containsExactly(items);
}
...

```
