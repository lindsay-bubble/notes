[TOC]



# 一、引入bean

## 1、有包括所引入依赖项的所有Bean

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = XXXQueryApplicationContext.class)
@ActiveProfiles("test")
public class BaseQueryTest {

    @Test
    public void test() {
        //
    }

}
```

## 2、只引入自己项目里的bean

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = QueryMockConfig.class)
@ActiveProfiles("test")
public class BaseQueryTest {

    @Test
    public void test() {
        //
    }

}

@Configuration
@Import(value = {
        XXXQueryApplicationContext.class
})
@Profile("test")
public class QueryMockConfig {
}
```

# 二、Mock Bean

## 1、用反射替换@Autowired的变量

```java
// ReflectionTestUtils.setField(对象，被Autowired的域名，mock出来的该域名对象);
ReflectionTestUtils.setField(weixinService, "resourceFileMapService", resourceFileMapService);

```

## 2、mock局部方法

### 1）mock

```java
Stock stock = mock(Stock.class);
when(stock.getQuantity()).thenReturn(200);    // Mock implementation
when(stock.getValue()).thenCallRealMethod();  // Real implementation
```

### 2）spy

```java
Stock stock = spy(Stock.class);
when(stock.getQuantity()).thenReturn(200);    // Mock implementation
// All other method call will use the real implementations
```



