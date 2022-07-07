# MyBatis

```java
InputStream inputStream = Resources.getResourceAsStream("mybatis.xml");
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
try (SqlSession sqlSession = sqlSessionFactory.openSession()) {
    List<Object> list1 = sqlSession.selectList("com.test.dao.FooDAO.query");
}
```

`SqlSessionFactoryBuilder`类解析配置文件，将所有配置通过`Configuration`对象保存。
解析 mapper 文件中的 SQL 语句包装成`MappedStatement`对象，`MappedStatement`包含了 SQL 本身、SQL 类型、参数类型、结果类型、表字段、映射属性等。
调用`SqlSession`的方法时，会先从`Configuration`中获取对应的`MappedStatement`，然后通过`Executor`生成`PreparedStatement`并执行。

