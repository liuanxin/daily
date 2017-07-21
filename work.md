***
## 通过 Example 简化查询
如果我们想要生成如下的 sql 语句:
```sql
select xxx from `t_user` where user_name = 'sz' and level > 1 and verify in (1, 2, 3)
```
这样构建  example 来达到上面的效果
```java
UserExample userExample = new UserExample();
userExample.or().andUserNameEqualTo(phone).andLevelGreaterThan(1).andVerifyIn(Arrays.asList(1, 2, 3));
userMapper.selectByExample(userExample);
```
同样的 where 条件也可以用在 update 和 delete 上
```java
UserExample userExample = new UserExample();
userExample.or().andUserNameNameEqualTo(phone).andLevelGreaterThan(1).andVerifyIn(Arrays.asList(1, 2, 3));
userMapper.updateByExampleSelective(new User().setPassword("abc"), userExample);
```
上面将会生成如下的 sql 语句
```sql
update `t_user` set password='abc' where user_name = 'sz' and level > 1 and verify in (1, 2, 3)
```
如果要生成 or 语句, 可以像这样
```java
UserExample userExample = new UserExample();
userExample.or().andUserNameEqualTo("xxx").andCreateTimeLessThan(new Date());
userExample.or().andEmailEqualTo("xxx").andCertIsNotNull();
userExample.or().andPhoneEqualTo("xxx").andVerifyIn(Arrays.asList(1, 2, 3));
userMapper.selectByExample(userExample);
```
生成的 sql 如下
```sql
select xx from `t_user` where (name = 'xx' and create_time &lt; now) or (email = 'xxx' and `cert` is not null) or (phone = 'xxx' and verify in (1, 2, 3))
```

如果要生成条件复杂的 or 语句(比如在一个 and 条件里面有好几个 or), exmple 将会无法实现, 此时就需要手写 sql 了

***
## 何时需要手工写自己的 _customer mapper 文件
当有一些不得不联表的 sql 语句, 或者基于 example 很难生成的 or 查询, 此时放在 custom.xml 中, 确保自动生成和手写的 sql 分开管理.

PS: 尽量不要使用 join 来联表, 尽量由应用程序来组装数据并每次向数据库发起单一且易维护的 sql 语句, 这样的好处是就算到了大后期, 对于数据库而言, 压力也全在单表的 sql 上, 优化起来很容易, 而且应用程序还可以在这里加上二级缓存, 将大部分的压力由 db 的 io 操作转移到了应用程序的内部运算和网卡的数据库连接上, java 做内部运算本就是强项, 这一块绝对不可能成为瓶颈, 数据库连接可以由 druid 连接池来达到高性能操作.

阿里在 17 年初出的开发手册中也明确说明: 超级三个表禁止 join, 是有其原因的

***
## 如何把枚举类映射成数据库字段
枚举可以参考 性别 这个类
```java
import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonValue;
import com.weiqitong.common.util.U;

/** 用户性别 */
public enum Gender {

    Nil(0, "未知"), Male(1, "男"), Female(2, "女");

    private int code;
    private String value;
    private Gender(int code, String value) {
        this.code = code;
        this.value = value;
    }

    /** 显示用 */
    public String getValue() {
        return value;
    }
    /** 数据关联用 */
    @JsonValue
    public int getCode() {
        return code;
    }
    /** 数据反序列化时调用 */
    @JsonCreator
    public static Gender deserializer(Object obj) {
        return U.toEnum(Gender.class, obj);
    }
}
```
其中 code 和 value 都要有, 一个用来存入数据库, 一个用来显示, 两个 jackson 的注解已经说明了序列化和反序列化的规则, 此时, 还需要让 mybatis 也知道, 我在每个模块的 test 中放了 GenerateEnumHandle 这个测试类, 运行后会在当前模块的 handler 包中生成对应的枚举处理类, 就像下面这样
```java
import com.weiqitong.common.util.U;
import com.weiqitong.user.enums.Gender;
import org.apache.ibatis.type.BaseTypeHandler;
import org.apache.ibatis.type.JdbcType;

import java.sql.CallableStatement;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

/**
 * 当前 handle 是自动生成的
 *
 * @see org.apache.ibatis.type.TypeHandlerRegistry
 * @see org.apache.ibatis.type.EnumTypeHandler
 * @see org.apache.ibatis.type.EnumOrdinalTypeHandler
 */
public class GenderHandler extends BaseTypeHandler {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Gender parameter,
                                    JdbcType jdbcType) throws SQLException {
        if (jdbcType == null || jdbcType == JdbcType.VARCHAR) {
            // 如果数据库的字段类型是 varchar, 就使用枚举的 name 来存储
            ps.setString(i, parameter.name());
        } else {
            // 否则使用 getCode 返回的 int 值来存储
            ps.setInt(i, parameter.getCode());
        }
    }

    @Override
    public Gender getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return U.toEnum(Gender.class, rs.getObject(columnName));
    }

    @Override
    public Gender getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return U.toEnum(Gender.class, rs.getObject(columnIndex));
    }

    @Override
    public Gender getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return U.toEnum(Gender.class, cs.getObject(columnIndex));
    }
}
```
这个类会被装载到 mybatis 的上下文中去, 这样在整个项目过程中, 任意地方都可以直接使用枚举而不需要基于数值转来转去

***
## 如何开启 MyBatis 端的 redis 缓存
在相应的模块中添加如下的配置(这个 md 插件处理不了尖括号对)
```xml
<dependency>
    <groupId>com.github.mte</groupId>
    <artifactId>mybatis-redis-cache</artifactId>
    <scope>provided</scope>
</dependency>
```
并在对应的 mapper.xml 中添加下面的代码(这个 md 插件处理不了尖括号)
```xml
<cache type="com.github.mte.caches.RedisCache" />
```
此 xml 中所有的 sql 都会走缓存, 相关的 redis 配置会先从 applition.yml 中获取, 如果未获取到, 再读 redis.properties

***
## NeedLogin 注解的使用场景
当某个方法是需要登录才能请求的, 在此方法上标此注解即可. 用户在调用此方法时如果未登录将会直接返回 code 10
PS: 在此方法中可以直接使用 ApiSessionUtil.getUserId 进行获取

***
## 一些提升开发效率的插件

lombok plugin: 在类上标注解来自动给实体生成 set get 及构造方法
free mybatis plugin: 直接在 mapper 类及 xml 中快速定位, 也可以在 xml 中手写 sql 时有更多提示
