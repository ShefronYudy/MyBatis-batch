# MyBatis-batch
基于 MyBatis 执行 SQL 批量操作的插件。

MyBatis conf 配置：
```xml
<configuration>
	<plugins>
		<plugin interceptor="net.coderbee.mybatis.batch.BatchParameterHandler" />
		<plugin interceptor="net.coderbee.mybatis.batch.BatchStatementHandler" />
	</plugins>
</configuration>
```

测试表的 SQL 语句：
```sql
create table simple_user (
	id int not null generated by default as identity (start with 1000),
	name varchar(255) not null,
	email varchar(255) not null,
	primary key (id)
);
```

Mapper XML 编写说明，需要把 `parameterType` 指定为 `net.coderbee.mybatis.batch.BatchParameter`，如下：
```xml
<update id="updateBatch" parameterType="net.coderbee.mybatis.batch.BatchParameter">
	update simple_user set name = name || #{} where id = #{id}
</update>

<sql id="insertSql">
	insert into simple_user (
		name,
		email
	) values (
		#{name},
		#{email}
	)
</sql>

<insert id="batchInsertByMap" useGeneratedKeys="true"
	keyProperty="id" parameterType="net.coderbee.mybatis.batch.BatchParameter">
	<include refid="insertSql" />
</insert>
```


调用示例见  [TestBatch.java](https://github.com/wen866595/MyBatis-batch/blob/master/src/test/java/net/coderbee/mybatis/batch/TestBatch.java)，截取部分如下：
```java
@SuppressWarnings("unchecked")
@Test
public void testOnHsqldb() {
	SqlSession sqlSession = hsqldbSqlSessionFactory.openSession();

	UserMapper userMapper = sqlSession.getMapper(UserMapper.class);

	List<User> userList = buildUsers();

	BatchParameter<User> users = BatchParameter.wrap(userList);
	userMapper.batchInsert2Hsqldb(users);

	checkResult(userList);

	testBatchDelete(userMapper);

	Map<String, Object> map = new HashMap<String, Object>();
	map.put("name", "abc123");
	map.put("email", "abc123@126.com");
	List<Map<String, Object>> list = Arrays.asList(map);
	userMapper.batchInsertByMap(
			BatchParameter.<Map<String, Object>> wrap(list));

	Assert.assertNotNull(((Number) map.get("id")).intValue() == 1003);
}

public void testBatchDelete(UserMapper userMapper) {
	List<User> list = userMapper.selectAll();

	List<User> users = buildToUpdateUsers();
	int counts = userMapper.deleteBatch(BatchParameter.wrap(users));
	Assert.assertTrue(counts == 2);

	List<User> afterDeleteList = userMapper.selectAll();
	Assert.assertTrue(list.size() - afterDeleteList.size() == 2);
}

protected List<User> buildToUpdateUsers() {
	List<User> users = new ArrayList<User>();
	users.add(createUser(1, "-x"));
	users.add(createUser(2, "-y"));
	users.add(createUser(3, "-z"));
	return users;
}

protected void checkResult(List<User> userList) {
	int idStart = 1000; // 自动增长的 ID 从 1000 开始
	for (User user : userList) {
		Assert.assertEquals(idStart++, user.getId());
	}
}

protected List<User> buildUsers() {
	List<User> userList = new ArrayList<User>();
	userList.add(createUser("bruce.liu", "wen866595@163.com"));
	userList.add(createUser("coderbee.net", "wen866595@gmail.com"));
	userList.add(createUser("world", "hello@gmail.com"));
	return userList;
}

private User createUser(int id, String name) {
	User u = new User();
	u.setId(id);
	u.setName(name);
	return u;
}

private User createUser(String name, String email) {
	User u = new User();
	u.setName(name);
	u.setEmail(email);
	return u;
}
```

