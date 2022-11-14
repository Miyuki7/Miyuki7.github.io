1.数据库表

#### 用户表 user

| 字段            | 类型      | 备注                               |
| --------------- | --------- | ---------------------------------- |
| id              | int       | 主键、自增                         |
| username        | varchar   | 用户名，创建索引                   |
| password        | varchar   | 用户密码                           |
| salt            | varchar   | 加密盐值                           |
| email           | varchar   | 用户邮箱，创建索引                 |
| type            | int       | 用户类型：0 普通、1 管理员、2 版主 |
| status          | int       | 用户状态：0 未激活、1 已激活       |
| activation_code | varchar   | 激活码                             |
| header_url      | varchar   | 用户头像地址                       |
| create_time     | timestamp | 注册时间                           |

#### 评论表 comment

| 字段        | 类型      | 备注                                 |
| ----------- | --------- | ------------------------------------ |
| id          | int       | 主键、自增                           |
| user_id     | int       | 评论的用户 id，创建索引              |
| entity_id   | int       | 评论实体 id，创建索引                |
| entity_type | int       | 评论实体类型：1 帖子评论、2 评论回复 |
| target_id   | int       | 评论目标 id                          |
| content     | text      | 评论内容                             |
| status      | int       | 评论状态：0 有效、1 无效             |
| create_time | timestamp | 评论发表时间                         |

#### 帖子表 discuss_post

| 字段          | 类型      | 备注                             |
| ------------- | --------- | -------------------------------- |
| id            | int       | 主键、自增                       |
| user_id       | int       | 发帖的用户 id，创建索引          |
| title         | varchar   | 帖子表标题                       |
| content       | text      | 帖子内容                         |
| type          | int       | 帖子类型：0 普通、1 置顶         |
| comment_count | int       | 评论数量                         |
| status        | int       | 帖子状态：0 普通、1 精华、2 拉黑 |
| score         | int       | 帖子分数，用于排名               |
| create_time   | timestamp | 帖子创建时间                     |

#### 用户登录凭证表 login_ticket

后续作废，将token存入redis中

| 字段    | 类型      | 备注                     |
| ------- | --------- | ------------------------ |
| id      | int       | 主键、自增               |
| user_id | int       | 登录用户 id              |
| ticket  | varchar   | 登录凭证，随机字符串     |
| status  | int       | 登录状态：0 有效、1 无效 |
| expired | timestamp | 过期时间                 |

#### 消息表 message

| 字段            | 类型      | 备注                                                         |
| --------------- | --------- | ------------------------------------------------------------ |
| id              | int       | 主键、自增                                                   |
| from_id         | int       | 发消息的 id，创建索引，1代表系统信息                         |
| to_id           | int       | 收消息的 id，创建索引                                        |
| conversation_id | varchar   | 会话 id，由通信双方 id 拼接，创建索引，如果是系统消息，该字段存储的是comment或者like或者关注 |
| content         | text      | 消息内容，不是系统消息的话存储的是具体内容，是系统消息的话，存一条json数据，详情看下图 |
| status          | int       | 消息状态：0 未读、1 已读、2 删除                             |
| create_time     | timestamp | 消息发送时间                                                 |

![image-20221113113911288](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113113911288.png)

![image-20221113114107362](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113114107362.png)

## 2.搭建基本环境

构建 SpringBoot 的 maven 项目，引入 mysql 和 mybatis 依赖。

------

在 `application.properties` 配置文件中：

- 关闭 thymeleaf 缓存 

- 配置数据库，设置基本连接信息、最大线程数，最小空闲线程数，最大空闲时间等 

- mybatis，设置 mapper 文件的位置、实体类包名、使用主键、日志级别（debug方便调试）等 

  ![image-20221109154910273](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109154910273.png)

------

创建 community 数据库和数据库表。

------

用户相关操作：

- 创建对应 user 表的 User 实体类 

- 创建 UserMapper 接口，使用 `@Mapper` 注解 

- 创建 user-mapper.xml，重复 sql 语句可以写在 `<sql id = "xxx">` 标签，通过 `<include refid="xxx"/>` 引用，#{id}引入真正的属性。

  ![image-20221109154618164](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109154618164.png)

  ![image-20221109154659631](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109154659631.png)

作者：是瑶瑶公主呀
链接：https://www.nowcoder.com/discuss/481089?type=5
来源：牛客网



## 3.开发社区首页（discuss_post 表）

功能拆分：开发社区首页，显示前 10 个帖子。开发分页组件，分页显示所有帖子。

用到的表是 discuss_post 数据库表，包括帖子 id、发帖人 id、标题、内容、类型、状态、发帖时间、评论数量（为了提高效率，避免关联查询，因此冗余存储）、分数（用于进行热度排名）。

### 开发数据层

帖子相关操作：

- 创建对应 discuss_post 表的 DisscussPost 实体类。
- 创建 DisscussPostMapper 接口，使用 `@Mapper` 注解。
  - 分页查询中用户 id 是可选参数，通过动态 SQL 选择，如果为 0 就不使用，在开发用户个人主页查询用户发帖记录时需要使用。 
  
  - 如果只有一个参数，并且在动态 SQL 的 `<if>` 里使用，必须使用 `@Param` 加别名。 
  
    ![image-20221109160001305](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109160001305.png)
- 创建 `disscusspost-mapper.xml`。
  - `where status != 2` 拉黑的帖子不展现。
  
  - `<if test="userId!=0">`  userID 为 0查询的时所有帖子，userID不为0时查询某个用户的帖子列表，按照类型，发帖时间排序。
  
    ![image-20221109160038322](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109160038322.png)

------

### 开发业务层

创建 DiscussPostService 类，可以分页查询帖子和帖子数量。

创建 UserService 类，实现根据 id 查询用户功能，因为显示帖子时不显示用户 id，而是显示用户名，到时候组合返回给前端就可以显示用户名了。

![image-20221109161058316](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109161058316.png)

![image-20221109161226850](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109161226850.png)

------

### 开发视图层

把静态资源 css、html、img、js 放到 static 目录下。

把模板 mail、site、index.html 放到 template 目录下。

创建 HomeController，`getIndexPage` 方法，用 map 集合把帖子和用户封装到一起。

修改 `index.html`，使用 `<th:text="${map.xxx.xxx}"` 动态替换。

【问题】使用帖子关联查询用户时，给查询用户的 `findUserById` 方法传入了帖子的 `getId` 方法，应该是 `getUserId` 方法。

![image-20221109161829029](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109161829029.png)

![image-20221109162407178](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109162407178.png)

------

### 开发分页组件

创建 Page 实体类，封装分页信息，包括当前页码、显示限制、帖子总数、查询路径等。显示的起始页不能小于 1，最大页不能超过 total。

在 `index.html` 中，当 `page.rows > 0` 时显示分页信息。

如果 `page.current`  等于 1 或 `page.total`，代表是首页或末页，此时不能点击上一页和下一页，用 `disabled` 属性实现。

> [!TIP]
>
> ```
> 方法调用钱,SpringMVC会自动实例化Model和Page,并将Page注入Model.所以,在thymeleaf中可以直接访问Page对象中的数据.
> ```

``` java
public class Page {

    // 当前页码
    private int current = 1;
    // 显示上限
    private int limit = 10;
    // 数据总数(用于计算总页数)
    private int rows;
    // 查询路径(用于复用分页链接)
    private String path;

    public int getCurrent() {
        return current;
    }

    public void setCurrent(int current) {
        if (current >= 1) {
            this.current = current;
        }
    }

    public int getLimit() {
        return limit;
    }

    public void setLimit(int limit) {
        if (limit >= 1 && limit <= 100) {
            this.limit = limit;
        }
    }

    public int getRows() {
        return rows;
    }

    public void setRows(int rows) {
        if (rows >= 0) {
            this.rows = rows;
        }
    }

    public String getPath() {
        return path;
    }

    public void setPath(String path) {
        this.path = path;
    }

    /**
     * 获取当前页的起始行
     *
     * @return
     */
    public int getOffset() {
        // current * limit - limit
        return (current - 1) * limit;
    }

    /**
     * 获取总页数
     *
     * @return
     */
    public int getTotal() {
        // rows / limit [+1]
        if (rows % limit == 0) {
            return rows / limit;
        } else {
            return rows / limit + 1;
        }
    }

    /**
     * 获取起始页码
     *
     * @return
     */
    public int getFrom() {
        int from = current - 2;
        return from < 1 ? 1 : from;
    }

    /**
     * 获取结束页码
     *
     * @return
     */
    public int getTo() {
        int to = current + 2;
        int total = getTotal();
        return to > total ? total : to;
    }

}
```

![image-20221109164923424](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109164923424.png)

拼接后的访问路径是/index?current = 1这种，SpringMvc会自动将current参数装入到服务器端page中的current。

![image-20221109165446656](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109165446656.png)



###  项目日志级别设置

日志一共四种INFO、debug、warn、error，配置日志存放路径和文件夹，配置针对每种级别的具体策略，文件名滚动策略、日志具体格式、是否追加或者覆盖，下面注释很详尽。例：D:/work/data/community/error/log-error-2022-10-02.1.log

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <contextName>community</contextName>
    <!-- 日志存放路径-->
    <property name="LOG_PATH" value="D:/work/data"/>
    <!-- 日志存放文件夹-->
    <property name="APPDIR" value="community"/>

    <!-- error file error级别的日志文件-->
    <appender name="FILE_ERROR" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APPDIR}/log_error.log</file>
        <!-- 日志滚动策略 每份日志最大5mb，按天数当文件名-->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APPDIR}/error/log-error-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>5MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <!-- append代表追加方式，会形成多个文件-->
        <append>true</append>
        <!-- 具体的日志记录格式： 级别+线程+文件的第几行+具体的信息-->
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%d %level [%thread] %logger{10} [%file:%line] %msg%n</pattern>
            <charset>utf-8</charset>
        </encoder>
        <!--设置过滤器表示拦截error级别的日志-->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>error</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>
<!-- console控制台日志 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d %level [%thread] %logger{10} [%file:%line] %msg%n</pattern>
            <charset>utf-8</charset>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>debug</level>
        </filter>
    </appender>
    <!--设置该项目种的日志级别是debug-->
    <logger name="com.nowcoder.community" level="debug"/>
    <!--总设置-->
    <root level="info">
        <appender-ref ref="FILE_ERROR"/>
        <appender-ref ref="FILE_WARN"/>
        <appender-ref ref="FILE_INFO"/>
        <appender-ref ref="STDOUT"/>
    </root>

</configuration>
```

作者：是瑶瑶公主呀
链接：https://www.nowcoder.com/discuss/481089?type=5
来源：牛客网



## 4.开发注册登录模块

------

### 发送邮件

在新浪邮箱打开 SMTP 服务。

引入 `spring-boot-starter-mail` 依赖。

在配置文件配置主机、端口、发送邮箱、授权码等。

创建 MailClient 类，调用 JavaMailSender 发送邮件。

使用 thymeleaf 发送 HTML 邮件，调用 TemplateEngine 把信息封装到 HTML 模板。

``` xml
# spring 相关配置
spring:
  # 发送者邮箱相关配置
  mail:
    # SMTP服务器域名
    host: smtp.sina.com
    # 编码集
    default-encoding: UTF-8
    # 邮箱用户名
    username: csp******@163.com
    # 授权码(注意不是邮箱密码！)
    password: WDS*******XCQA
    # 协议：smtps
    protocol: smtps
    # 详细配置
    properties:
      mail:
        smtp:
          # 设置是否需要认证,如果为true,那么用户名和密码就必须的,
          # 如果设置false,可以不设置用户名和密码
          # (前提要知道对接的平台是否支持无密码进行访问的)
          auth: true
          # STARTTLS[1]  是对纯文本通信协议的扩展。
          # 它提供一种方式将纯文本连接升级为加密连接（TLS或SSL）
          # 而不是另外使用一个端口作加密通信。
          starttls:
            enable: true
            required: true
```

![image-20221109180937496](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109180937496.png)

【问题】发送邮件成功但没接收到，在垃圾箱中可找到。

------

### 注册功能

* 把 register.html 地址关联到首页的注册 href 属性。

* 设置域名、创建 CommunityUtil 工具类，在工具类创建生产随机字符串和 MD5 加密方法。

​       在properties文件中配置域名

![image-20221109181558108](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109181558108.png)

md5加密中的digestutils是spring中的工具。

![image-20221109182120390](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109182120390.png)

创建 LoginController，创建 `getRegisterPage` 方法，跳转注册页面。

``` java
@RequestMapping(path = "/register", method = RequestMethod.GET)
    public String getRegisterPage() {
        return "/site/register";
    }
```



在 UserService 中创建 `register` 方法，判断注册信息合规后插入数据库，发送激活邮件。

``` java
public Map<String, Object> register(User user) {
        Map<String, Object> map = new HashMap<>();

        // 空值处理
        if (user == null) {
            throw new IllegalArgumentException("参数不能为空!");
        }
        if (StringUtils.isBlank(user.getUsername())) {
            map.put("usernameMsg", "账号不能为空!");
            return map;
        }
        if (StringUtils.isBlank(user.getPassword())) {
            map.put("passwordMsg", "密码不能为空!");
            return map;
        }
        if (StringUtils.isBlank(user.getEmail())) {
            map.put("emailMsg", "邮箱不能为空!");
            return map;
        }

        // 验证账号
        User u = userMapper.selectByName(user.getUsername());
        if (u != null) {
            map.put("usernameMsg", "该账号已存在!");
            return map;
        }

        // 验证邮箱
        u = userMapper.selectByEmail(user.getEmail());
        if (u != null) {
            map.put("emailMsg", "该邮箱已被注册!");
            return map;
        }

        // 注册用户
        user.setSalt(CommunityUtil.generateUUID().substring(0, 5));
        user.setPassword(CommunityUtil.md5(user.getPassword() + user.getSalt()));
        user.setType(0);
        user.setStatus(0);
        user.setActivationCode(CommunityUtil.generateUUID());
        user.setHeaderUrl(String.format("http://images.nowcoder.com/head/%dt.png", new Random().nextInt(1000)));
        user.setCreateTime(new Date());
        userMapper.insertUser(user);

        // 激活邮件 调用templateEngine引擎将context种保存的email和url动态替换到html中
        Context context = new Context();
        context.setVariable("email", user.getEmail());
        // http://localhost:8080/community/activation/101/code
        String url = domain + contextPath + "/activation/" + user.getId() + "/" + user.getActivationCode();
        context.setVariable("url", url);
        String content = templateEngine.process("/mail/activation", context);
        mailClient.sendMail(user.getEmail(), "激活账号", content);

        return map;
    }
```



在 LoginController 创建 `register` 方法，调用 UserService 的 `register` 方法。注册成功就跳转到操作成功页面。

``` java
@RequestMapping(path = "/register", method = RequestMethod.POST)
    public String register(Model model, User user) {
        Map<String, Object> map = userService.register(user);
        if (map == null || map.isEmpty()) {
            model.addAttribute("msg", "注册成功,我们已经向您的邮箱发送了一封激活邮件,请尽快激活!");
            model.addAttribute("target", "/index");
            //跳转到操作结果页面
            return "/site/operate-result";
        } else {
            model.addAttribute("usernameMsg", map.get("usernameMsg"));
            model.addAttribute("passwordMsg", map.get("passwordMsg"));
            model.addAttribute("emailMsg", map.get("emailMsg"));
            return "/site/register";
        }
    }
```

![image-20221109185237870](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109185237870.png)

创建接口 CommunityConstant，定义激活码的三种状态，成功、重复、失败，让 UserService 和 LoginController 实现该接口。

``` java
public interface CommunityConstant {

    /**
     * 激活成功
     */
    int ACTIVATION_SUCCESS = 0;

    /**
     * 重复激活
     */
    int ACTIVATION_REPEAT = 1;

    /**
     * 激活失败
     */
    int ACTIVATION_FAILURE = 2;

    /**
     * 默认状态的登录凭证的超时时间
     */
    int DEFAULT_EXPIRED_SECONDS = 3600 * 12;

    /**
     * 记住状态的登录凭证超时时间
     */
    int REMEMBER_EXPIRED_SECONDS = 3600 * 24 * 100;

    /**
     * 实体类型: 帖子
     */
    int ENTITY_TYPE_POST = 1;

    /**
     * 实体类型: 评论
     */
    int ENTITY_TYPE_COMMENT = 2;

    /**
     * 实体类型: 用户
     */
    int ENTITY_TYPE_USER = 3;

    /**
     * 主题: 评论
     */
    String TOPIC_COMMENT = "comment";

    /**
     * 主题: 点赞
     */
    String TOPIC_LIKE = "like";

    /**
     * 主题: 关注
     */
    String TOPIC_FOLLOW = "follow";

    /**
     * 主题: 发帖
     */
    String TOPIC_PUBLISH = "publish";

    /**
     * 主题: 删帖
     */
    String TOPIC_DELETE = "delete";

    /**
     * 主题: 分享
     */
    String TOPIC_SHARE = "share";

    /**
     * 系统用户ID
     */
    int SYSTEM_USER_ID = 1;

    /**
     * 权限: 普通用户
     */
    String AUTHORITY_USER = "user";

    /**
     * 权限: 管理员
     */
    String AUTHORITY_ADMIN = "admin";

    /**
     * 权限: 版主
     */
    String AUTHORITY_MODERATOR = "moderator";

}
```

``` java
userservice.class
public int activation(int userId, String code) {
        User user = userMapper.selectById(userId);
        if (user.getStatus() == 1) {
            return ACTIVATION_REPEAT;
        } else if (user.getActivationCode().equals(code)) {
            userMapper.updateStatus(userId, 1);
            clearCache(userId);
            return ACTIVATION_SUCCESS;
        } else {
            return ACTIVATION_FAILURE;
        }
    }
logincontroller
// http://localhost:8080/community/activation/101/code
    @RequestMapping(path = "/activation/{userId}/{code}", method = RequestMethod.GET)
    public String activation(Model model, @PathVariable("userId") int userId, @PathVariable("code") String code) {
        int result = userService.activation(userId, code);
        if (result == ACTIVATION_SUCCESS) {
            model.addAttribute("msg", "激活成功,您的账号已经可以正常使用了!");
            model.addAttribute("target", "/login");
        } else if (result == ACTIVATION_REPEAT) {
            model.addAttribute("msg", "无效操作,该账号已经激活过了!");
            model.addAttribute("target", "/index");
        } else {
            model.addAttribute("msg", "激活失败,您提供的激活码不正确!");
            model.addAttribute("target", "/index");
        }
    	//跳转到该页面，该页面里面有js实现的自动跳转功能
        return "/site/operate-result";
    }
```



点击激活邮件的 url 【本地服务器的url】后，服务器通过 LoginController 的 `activation` 方法查询数据库用户，如果 url 中的激活码和设置的一样，就把用户 status 改为 1。

------

### 会话管理（session、cookie）

![image-20221109193950353](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109193950353.png)

![image-20221109193414471](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109193414471.png)

服务器创建的cookie里面只能存字符串

服务器创建的session可以存任何东西，表现形式是在服务器端存储session，生成一个含有jsessionid的cookie返回给客户端，客户端可以凭借这个来做到记忆性。

由于现在网站基本是多台服务器分布式部署的，如果将用户信息存到session中，而session是存到服务器上，在分布式环境下，由于各个服务器主机之间的信息并不共享，将用户信息存到服务器1上，同一个用户的下一个请求过来的时候，由于nginx的负载均衡策略，去请求了服务器2，就找不到之前的session了。下面介绍几种分布式Session问题的解决策略。

1. 粘性session：同一个ip分给同一个服务器，很难做负载均衡。

2. 同步Session：当一个服务器创建了session之后，会将该Session同步给其他，服务器。服务器之间耦合，加大服务器之间的同步开销。

3. Session服务器：专门一个服务器管理Session，这台服务器是单体的，万一挂掉，有安全隐患。

4. **将客户端会话数据不存到Session中而是存到数据库中：**
   关系型数据库性能较慢
   存到redis中（项目中采用的方式）

   ![image-20221109203605285](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109203605285.png)

![image-20221109193823756](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109193823756.png)

**session、cookie相关示例代码：**

``` java
@RequestMapping(path = "/cookie/set", method = RequestMethod.GET)
    @ResponseBody
    public String setCookie(HttpServletResponse response) {
        // 创建cookie
        Cookie cookie = new Cookie("code", CommunityUtil.generateUUID());
        // 设置cookie生效的范围
        cookie.setPath("/community/alpha");
        // 设置cookie的生存时间
        cookie.setMaxAge(60 * 10);
        // 发送cookie
        response.addCookie(cookie);

        return "set cookie";
    }

    @RequestMapping(path = "/cookie/get", method = RequestMethod.GET)
    @ResponseBody
    public String getCookie(@CookieValue("code") String code) {
        System.out.println(code);
        return "get cookie";
    }

    // session示例

    @RequestMapping(path = "/session/set", method = RequestMethod.GET)
    @ResponseBody
    public String setSession(HttpSession session) {
        session.setAttribute("id", 1);
        session.setAttribute("name", "Test");
        return "set session";
    }

    @RequestMapping(path = "/session/get", method = RequestMethod.GET)
    @ResponseBody
    public String getSession(HttpSession session) {
        System.out.println(session.getAttribute("id"));
        System.out.println(session.getAttribute("name"));
        return "get session";
    }
```

------

### 生成验证码

在 `pom.xml` 导入 kaptcha 的 jar 包。

创建配置类 KaptchaConfig，设置验证码的大小、范围、长度等。

``` java
@Configuration
public class KaptchaConfig {

    @Bean
    public Producer kaptchaProducer() {
        Properties properties = new Properties();
        properties.setProperty("kaptcha.image.width", "100");
        properties.setProperty("kaptcha.image.height", "40");
        properties.setProperty("kaptcha.textproducer.font.size", "32");
        properties.setProperty("kaptcha.textproducer.font.color", "0,0,0");
        properties.setProperty("kaptcha.textproducer.char.string", "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYAZ");
        properties.setProperty("kaptcha.textproducer.char.length", "4");
        properties.setProperty("kaptcha.noise.impl", "com.google.code.kaptcha.impl.NoNoise");

        DefaultKaptcha kaptcha = new DefaultKaptcha();
        Config config = new Config(properties);
        kaptcha.setConfig(config);
        return kaptcha;
    }

}
```



在 LoginController 类新增 `getKaptcha` 方法生成验证码图片,

最初将验证码存入session中，后面改进了将验证码存入redis中。

``` java
 @RequestMapping(path = "/kaptcha", method = RequestMethod.GET)
    public void getKaptcha(HttpServletResponse response/*, HttpSession session*/) {
        // 生成验证码
        String text = kaptchaProducer.createText();
        BufferedImage image = kaptchaProducer.createImage(text);

        // 将验证码存入session
        // session.setAttribute("kaptcha", text);
        // 验证码的归属
        String kaptchaOwner = CommunityUtil.generateUUID();
        Cookie cookie = new Cookie("kaptchaOwner", kaptchaOwner);
        cookie.setMaxAge(60);
        cookie.setPath(contextPath);
        response.addCookie(cookie);
        // 将验证码存入Redis
        String redisKey = RedisKeyUtil.getKaptchaKey(kaptchaOwner);
        redisTemplate.opsForValue().set(redisKey, text, 60, TimeUnit.SECONDS);

        // 将突图片输出给浏览器
        response.setContentType("image/png");
        try {
            OutputStream os = response.getOutputStream();
            ImageIO.write(image, "png", os);
        } catch (IOException e) {
            logger.error("响应验证码失败:" + e.getMessage());
        }
    }
```



在 `login.html` 中，将刷新验证码的链接绑定 `refresh_kaptcha` 方法，通过 id 选择器获取 img 组件，重新访问  `getKaptcha` 方法生成验证码图片。

``` javascript
function refresh_kaptcha() {
			var path = CONTEXT_PATH + "/kaptcha?p=" + Math.random();
			$("#kaptcha").attr("src", path);
		}
```



【问题】由于访问同一个生成验证码路径，需要在 url 参数加上一个随机数字，保证会重新请求获取新图片。

------

### 登录退出功能（login_ticket 表）

![image-20221109205916163](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109205916163.png)

登录成功时，需要生成一个登录凭证发送给客户端。凭证可以在多个业务中连续地验证用户的登陆状态，凭证信息存储在 login_ticket 数据库表中，status 的 0 和 1 表示有效和无效，expire 表示过期时间。

创建对应 login_ticket 表的 LoginTicket 实体类，对应 login_ticket 数据库表。

![image-20221109210124396](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109210124396.png)

创建 LoginTicketMapper 接口，通过 `@Insert`、`@Select`、`@Update` 注解来插入、查询、更新凭证。

![image-20221109210628379](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109210628379.png)

在 UserService 

- 创建 `login` 方法，验证账户合规后将凭证信息插入数据库，添加登录凭证到 map 中。(改进之后，将登录凭证存入redis中)

  ``` java
  public Map<String, Object> login(String username, String password, long expiredSeconds) {
          Map<String, Object> map = new HashMap<>();
          // 空值处理
          if (StringUtils.isBlank(username)) {
              map.put("usernameMsg", "账号不能为空!");
              return map;
          }
          if (StringUtils.isBlank(password)) {
              map.put("passwordMsg", "密码不能为空!");
              return map;
          }
          // 验证账号
          User user = userMapper.selectByName(username);
          if (user == null) {
              map.put("usernameMsg", "该账号不存在!");
              return map;
          }
          // 验证状态
          if (user.getStatus() == 0) {
              map.put("usernameMsg", "该账号未激活!");
              return map;
          }
          // 验证密码
          password = CommunityUtil.md5(password + user.getSalt());
          if (!user.getPassword().equals(password)) {
              map.put("passwordMsg", "密码不正确!");
              return map;
          }
  
          // 生成登录凭证
          LoginTicket loginTicket = new LoginTicket();
          loginTicket.setUserId(user.getId());
          loginTicket.setTicket(CommunityUtil.generateUUID());
          loginTicket.setStatus(0);
          loginTicket.setExpired(new Date(System.currentTimeMillis() + expiredSeconds * 1000));
          //改进前，将登录凭证插入到数据库
  //        loginTicketMapper.insertLoginTicket(loginTicket);
          //改进后，将登录凭证放入redis中
          String redisKey = RedisKeyUtil.getTicketKey(loginTicket.getTicket());
          redisTemplate.opsForValue().set(redisKey, loginTicket);
  
          map.put("ticket", loginTicket.getTicket());
          return map;
      }
  ```

  

- 创建 `logout` 方法，将对应凭证设为无效。(改进前，是要修改数据库中登录凭证的状态；改进后，修改登录凭证再redis中的状态即可)

  ``` java
  public void logout(String ticket) {
  //        loginTicketMapper.updateStatus(ticket, 1);
          String redisKey = RedisKeyUtil.getTicketKey(ticket);
          LoginTicket loginTicket = (LoginTicket) redisTemplate.opsForValue().get(redisKey);
          loginTicket.setStatus(1);
          redisTemplate.opsForValue().set(redisKey, loginTicket);
      }
  ```

  

在 LoginController 

- 创建 `login` 方法，判断验证码正确后调用  UserServce 的 `login` 方法，如果 map 包含 ticket 代表登录成功，将ticket设置到cookie中重定向跳转首页，否则添加错误信息并跳回登录页。

  ``` java
  @RequestMapping(path = "/login", method = RequestMethod.POST)
      public String login(String username, String password, String code, boolean rememberme,
                          Model model, /*HttpSession session, */HttpServletResponse response,
                          @CookieValue("kaptchaOwner") String kaptchaOwner) {
          // 检查验证码
          // String kaptcha = (String) session.getAttribute("kaptcha");
          String kaptcha = null;
          if (StringUtils.isNotBlank(kaptchaOwner)) {
              String redisKey = RedisKeyUtil.getKaptchaKey(kaptchaOwner);
              kaptcha = (String) redisTemplate.opsForValue().get(redisKey);
          }
          if (StringUtils.isBlank(kaptcha) || StringUtils.isBlank(code) || !kaptcha.equalsIgnoreCase(code)) {
              model.addAttribute("codeMsg", "验证码不正确!");
              //回到登录页面
              return "/site/login";
          }
  
          // 检查账号,密码
          int expiredSeconds = rememberme ? REMEMBER_EXPIRED_SECONDS : DEFAULT_EXPIRED_SECONDS;
          Map<String, Object> map = userService.login(username, password, expiredSeconds);
          if (map.containsKey("ticket")) {
              //将ticket保存到cookie中，setPath表示设置到/community下，项目路径下就全部生效了
              Cookie cookie = new Cookie("ticket", map.get("ticket").toString());
              cookie.setPath(contextPath);
              cookie.setMaxAge(expiredSeconds);
              response.addCookie(cookie);
              return "redirect:/index";
          } else {
              model.addAttribute("usernameMsg", map.get("usernameMsg"));
              model.addAttribute("passwordMsg", map.get("passwordMsg"));
              return "/site/login";
          }
      }
  ```

  

- 创建 `logout` 方法，判断验证码正确后调用  UserServce 的`logout` 方法，跳转至登录页(后期加入Spring Security)。

  ``` java
  @RequestMapping(path = "/logout", method = RequestMethod.GET)
      public String logout(@CookieValue("ticket") String ticket) {
          userService.logout(ticket);
          SecurityContextHolder.clearContext();
          return "redirect:/login";
      }
  ```

  

在 `login.html` 绑定登录链接，`index.html` 绑定退出登录链接。

------

### 显示登录信息

![image-20221109220348991](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109220348991.png)

Interceptor有三个方法，在控制器前执行，在控制器后执行，在TemplateEngine（模板引擎）之后执行

``` java
@Component
public class AlphaInterceptor implements HandlerInterceptor {

    private static final Logger logger = LoggerFactory.getLogger(AlphaInterceptor.class);

    // 在Controller之前执行
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        logger.debug("preHandle: " + handler.toString());
        return true;
    }

    // 在Controller之后执行
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        logger.debug("postHandle: " + handler.toString());
    }

    // 在TemplateEngine之后执行
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        logger.debug("afterCompletion: " + handler.toString());
    }
}
```

将interceptor降入到WebMvcConfig（implement WebMvcConfigurer实现Spring Mvc中的方法）中，实现真正的配置

![image-20221109223527815](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109223527815.png)

创建 CookieUtil 工具类，通过 name 查询对应 cookie 的 value。

``` java
public class CookieUtil {

    public static String getValue(HttpServletRequest request, String name) {
        if (request == null || name == null) {
            throw new IllegalArgumentException("参数为空!");
        }
        Cookie[] cookies = request.getCookies();
        if (cookies != null) {
            for (Cookie cookie : cookies) {
                if (cookie.getName().equals(name)) {
                    return cookie.getValue();
                }
            }
        }
        return null;
    }

}
```



在 UserService 中新增 `findLoginTicket` 方法，根据 ticket 查询 LoginTicket(优化后从redis中查询登录凭证)。

``` java
public LoginTicket findLoginTicket(String ticket) {
//        return loginTicketMapper.selectByTicket(ticket);
        String redisKey = RedisKeyUtil.getTicketKey(ticket);
        return (LoginTicket) redisTemplate.opsForValue().get(redisKey);
    }
```



创建 HostHolder 类用来模拟 session 的功能，利用 ThreadLocal 实现，存储用户信息。

``` java
@Component
public class HostHolder {

    private ThreadLocal<User> users = new ThreadLocal<>();

    public void setUser(User user) {
        users.set(user);
    }

    public User getUser() {
        return users.get();
    }

    public void clear() {
        users.remove();
    }

}
```



创建 LoginTicketInterceptor 拦截器，实现 HandlerInterceptor 接口。

- 在 `preHandle` 方法中通过 CookieUtil 的 `getValue` 方法查询是否有凭证 cookie，如果有则通过 UserService 的  `findloginTicket` 方法查询用户 ID，再通过用户 ID 查询用户。最后将用户放入 hostHolder 中。 
- 在 `postHandle` 方法中通过 hostHolder 的 `get` 方法获取用户，并将其存入视图中。 
- 在 `afterCompletion` 方法中清除 hostHolder 中存放的用户信息。 

``` java
@Component
public class LoginTicketInterceptor implements HandlerInterceptor {

    @Autowired
    private UserService userService;

    @Autowired
    private HostHolder hostHolder;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // 从cookie中获取凭证
        String ticket = CookieUtil.getValue(request, "ticket");

        if (ticket != null) {
            // 查询凭证
            LoginTicket loginTicket = userService.findLoginTicket(ticket);
            // 检查凭证是否有效
            if (loginTicket != null && loginTicket.getStatus() == 0 && loginTicket.getExpired().after(new Date())) {
                // 根据凭证查询用户
                User user = userService.findUserById(loginTicket.getUserId());
                // 在本次请求中持有用户
                hostHolder.setUser(user);
                // 构建用户认证的结果,并存入SecurityContext,以便于Security进行授权.
                Authentication authentication = new UsernamePasswordAuthenticationToken(
                        user, user.getPassword(), userService.getAuthorities(user.getId()));
                SecurityContextHolder.setContext(new SecurityContextImpl(authentication));
            }
        }

        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        User user = hostHolder.getUser();
        if (user != null && modelAndView != null) {
            modelAndView.addObject("loginUser", user);
        }
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        hostHolder.clear();
        SecurityContextHolder.clearContext();
    }
}
```



在WebMvcConfig 配置类，配置 LoginTicketInterceptor，拦截除了静态资源之外的所有路径。

```java
registry.addInterceptor(loginTicketInterceptor)
        .excludePathPatterns("/**/*.css", "/**/*.js", "/**/*.png", "/**/*.jpg", "/**/*.jpeg");
```

------

### 上传头像

![image-20221109225658330](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109225658330.png)

在 UserService 新增 `updateHeader` 方法，更改指定用户的头像。

``` java
public int updateHeader(int userId, String headerUrl) {
//        return userMapper.updateHeader(userId, headerUrl);
        int rows = userMapper.updateHeader(userId, headerUrl);
        clearCache(userId);
        return rows;
    }
```



创建 UserController

- 新增 `getSettingPage` 方法访问账户设置 `setting.html` ，并在 `index.html` 的账号设置按钮关联该链接(改进：将头像上传到七牛云)。

  ``` java
  @LoginRequired
      @RequestMapping(path = "/setting", method = RequestMethod.GET)
      public String getSettingPage(Model model) {
          // 上传文件名称
          String fileName = CommunityUtil.generateUUID();
          // 设置响应信息
          StringMap policy = new StringMap();
          policy.put("returnBody", CommunityUtil.getJSONString(0));
          // 生成上传凭证
          Auth auth = Auth.create(accessKey, secretKey);
          String uploadToken = auth.uploadToken(headerBucketName, fileName, 3600, policy);
  
          model.addAttribute("uploadToken", uploadToken);
          model.addAttribute("fileName", fileName);
  
          return "/site/setting";
      }
  ```

  

- 新增 `uploadHeader` 方法更新用户头像，如果上传出现错误将错误信息存在 Model 对象中。

  ```
  // 废弃,这是存在本地的，后续不用该方法
  @LoginRequired
  @RequestMapping(path = "/upload", method = RequestMethod.POST)
  public String uploadHeader(MultipartFile headerImage, Model model) {
      if (headerImage == null) {
          model.addAttribute("error", "您还没有选择图片!");
          return "/site/setting";
      }
  
      String fileName = headerImage.getOriginalFilename();
      String suffix = fileName.substring(fileName.lastIndexOf("."));
      if (StringUtils.isBlank(suffix)) {
          model.addAttribute("error", "文件的格式不正确!");
          return "/site/setting";
      }
  
      // 生成随机文件名
      fileName = CommunityUtil.generateUUID() + suffix;
      // 确定文件存放的路径
      File dest = new File(uploadPath + "/" + fileName);
      try {
          // 存储文件
          headerImage.transferTo(dest);
      } catch (IOException e) {
          logger.error("上传文件失败: " + e.getMessage());
          throw new RuntimeException("上传文件失败,服务器发生异常!", e);
      }
  
      // 更新当前用户的头像的路径(web访问路径)
      // http://localhost:8080/community/user/header/xxx.png
      User user = hostHolder.getUser();
      String headerUrl = domain + contextPath + "/user/header/" + fileName;
      userService.updateHeader(user.getId(), headerUrl);
  
      return "redirect:/index";
  }
  ```

  如果没有错误，生成一个文件对象 dest，利用 MultipartFile 接口的 `transferTo` 方法将用户上传文件导入 dest，并从 hostHolder 中取出用户，更新用户的头像路径。

- 新增 `getHeader` 方法获取用户头像，利用文件输入流读取图片数据，利用 HttpServletResponse 的字节输出流再进行输出。

  ``` java
  // 废弃
      @RequestMapping(path = "/header/{fileName}", method = RequestMethod.GET)
      public void getHeader(@PathVariable("fileName") String fileName, HttpServletResponse response) {
          // 服务器存放路径
          fileName = uploadPath + "/" + fileName;
          // 文件后缀
          String suffix = fileName.substring(fileName.lastIndexOf("."));
          // 响应图片
          response.setContentType("image/" + suffix);
          try (
                  FileInputStream fis = new FileInputStream(fileName);
                  OutputStream os = response.getOutputStream();
          ) {
              byte[] buffer = new byte[1024];
              int b = 0;
              while ((b = fis.read(buffer)) != -1) {
                  os.write(buffer, 0, b);
              }
          } catch (IOException e) {
              logger.error("读取头像失败: " + e.getMessage());
          }
      }
  ```

  

调整 `setting.html` 的 form 表单， method="post"，enctype="multipart/form-data"，并设置提交路径。(后续将文件上传到七牛云)

![image-20221109232415570](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109232415570.png)



------

### 修改密码

在 UserService 中新增 `changePassword` 方法，判断原密码是否正确，正确则修改密码并返回 1，否则返回 0。

在 UserController 中新增 `changePassword` 方法，根据 UserService 的  `changePassword` 方法的返回值判断原密码是否成功修改，封装为 JSON 数据并返回。

在 `setting.html` 中

- 首先在前端判断两次输入的新密码是否一致，如果不一致不允许点击提交并显示错误信息。 
- 利用 ajax 向 UserController 的  `changePassword` 方法发送 POST 请求，得到 JSON 数据并解析，如果状态码为 0 提示错误，如果状态码为 1 弹出修改成功提示。 

【问题】js 的虚拟路径问题，需要加上 `../`。

【问题】使用 ajax 请求时，表单按钮类型必须是 button，不能是 submit，否则 405 报错。

【问题】使用 ajax 请求时，Controller 中方法的返回值必须是 JSON 数据，并且需要加上 `@ResponseBody`。

【问题】使用 ajax 请求时，回调函数需要先对返回的 JSON 数据进行解析再使用。

------

### 检查登录状态

![image-20221109233111654](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109233111654.png)

* Target：作用在哪些位置：方法、类
* Retention：生效时间：运行时有效还是编译时有效
* Document：在生成文档时要不要带上该注解
* Inherited：用于继承，要不要继承父类注解

利用拦截器，实现只处理带有自定义注解的方法，防止用户在未登录情况下通过 url 访问没有权限的页面。创建 `@LoginRequired` 自定义注解，作用范围在方法上，有效期为运行时。

``` java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LoginRequired {
}
```



在 UserController 中需要在登录状态下调用的方法，访问设置页面、修改密码、上传头像等加上该自定义注解。

创建 LoginRequiredInterceptor 拦截器，在 `preHandle` 方法中判断方法是否加了 `@LoginRequired` 注解（利用注解），如果加了注解并且此时从 hostHolder 中获取不到用户则拒绝访问,并重定向到登陆页面。

``` java
@Component
public class LoginRequiredInterceptor implements HandlerInterceptor {

    @Autowired
    private HostHolder hostHolder;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        if (handler instanceof HandlerMethod) {
            HandlerMethod handlerMethod = (HandlerMethod) handler;
            Method method = handlerMethod.getMethod();
            LoginRequired loginRequired = method.getAnnotation(LoginRequired.class);
            if (loginRequired != null && hostHolder.getUser() == null) {
                response.sendRedirect(request.getContextPath() + "/login");
                return false;
            }
        }
        return true;
    }
}
```



在 WebMvcConfig 配置类配置 LoginRequiredInterceptor，拦截除了静态资源之外的所有路径。

作者：是瑶瑶公主呀
链接：https://www.nowcoder.com/discuss/481089?type=5
来源：牛客网



## 5.开发核心功能

### 敏感词过滤

![image-20221109234844991](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109234844991.png)

利用字典树数据结构解决。一共3个指针，第一个指向树的根节点，第二个作为字串的开头，第三个作为字串的结尾。

![image-20221109235759742](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221109235759742.png)

创建 SensitiveFilter 类

- 创建静态内部类 TrieNode ，通过 boolean 类型的结束符判断是否匹配到关键字尾部。 

- 利用 `@PostConstruct` 注解，在构造方法执行后初始化字典树。 

- 添加 `filter` 方法，利用双指针进行匹配，过滤敏感词。 

  ``` java
  @Component
  public class SensitiveFilter {
  
      private static final Logger logger = LoggerFactory.getLogger(SensitiveFilter.class);
  
      // 替换符
      private static final String REPLACEMENT = "***";
  
      // 根节点
      private TrieNode rootNode = new TrieNode();
  
      @PostConstruct
      public void init() {
          try (
                  InputStream is = this.getClass().getClassLoader().getResourceAsStream("sensitive-words.txt");
                  BufferedReader reader = new BufferedReader(new InputStreamReader(is));
          ) {
              String keyword;
              while ((keyword = reader.readLine()) != null) {
                  // 添加到前缀树
                  this.addKeyword(keyword);
              }
          } catch (IOException e) {
              logger.error("加载敏感词文件失败: " + e.getMessage());
          }
      }
  
      // 将一个敏感词添加到前缀树中
      private void addKeyword(String keyword) {
          TrieNode tempNode = rootNode;
          for (int i = 0; i < keyword.length(); i++) {
              char c = keyword.charAt(i);
              TrieNode subNode = tempNode.getSubNode(c);
  
              if (subNode == null) {
                  // 初始化子节点
                  subNode = new TrieNode();
                  tempNode.addSubNode(c, subNode);
              }
  
              // 指向子节点,进入下一轮循环
              tempNode = subNode;
  
              // 设置结束标识
              if (i == keyword.length() - 1) {
                  tempNode.setKeywordEnd(true);
              }
          }
      }
  
      /**
       * 过滤敏感词
       *
       * @param text 待过滤的文本
       * @return 过滤后的文本
       */
      public String filter(String text) {
          if (StringUtils.isBlank(text)) {
              return null;
          }
  
          // 指针1
          TrieNode tempNode = rootNode;
          // 指针2
          int begin = 0;
          // 指针3
          int position = 0;
          // 结果
          StringBuilder sb = new StringBuilder();
  
          while (position < text.length()) {
              char c = text.charAt(position);
  
              // 跳过符号
              if (isSymbol(c)) {
                  // 若指针1处于根节点,将此符号计入结果,让指针2向下走一步
                  if (tempNode == rootNode) {
                      sb.append(c);
                      begin++;
                  }
                  // 无论符号在开头或中间,指针3都向下走一步
                  position++;
                  continue;
              }
  
              // 检查下级节点
              tempNode = tempNode.getSubNode(c);
              if (tempNode == null) {
                  // 以begin开头的字符串不是敏感词
                  sb.append(text.charAt(begin));
                  // 进入下一个位置
                  position = ++begin;
                  // 重新指向根节点
                  tempNode = rootNode;
              } else if (tempNode.isKeywordEnd()) {
                  // 发现敏感词,将begin~position字符串替换掉
                  sb.append(REPLACEMENT);
                  // 进入下一个位置
                  begin = ++position;
                  // 重新指向根节点
                  tempNode = rootNode;
              } else {
                  // 检查下一个字符
                  position++;
              }
          }
  
          // 将最后一批字符计入结果
          sb.append(text.substring(begin));
  
          return sb.toString();
      }
  
      // 判断是否为符号
      private boolean isSymbol(Character c) {
          // 0x2E80~0x9FFF 是东亚文字范围
          return !CharUtils.isAsciiAlphanumeric(c) && (c < 0x2E80 || c > 0x9FFF);
      }
  
      // 前缀树
      private class TrieNode {
  
          // 关键词结束标识
          private boolean isKeywordEnd = false;
  
          // 子节点(key是下级字符,value是下级节点)
          private Map<Character, TrieNode> subNodes = new HashMap<>();
  
          public boolean isKeywordEnd() {
              return isKeywordEnd;
          }
  
          public void setKeywordEnd(boolean keywordEnd) {
              isKeywordEnd = keywordEnd;
          }
  
          // 添加子节点
          public void addSubNode(Character c, TrieNode node) {
              subNodes.put(c, node);
          }
  
          // 获取子节点
          public TrieNode getSubNode(Character c) {
              return subNodes.get(c);
          }
  
      }
  
  }
  ```

------

### 发布帖子

![image-20221110002638925](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221110002638925.png)

引入 fastjson 依赖，在 CommunityUtil 中新增 `getJSONString` 方法封装 JSON 信息。

``` java
public static String getJSONString(int code, String msg, Map<String, Object> map) {
        JSONObject json = new JSONObject();
        json.put("code", code);
        json.put("msg", msg);
        if (map != null) {
            for (String key : map.keySet()) {
                json.put(key, map.get(key));
            }
        }
        return json.toJSONString();
    }

    public static String getJSONString(int code, String msg) {
        return getJSONString(code, msg, null);
    }

    public static String getJSONString(int code) {
        return getJSONString(code, null, null);
    }
```



在 DisscussPostMapper 接口新增 `insertDiscussPost` 方法，并在 `disscusspost-mapper.xml` 配置 insert 语句。

``` xml
<insert id="insertDiscussPost" parameterType="com.nowcoder.community.entity.DiscussPost" keyProperty="id">
        insert into discuss_post(<include refid="insertFields"></include>)
        values(#{userId},#{title},#{content},#{type},#{status},#{createTime},#{commentCount},#{score})
    </insert>
```



在 DiscussPostService 新增 `addDiscussPost` 方法调用 DisscussPostMapper 的 `insertDiscussPost` 方法，其中需要进行对标题内容和发帖内容进行 HTML 转义以及过滤敏感词。

``` java
public int addDiscussPost(DiscussPost post) {
        if (post == null) {
            throw new IllegalArgumentException("参数不能为空!");
        }

        // 转义HTML标记,将类似与<script></script>当作寻常字符串
        post.setTitle(HtmlUtils.htmlEscape(post.getTitle()));
        post.setContent(HtmlUtils.htmlEscape(post.getContent()));
        // 过滤敏感词
        post.setTitle(sensitiveFilter.filter(post.getTitle()));
        post.setContent(sensitiveFilter.filter(post.getContent()));

        return discussPostMapper.insertDiscussPost(post);
    }
```



创建 DiscussPostController 类，新增 `addDiscussPost` 方法，调用 DiscussPostService 的 `addDiscussPost` 方法发帖(优化：利用消息队列来异步处理)。

``` java
@RequestMapping(path = "/add", method = RequestMethod.POST)
    @ResponseBody
    public String addDiscussPost(String title, String content) {
        User user = hostHolder.getUser();
        if (user == null) {
            return CommunityUtil.getJSONString(403, "你还没有登录哦!");
        }

        DiscussPost post = new DiscussPost();
        post.setUserId(user.getId());
        post.setTitle(title);
        post.setContent(content);
        post.setCreateTime(new Date());
        discussPostService.addDiscussPost(post);

        // 触发发帖事件
        Event event = new Event()
                .setTopic(TOPIC_PUBLISH)
                .setUserId(user.getId())
                .setEntityType(ENTITY_TYPE_POST)
                .setEntityId(post.getId());
        eventProducer.fireEvent(event);

        // 计算帖子分数
        String redisKey = RedisKeyUtil.getPostScoreKey();
        redisTemplate.opsForSet().add(redisKey, post.getId());

        // 报错的情况,将来统一处理.
        return CommunityUtil.getJSONString(0, "发布成功!");
    }
```



在 `index.html` 中为发帖按钮绑定函数，利用 Ajax 向 DiscussPostController 的 `addDiscussPost` 方法发送 POST 请求。

``` xml
// 获取标题和内容
	var title = $("#recipient-name").val();
	var content = $("#message-text").val();
	// 发送异步请求(POST)
	$.post(
	    CONTEXT_PATH + "/discuss/add",
	    {"title":title,"content":content},
	    function(data) {
	        data = $.parseJSON(data);
	        // 在提示框中显示返回消息
	        $("#hintBody").text(data.msg);
	        // 显示提示框
            $("#hintModal").modal("show");
            // 2秒后,自动隐藏提示框
            setTimeout(function(){
                $("#hintModal").modal("hide");
                // 刷新页面
                if(data.code == 0) {
                    window.location.reload();
                }
            }, 2000);
	    }
	);
```



------

### 显示帖子内容

![image-20221110010411183](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221110010411183.png)

在 DisscussPostMapper 接口新增 `selectDiscussPostById` 方法，在 `disscusspost-mapper.xml` 配置 select 语句。

``` xml
<select id="selectDiscussPostById" resultType="com.nowcoder.community.entity.DiscussPost">
        select <include refid="selectFields"></include>
        from discuss_post
        where id = #{id}
    </select>
```



在 DiscussPostService 新增 `findDiscussPostById` 方法调用 DisscussPostMapper 的 `selectDiscussPostById` 方法。

``` java
public DiscussPost findDiscussPostById(int id) {
        return discussPostMapper.selectDiscussPostById(id);
    }
```



在 DiscussPostController 新增 `getDiscussPost` 方法，调用 DiscussPostService 的 `findDiscussPostById` 方法查询帖子内容，将 DiscussPost 对象和 User 对象（通过 userId 查询，不在 DAO 层关联查询）数据存放到 Model 对象，返回模板 `discuss-detail`。(一大串关联查询，所采取的做法是挨个查，先查帖子，再根据帖子id查用户，再查评论，点赞数，组装成一个map)

``` java
@RequestMapping(path = "/detail/{discussPostId}", method = RequestMethod.GET)
    public String getDiscussPost(@PathVariable("discussPostId") int discussPostId, Model model, Page page) {
        // 帖子
        DiscussPost post = discussPostService.findDiscussPostById(discussPostId);
        model.addAttribute("post", post);
        // 作者
        User user = userService.findUserById(post.getUserId());
        model.addAttribute("user", user);
        // 点赞数量
        long likeCount = likeService.findEntityLikeCount(ENTITY_TYPE_POST, discussPostId);
        model.addAttribute("likeCount", likeCount);
        // 点赞状态
        int likeStatus = hostHolder.getUser() == null ? 0 :
                likeService.findEntityLikeStatus(hostHolder.getUser().getId(), ENTITY_TYPE_POST, discussPostId);
        model.addAttribute("likeStatus", likeStatus);

        // 评论分页信息
        page.setLimit(5);
        page.setPath("/discuss/detail/" + discussPostId);
        page.setRows(post.getCommentCount());

        // 评论: 给帖子的评论
        // 回复: 给评论的评论
        // 评论列表
        List<Comment> commentList = commentService.findCommentsByEntity(
                ENTITY_TYPE_POST, post.getId(), page.getOffset(), page.getLimit());
        // 评论VO列表
        List<Map<String, Object>> commentVoList = new ArrayList<>();
        if (commentList != null) {
            for (Comment comment : commentList) {
                // 评论VO
                Map<String, Object> commentVo = new HashMap<>();
                // 评论
                commentVo.put("comment", comment);
                // 作者
                commentVo.put("user", userService.findUserById(comment.getUserId()));
                // 点赞数量
                likeCount = likeService.findEntityLikeCount(ENTITY_TYPE_COMMENT, comment.getId());
                commentVo.put("likeCount", likeCount);
                // 点赞状态
                likeStatus = hostHolder.getUser() == null ? 0 :
                        likeService.findEntityLikeStatus(hostHolder.getUser().getId(), ENTITY_TYPE_COMMENT, comment.getId());
                commentVo.put("likeStatus", likeStatus);

                // 回复列表
                List<Comment> replyList = commentService.findCommentsByEntity(
                        ENTITY_TYPE_COMMENT, comment.getId(), 0, Integer.MAX_VALUE);
                // 回复VO列表
                List<Map<String, Object>> replyVoList = new ArrayList<>();
                if (replyList != null) {
                    for (Comment reply : replyList) {
                        Map<String, Object> replyVo = new HashMap<>();
                        // 回复
                        replyVo.put("reply", reply);
                        // 作者
                        replyVo.put("user", userService.findUserById(reply.getUserId()));
                        // 回复目标
                        User target = reply.getTargetId() == 0 ? null : userService.findUserById(reply.getTargetId());
                        replyVo.put("target", target);
                        // 点赞数量
                        likeCount = likeService.findEntityLikeCount(ENTITY_TYPE_COMMENT, reply.getId());
                        replyVo.put("likeCount", likeCount);
                        // 点赞状态
                        likeStatus = hostHolder.getUser() == null ? 0 :
                                likeService.findEntityLikeStatus(hostHolder.getUser().getId(), ENTITY_TYPE_COMMENT, reply.getId());
                        replyVo.put("likeStatus", likeStatus);

                        replyVoList.add(replyVo);
                    }
                }
                commentVo.put("replys", replyVoList);

                // 回复数量
                int replyCount = commentService.findCommentCount(ENTITY_TYPE_COMMENT, comment.getId());
                commentVo.put("replyCount", replyCount);

                commentVoList.add(commentVo);
            }
        }

        model.addAttribute("comments", commentVoList);

        return "/site/discuss-detail";
    }
```



在 `discuss-detail.html` 取出 Model 对象存放的数据绑定到对应组件显示。

------

### 事务管理

![image-20221110185337710](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221110185337710.png)

在AlphaController中添加两个测试方法。

* 声明型事务（设置隔离级别和传播行为）

  ![image-20221110190642619](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221110190642619.png)

* 编程式事务(通过transactionTemplate或TransactionManager来实现)

  ``` java
  public Object save2() {
          transactionTemplate.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
          transactionTemplate.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
  
          return transactionTemplate.execute(new TransactionCallback<Object>() {
              @Override
              public Object doInTransaction(TransactionStatus status) {
                  // 新增用户
                  User user = new User();
                  user.setUsername("beta");
                  user.setSalt(CommunityUtil.generateUUID().substring(0, 5));
                  user.setPassword(CommunityUtil.md5("123" + user.getSalt()));
                  user.setEmail("beta@qq.com");
                  user.setHeaderUrl("http://image.nowcoder.com/head/999t.png");
                  user.setCreateTime(new Date());
                  userMapper.insertUser(user);
  
                  // 新增帖子
                  DiscussPost post = new DiscussPost();
                  post.setUserId(user.getId());
                  post.setTitle("你好");
                  post.setContent("我是新人!");
                  post.setCreateTime(new Date());
                  discussPostMapper.insertDiscussPost(post);
  
                  Integer.valueOf("abc");
  
                  return "ok";
              }
          });
      }
  ```

  

------



### 显示评论（comment 表）

![image-20221111151434355](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221111151434355.png)

创建 comment 表对应的实体类 Comment。

创建 CommentMapper 接口

- 新增 `selectCommentsByEntity` 方法，根据实体查询一页的评论数据。 

- 新增 `selectCountByEntity` 方法，根据实体查询评论的数量。 

- 在 `comment-mapper.xml` 配置 select 语句。 

  ``` java
  <select id="selectCommentsByEntity" resultType="com.nowcoder.community.entity.Comment">
          select <include refid="selectFields"></include>
          from comment
          where status = 0
          and entity_type = #{entityType}
          and entity_id = #{entityId}
          order by create_time asc
          limit #{offset}, #{limit}
      </select>
  
      <select id="selectCountByEntity" resultType="int">
          select count(id)
          from comment
          where status = 0
          and entity_type = #{entityType}
          and entity_id = #{entityId}
      </select>
  ```

  

创建 CommentService 类

- 新增 `findCommentByEntity` 方法，调用 CommentMapper 的 `selectCommentByEntity` 方法。 

- 新增 `findCommentCount` 方法，调用 CommentMapper 的 `selectCountByEntity` 方法。 

  ``` java
  public List<Comment> findCommentsByEntity(int entityType, int entityId, int offset, int limit) {
          return commentMapper.selectCommentsByEntity(entityType, entityId, offset, limit);
      }
  
      public int findCommentCount(int entityType, int entityId) {
          return commentMapper.selectCountByEntity(entityType, entityId);
      }
  ```

  

在 DiscussPostController 的 `getDiscussPost` 方法中增加查询帖子评论和回复的逻辑，将结果存储在 Model 对象。(嵌套查询，组装成list集合，这里没有创建真正的vo对象)

------

### 添加评论

![image-20221111153925155](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221111153925155.png)

**注意事务管理，增加评论和更新贴子要在一个事务里**

在 CommentMapper 接口新增 `insertComment` 方法，添加评论数据，在 `comment-mapper` 配置对应 sql。

在 DiscussPostMapper 接口新增 `updateCommentCount` 方法，增加评论数量，在 `discusspost-mapper` 配置对应 sql。（DiscussPost表中有冗余字段，冗余了commentcount字段，可以直接查出评论数量）

在 DiscussPostService 类新增 `updateCommentCount` 方法，调用 DiscussPostMapper 的 `updateCommentCount` 方法。

在 CommentService 类新增 `addComment` 方法，调用 CommentMapper 的 `insertComment` 新增评论，并调用 DiscussPostService 的 `updateCommentCount` 更新评论数量，使用 `@Transactional` 注解保证事务。

``` java
@Transactional(isolation = Isolation.READ_COMMITTED, propagation = Propagation.REQUIRED)
    public int addComment(Comment comment) {
        if (comment == null) {
            throw new IllegalArgumentException("参数不能为空!");
        }

        // 添加评论
        comment.setContent(HtmlUtils.htmlEscape(comment.getContent()));
        comment.setContent(sensitiveFilter.filter(comment.getContent()));
        int rows = commentMapper.insertComment(comment);

        // 更新帖子评论数量
        if (comment.getEntityType() == ENTITY_TYPE_POST) {
            int count = commentMapper.selectCountByEntity(comment.getEntityType(), comment.getEntityId());
            discussPostService.updateCommentCount(comment.getEntityId(), count);
        }

        return rows;
    }
```



创建 CommentController 类，新增 `addComment` 方法，从 hostHolder 获取用户信息，然后调用 CommentService 的 `addComment` 方法添加评论。

【问题】sql 的 xml 文件中绑定参数时，应传入实体类属性名，拼错成数据库字段名（entityId 写成 entity_id）。

------

### 显示私信列表 （message 表）

![image-20221111162234346](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221111162234346.png)

创建对应 message 表的实体类 Message。

| 字段            | 类型      | 备注                                                         |
| --------------- | --------- | ------------------------------------------------------------ |
| id              | int       | 主键、自增                                                   |
| from_id         | int       | 发消息的 id，创建索引（id时一表示系统消息）                  |
| to_id           | int       | 收消息的 id，创建索引                                        |
| conversation_id | varchar   | 会话 id，由通信双方 id 拼接，创建索引 （总是小在前大在后；例如101_102方便统计） |
| content         | text      | 消息内容                                                     |
| status          | int       | 消息状态：0 未读、1 已读、2 删除                             |
| create_time     | timestamp | 消息发送时间                                                 |

创建 MessageMapper 接口，增加查询会话列表、会话数量、私信列表、私信数量、未读私信数量等方法，在 `message-mapper.xml` 中配置对应的 sql。

``` java
<select id="selectConversations" resultType="com.nowcoder.community.entity.Message">
        select <include refid="selectFields"></include>
        from message
        where id in (
            select max(id) from message
            where status != 2
            and from_id != 1
            and (from_id = #{userId} or to_id = #{userId})
            group by conversation_id
        )
        order by id desc
        limit #{offset}, #{limit}
    </select>

    <select id="selectConversationCount" resultType="int">
        select count(m.maxid) from (
            select max(id) as maxid from message
            where status != 2
            and from_id != 1
            and (from_id = #{userId} or to_id = #{userId})
            group by conversation_id
        ) as m
    </select>

    <select id="selectLetters" resultType="com.nowcoder.community.entity.Message">
        select <include refid="selectFields"></include>
        from message
        where status != 2
        and from_id != 1
        and conversation_id = #{conversationId}
        order by id desc
        limit #{offset}, #{limit}
    </select>

    <select id="selectLetterCount" resultType="int">
        select count(id)
        from message
        where status != 2
        and from_id != 1
        and conversation_id = #{conversationId}
    </select>

    <select id="selectLetterUnreadCount" resultType="int">
        select count(id)
        from message
        where status = 0
        and from_id != 1
        and to_id = #{userId}
        <if test="conversationId!=null">
            and conversation_id = #{conversationId}
        </if>
    </select>
```



创建 MessageService，调用 MessageMapper 中的方法。

创建 MessgaeController

- 新增 `getLetterList` 方法，将会话列表信息存储到 Model 对象，返回 `letter` 视图。 

  ```java
  // 私信列表
  @RequestMapping(path = "/letter/list", method = RequestMethod.GET)
  public String getLetterList(Model model, Page page) {
      User user = hostHolder.getUser();
      // 分页信息
      page.setLimit(5);
      page.setPath("/letter/list");
      page.setRows(messageService.findConversationCount(user.getId()));
  
      // 会话列表
      List<Message> conversationList = messageService.findConversations(
              user.getId(), page.getOffset(), page.getLimit());
      List<Map<String, Object>> conversations = new ArrayList<>();
      if (conversationList != null) {
          for (Message message : conversationList) {
              Map<String, Object> map = new HashMap<>();
              map.put("conversation", message);
              map.put("letterCount", messageService.findLetterCount(message.getConversationId()));
              map.put("unreadCount", messageService.findLetterUnreadCount(user.getId(), message.getConversationId()));
              int targetId = user.getId() == message.getFromId() ? message.getToId() : message.getFromId();
              map.put("target", userService.findUserById(targetId));
  
              conversations.add(map);
          }
      }
      model.addAttribute("conversations", conversations);
  
      // 查询未读消息数量
      int letterUnreadCount = messageService.findLetterUnreadCount(user.getId(), null);
      model.addAttribute("letterUnreadCount", letterUnreadCount);
      int noticeUnreadCount = messageService.findNoticeUnreadCount(user.getId(), null);
      model.addAttribute("noticeUnreadCount", noticeUnreadCount);
  
      return "/site/letter";
  }
  ```

- 新增 `getLetterDetail` 方法，将每个会话具体的私信信息存储到 Model 对象，返回 `letter-datail` 视图。 

  ```java
  @RequestMapping(path = "/letter/detail/{conversationId}", method = RequestMethod.GET)
  public String getLetterDetail(@PathVariable("conversationId") String conversationId, Page page, Model model) {
      // 分页信息
      page.setLimit(5);
      page.setPath("/letter/detail/" + conversationId);
      page.setRows(messageService.findLetterCount(conversationId));
  
      // 私信列表
      List<Message> letterList = messageService.findLetters(conversationId, page.getOffset(), page.getLimit());
      List<Map<String, Object>> letters = new ArrayList<>();
      if (letterList != null) {
          for (Message message : letterList) {
              Map<String, Object> map = new HashMap<>();
              map.put("letter", message);
              map.put("fromUser", userService.findUserById(message.getFromId()));
              letters.add(map);
          }
      }
      model.addAttribute("letters", letters);
  
      // 私信目标
      model.addAttribute("target", getLetterTarget(conversationId));
  
      // 设置已读
      List<Integer> ids = getLetterIds(letterList);
      if (!ids.isEmpty()) {
          messageService.readMessage(ids);
      }
  
      return "/site/letter-detail";
  }
  
  private User getLetterTarget(String conversationId) {
      String[] ids = conversationId.split("_");
      int id0 = Integer.parseInt(ids[0]);
      int id1 = Integer.parseInt(ids[1]);
  
      if (hostHolder.getUser().getId() == id0) {
          return userService.findUserById(id1);
      } else {
          return userService.findUserById(id0);
      }
  }
  ```

------

### 发送私信

![image-20221111181545689](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221111181545689.png)

在 MessageMapper 

- 新增 `insertMessage` 方法插入私信记录，在 `message-mapper.xml`  配置 insert 语句。 

- 新增 `updateMessgae` 方法修改私信状态，在 `message-mapper.xml`  配置 update 语句，利用 foreach 动态 sql。 

  ```xml
  <insert id="insertMessage" parameterType="com.nowcoder.community.entity.Message" keyProperty="id">
      insert into message(<include refid="insertFields"></include>)
      values(#{fromId},#{toId},#{conversationId},#{content},#{status},#{createTime})
  </insert>
  //foreach动态拼接sql
  <update id="updateStatus">
      update message set status = #{status}
      where id in
      <foreach collection="ids" item="id" open="(" separator="," close=")">
          #{id}
      </foreach>
  </update>
  ```

在 MessageService

- 新增 `addMessage` 发送私信方法，过滤敏感词后，调用 MessageMapper 的 `insertMessage` 。

- 新增 `readMessage` 方法读取信息，调用MessageMapper 的 `updateMessgae` 更新私信的状态为 1。

  ```java
  public int addMessage(Message message) {
      message.setContent(HtmlUtils.htmlEscape(message.getContent()));
      message.setContent(sensitiveFilter.filter(message.getContent()));
      return messageMapper.insertMessage(message);
  }
  
  public int readMessage(List<Integer> ids) {
      return messageMapper.updateStatus(ids, 1);
  }
  ```

在 MessageController 

- 新增 `getLetterIds` 方法，将私信集合中未读私信的 id 添加到 List 集合并返回，在 `getLetterDetail` 方法调用该方法设置已读。

  ```java
  private List<Integer> getLetterIds(List<Message> letterList) {
      List<Integer> ids = new ArrayList<>();
  
      if (letterList != null) {
          for (Message message : letterList) {
              //只有该用户是消息接收者时后才会改变该消息的状态
              if (hostHolder.getUser().getId() == message.getToId() && message.getStatus() == 0) {
                  ids.add(message.getId());
              }
          }
      }
      return ids;
  }
  ```

- 新增 `sendLetter` 发送私信方法，设置私信信息后调用 MessageService 的 `addMessage` 发送。

- ```java
  @RequestMapping(path = "/letter/send", method = RequestMethod.POST)
  @ResponseBody
  public String sendLetter(String toName, String content) {
      User target = userService.findUserByName(toName);
      if (target == null) {
          return CommunityUtil.getJSONString(1, "目标用户不存在!");
      }
      Message message = new Message();
      message.setFromId(hostHolder.getUser().getId());
      message.setToId(target.getId());
      //将ConversationId始终设置成前小后大  101_102
      if (message.getFromId() < message.getToId()) {
          message.setConversationId(message.getFromId() + "_" + message.getToId());
      } else {
          message.setConversationId(message.getToId() + "_" + message.getFromId());
      }
      message.setContent(content);
      message.setCreateTime(new Date());
      messageService.addMessage(message);
      return CommunityUtil.getJSONString(0);
  }
  ```

------

### 统一异常处理

![image-20221111184406175](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221111184406175.png)

在 HomeController 中增加 `getErrorPage` 方法，返回错误页面。

``` java
@RequestMapping(path = "/error", method = RequestMethod.GET)
    public String getErrorPage() {
        return "/error/500";
    }
```



创建 ExceptionAdvice 类

- 加上 `@ControllerAdvice` 注解，表示该类是 Controller 的全局配置类。 

- 创建 `handleException` 方法，加上 `@ExceptionHandler` 注解，该方法在 Controller 出现异常后调用，处理捕获异常。如果是异步请求返回一个 JSON 数据，否则重定向至 HomeController 的 `getErrorPage` 方法。 

  ``` java
  @ControllerAdvice(annotations = Controller.class)
  public class ExceptionAdvice {
  
      private static final Logger logger = LoggerFactory.getLogger(ExceptionAdvice.class);
  
      @ExceptionHandler({Exception.class})
      public void handleException(Exception e, HttpServletRequest request, HttpServletResponse response) throws IOException {
          logger.error("服务器发生异常: " + e.getMessage());
          for (StackTraceElement element : e.getStackTrace()) {
              logger.error(element.toString());
          }
  
          String xRequestedWith = request.getHeader("x-requested-with");
          if ("XMLHttpRequest".equals(xRequestedWith)) {
              response.setContentType("application/plain;charset=utf-8");
              PrintWriter writer = response.getWriter();
              writer.write(CommunityUtil.getJSONString(1, "服务器异常!"));
          } else {
              response.sendRedirect(request.getContextPath() + "/error");
          }
      }
  
  }
  ```

  

------

### 统一日志处理

![image-20221112164827045](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221112164827045.png)

![image-20221112165024302](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221112165024302.png)

![image-20221112165047921](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221112165047921.png)

![image-20221112165421373](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221112165421373.png)

![image-20221112165644604](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221112165644604.png)

service层，记录业务日志



在 `pom.xml` 引入 aspectj 的依赖。

创建 ServiceLogAspect 类，添加 `@Aspect` 切面注解，配置切入点表达式，拦截所有 service 包下的方法，利用 `@Before` 记录日志。

```java
public class ServiceLogAspect {

    private static final Logger logger = LoggerFactory.getLogger(ServiceLogAspect.class);

    @Pointcut("execution(* com.nowcoder.community.service.*.*(..))")
    public void pointcut() {

    }

    @Before("pointcut()")
    public void before(JoinPoint joinPoint) {
        // 用户[1.2.3.4],在[xxx],访问了[com.nowcoder.community.service.xxx()].
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (attributes == null) {
            return;
        }
        HttpServletRequest request = attributes.getRequest();
        String ip = request.getRemoteHost();
        String now = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date());
        String target = joinPoint.getSignature().getDeclaringTypeName() + "." + joinPoint.getSignature().getName();
        logger.info(String.format("用户[%s],在[%s],访问了[%s].", ip, now, target));
    }

}
```

## 6.Redis

### 点赞

![image-20221112172850769](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221112172850769.png)

[通用点赞设计思路 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903722066116621) 

[如何设计微博点赞功能数据库？_51CTO博客_点赞 数据库设计](https://blog.51cto.com/lxw1844912514/5687288) 

[新浪微博「点赞功能」数据库如何设计的？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/63947513)可以参考，业务要求少的话用set就行（只存一个userid），业务要求多的话用hash或者zset（要存类似于时间呀等等）。

点赞作为一个高频率的操作，如果每次操作都读写数据库会增加数据库的压力，所以采用缓存+定时任务来实现。点赞数据是在redis中缓存半小时，同时定时任务是每隔5分钟执行一次，做持久化存储，这里的缓存时间和任务执行时间可根据项目情况而定。还可以用到前端动画，显示加一，然后在改动redis，减少一次查询，像b站一样，你下拉刷新那时候在真正的查询。

创建 RedisKeyUtil 工具类

- 定义分隔符 `:` 以及实体获得赞的 key 前缀常量 `like:entity`（set实现）。

- 新增 `getEntityLikeKey(int entityType,int entityId)` 方法，通过实体类型和实体 id 生成对应实体获得赞的 key。

  ![image-20221112181151985](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221112181151985.png)

创建业务层的 LikeService 类

- 注入 RedisTemplate 实例。

- 新增 `like` 点赞方法，首先通过 RedisKeyUtil 工具类的 `getEntityLikeKey` 方法获得实体点赞的 key，然后通过 RedisTemplate 对象对 set 集合的 `isMember` 方法查询 userId 是否存在于对应 key 的 set 集合中，如果存在则移除出点赞的用户集合，如果不存在则添加到点赞的用户集合。

  ```java
  public void like(int userId, int entityType, int entityId, int entityUserId) {
      redisTemplate.execute(new SessionCallback() {
          @Override
          public Object execute(RedisOperations operations) throws DataAccessException {
              String entityLikeKey = RedisKeyUtil.getEntityLikeKey(entityType, entityId);
              String userLikeKey = RedisKeyUtil.getUserLikeKey(entityUserId);
  
              boolean isMember = operations.opsForSet().isMember(entityLikeKey, userId);
  
              operations.multi();
  
              if (isMember) {
                  operations.opsForSet().remove(entityLikeKey, userId);
                  operations.opsForValue().decrement(userLikeKey);
              } else {
                  operations.opsForSet().add(entityLikeKey, userId);
                  operations.opsForValue().increment(userLikeKey);
              }
  
              return operations.exec();
          }
      });
  }
  ```

- 新增 `findEntityLikeCount` 方法查询实体的点赞数量，通过调用 set 集合的 `size` 方法查询元素个数。

  ```java
  // 查询某实体点赞的数量
  public long findEntityLikeCount(int entityType, int entityId) {
      String entityLikeKey = RedisKeyUtil.getEntityLikeKey(entityType, entityId);
      return redisTemplate.opsForSet().size(entityLikeKey);
  }
  ```

- 新增 `findEntityLikeStatus` 方法查询某用户对某实体的点赞状态，逻辑如 `like` 方法，通过 set 集合的 `isMember` 方法实现。

- ```java
  // 查询某人对某实体的点赞状态 0代表没点，1代表点了
  public int findEntityLikeStatus(int userId, int entityType, int entityId) {
      String entityLikeKey = RedisKeyUtil.getEntityLikeKey(entityType, entityId);
      return redisTemplate.opsForSet().isMember(entityLikeKey, userId) ? 1 : 0;
  }
  ```

创建表现层的 LikeController 类

- 注入 LikeService 和 HostHolder 实例。

- 新增 `like` 点赞方法，调用业务层的 `like` 方法进行点赞、调用 `findEntityLikeCount` 和 `findEntityLikeStatus` 查询点赞数量和点赞状态，封装到 map 集合，然后通过工具类封装成 JSON 数据返回。（后续加入Kafka操作队列异步触发点赞方法的真正执行和计算帖子分数）

  ```java
  @RequestMapping(path = "/like", method = RequestMethod.POST)
  @ResponseBody
  public String like(int entityType, int entityId, int entityUserId, int postId) {
      User user = hostHolder.getUser();
  
      // 点赞
      likeService.like(user.getId(), entityType, entityId, entityUserId);
  
      // 数量
      long likeCount = likeService.findEntityLikeCount(entityType, entityId);
      // 状态
      int likeStatus = likeService.findEntityLikeStatus(user.getId(), entityType, entityId);
      // 返回的结果
      Map<String, Object> map = new HashMap<>();
      map.put("likeCount", likeCount);
      map.put("likeStatus", likeStatus);
  
      // 触发点赞事件
      if (likeStatus == 1) {
          Event event = new Event()
                  .setTopic(TOPIC_LIKE)
                  .setUserId(hostHolder.getUser().getId())
                  .setEntityType(entityType)
                  .setEntityId(entityId)
                  .setEntityUserId(entityUserId)
                  .setData("postId", postId);
          eventProducer.fireEvent(event);
      }
  
      if(entityType == ENTITY_TYPE_POST) {
          // 计算帖子分数
          String redisKey = RedisKeyUtil.getPostScoreKey();
          redisTemplate.opsForSet().add(redisKey, postId);
      }
  
      return CommunityUtil.getJSONString(0, null, map);
  }
  ```

（更新首页帖子点赞数量）在表现层的 HomeController 类

- 注入 LikeService 实例。
- 在 `getIndexPage` 方法在通过 LikeService 类的方法获得点赞数量，存储到 map 集合。

------

### 收到的赞

对点赞功能进行重构

![image-20221112181840614](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221112181840614.png)

在 RedisUnitl 工具类

- 新增用户获得赞 key 的前缀常量 `like:user`
- 新增 `getUserLikeKey(int userId)` 方法，通过用户 id 生成对应用户获得赞的 key。

在 LikeService 中

- 重构 `like` 方法，在参数列表中加入 entityUserId 表示被点赞用户的 id，用来更新用户的被点赞数量。（代码在上一节）
  - 通过 RedisTemplate 对象的 `execute` 方法实现事务，保证被点赞用户点和点赞用户的数据更新一致。通过 `isMember` 方法查询用户的点赞状态，之后通过 `mutli` 方法开启事务。
  - 当用户已点赞时，调用 `remove` 方法将当前用户从点赞用户的集合中移除，调用 `decrement` 方法将被点赞用户的被点赞数减 1；当用户未点赞时，调用 `add` 方法将当前用户添加到点赞用户的集合，调用 `increment` 方法将被点赞用户的被点赞数加 1。
- 增加 `findUserLikeCount` 方法，以用户 id 作为 key，调用 `get` 方法查询用户所获得的点赞数。

在 LikeController 中给 `like` 方法增加 entityUserId 参数即可，相当于是在给用户点赞的时候，被点赞用户的总被赞数增加。

------

### 关注

![image-20221112205844736](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221112205844736.png)

数据结构采用zset，score表示关注事件

在 RedisUnitl 工具类

- 新增用户关注实体（帖子、评论、用户等）和粉丝（用户）的前缀常量 `followee` 和 `follower`
- 新增 `getFolloweeKey(int userId, int entityType)` 方法，通过用户 id 和实体类型生成用户关注实体的 key。
- 新增 `getFollowerKey(int entityType, int entityId)` 方法，通过实体类型和实体 id 生成实体用户粉丝的 key。

创建业务层的 FollowService 类

- 新增follow方法，当用户关注某实体时，

  - 调用 `add` 方法将当前实体 id 和时间作为 value 和 score加入用户的关注集合。

  - 调用 `add` 方法将当前用户 id 和时间作为 value 和 score 加入实体的粉丝集合。

    ```java
    public void follow(int userId, int entityType, int entityId) {
        redisTemplate.execute(new SessionCallback() {
            @Override
            public Object execute(RedisOperations operations) throws DataAccessException {
                String followeeKey = RedisKeyUtil.getFolloweeKey(userId, entityType);
                String followerKey = RedisKeyUtil.getFollowerKey(entityType, entityId);
    
                operations.multi();
    
                operations.opsForZSet().add(followeeKey, entityId, System.currentTimeMillis());
                operations.opsForZSet().add(followerKey, userId, System.currentTimeMillis());
    
                return operations.exec();
            }
        });
    }
    ```

- 新增unfollow方法，当用户取消关注某实体时，

  - 调用 `remove` 方法将当前实体从用户的关注集合移除。

  - 调用 `remove` 方法将用户从实体的粉丝集合移除。

    ```java
    public void unfollow(int userId, int entityType, int entityId) {
        redisTemplate.execute(new SessionCallback() {
            @Override
            public Object execute(RedisOperations operations) throws DataAccessException {
                String followeeKey = RedisKeyUtil.getFolloweeKey(userId, entityType);
                String followerKey = RedisKeyUtil.getFollowerKey(entityType, entityId);
    
                operations.multi();
    
                operations.opsForZSet().remove(followeeKey, entityId);
                operations.opsForZSet().remove(followerKey, userId);
    
                return operations.exec();
            }
        });
    }
    ```

------

### 个人主页

在业务层的 FollowService 类

- 新增 `findFolloweeCount` 方法，调用 zset 的 `zcard` 方法查询某用户关注的实体数量。

- 新增 `findFollowerCount` 方法，调用 zset 的 `zcard` 方法查询某实体的粉丝数量。

- 新增 `hasFollowed` 方法，根据 zset 的 `zscore` 方法返回值查询当前用户是否关注某实体。

  ```java
  // 查询关注的实体的数量
  public long findFolloweeCount(int userId, int entityType) {
      String followeeKey = RedisKeyUtil.getFolloweeKey(userId, entityType);
      return redisTemplate.opsForZSet().zCard(followeeKey);
  }
  
  // 查询实体的粉丝的数量
  public long findFollowerCount(int entityType, int entityId) {
      String followerKey = RedisKeyUtil.getFollowerKey(entityType, entityId);
      return redisTemplate.opsForZSet().zCard(followerKey);
  }
  
  // 查询当前用户是否已关注该实体
  public boolean hasFollowed(int userId, int entityType, int entityId) {
      String followeeKey = RedisKeyUtil.getFolloweeKey(userId, entityType);
      return redisTemplate.opsForZSet().score(followeeKey, entityId) != null;
  }
  ```

在 UserController 中新增 `getProfilePage` 方法获取个人主页。

- 调用 LikeService 的 `findUserLikeCount` 查询用户获赞数，并添加到 Model 中。
- 调用 FollowService 的`findFolloweeCount`、`findFollowerCount` 、`hasFollowed` 方法分别查询关注数量、粉丝数量、用户是否关注三项信息并添加到 Model 对象中存储。

------

### 关注列表和粉丝列表

在业务层的 FollowService 类

- 新增 `findFollowees` 方法，查询用户关注列表，主要通过 zset 的 `reverseRange` 获取 value 即关注用户的 userId，再查询出其 user，之后通过 `score` 获取关注时间，存入 map 集合，将 map 添加到 list 列表返回。
- 新增 `findFollowers` 方法，查询用户粉丝列表，主要通过 zset 的 `reverseRange` 获取 value 即粉丝的 userId，再查询出其 user，之后通过 `score` 获取关注时间，存入 map 集合，将 map 添加到 list 列表返回。

在表现层的 FollowController 类

- 新增 `getFollowees` 方法，获取关注列表，存入 Model 对象。
- 新增 `getFollowers` 方法，获取粉丝列表，存入 Model 对象。

------

### 优化登录模块

![image-20221112211805462](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221112211805462.png)

**存储验证码**

在 RedisUntil 工具类

- 新增验证码前缀常量 `kaptcha`
- 新增 `getKaptchaKey` 方法，通过一个用户凭证（**由于未登录，利用 cookie 实现**）获得对应验证码的 key 值（利用 string 存储验证码）。

在表现层的 LoginController 类

- 重构 `getKaptcha` 方法，将验证码存入 redis，**key 值是当前随机生成的一个字符串**，同时将该字符串存入 cookie。**用到了cookie**，不要用userid当前缀，可能被钻小空子。

- ![image-20221112213028034](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221112213028034.png)

  

- 重构 `login` 方法，从 cookie 中获得随机字符串，生成验证码的 key 值，然后获取对应的 value 值即验证码。

- ![image-20221112212920538](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221112212920538.png)

------

**存储登录凭证**

在 RedisUntil 工具类

- 新增登录凭证前缀常量 `ticket`

- 新增 `getTicketKey` 方法，通过字符串获得登录凭证的对应 key 值（利用 string 存储）。

  ```java
  // 登录的凭证
  public static String getTicketKey(String ticket) {
      return PREFIX_TICKET + SPLIT + ticket;
  }
  ```

在业务层的 UserService 类

- 重构 `login` 方法，将登录凭证存入 redis 中。

  ![image-20221112214247439](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221112214247439.png)

- 重构 `logout` 方法，先从 redis 中获取登录凭证对象，将状态设为无效再重新存储进 redis。

  ```java
   public void logout(String ticket) {
  //        loginTicketMapper.updateStatus(ticket, 1);
          String redisKey = RedisKeyUtil.getTicketKey(ticket);
          LoginTicket loginTicket = (LoginTicket) redisTemplate.opsForValue().get(redisKey);
          loginTicket.setStatus(1);
          redisTemplate.opsForValue().set(redisKey, loginTicket);
      }
  ```

- 重构 `findLoginTicket` 方法，根据 ticket 字符串获得对应登录凭证的 key，然后从 redis 查询登录凭证。

  ```java
  public LoginTicket findLoginTicket(String ticket) {
  //        return loginTicketMapper.selectByTicket(ticket);
          String redisKey = RedisKeyUtil.getTicketKey(ticket);
          return (LoginTicket) redisTemplate.opsForValue().get(redisKey);
      }
  ```

------

**缓存用户信息**

在 RedisUntil 工具类

- 新增用户前缀常量 `user`

- 新增 `getUserKey` 方法，通过用户 id 获得用户的对应 key 值（利用 string 存储）。

  ```java
  // 用户
  public static String getUserKey(int userId) {
      return PREFIX_USER + SPLIT + userId;
  }
  ```

在业务层的 UserService 类

- 新增 `getCache`，从缓存获取用户信息。

- 新增 `initCache`，从 MySQL 查询用户信息并存入 redis。

- 新增 `clearCache`，用户信息变更（更新头像，激活）时清除缓存。

- ```java
  // 1.优先从缓存中取值
  private User getCache(int userId) {
      String redisKey = RedisKeyUtil.getUserKey(userId);
      return (User) redisTemplate.opsForValue().get(redisKey);
  }
  
  // 2.取不到时初始化缓存数据
  private User initCache(int userId) {
      User user = userMapper.selectById(userId);
      String redisKey = RedisKeyUtil.getUserKey(userId);
      redisTemplate.opsForValue().set(redisKey, user, 3600, TimeUnit.SECONDS);
      return user;
  }
  
  // 3.数据变更时清除缓存数据
  private void clearCache(int userId) {
      String redisKey = RedisKeyUtil.getUserKey(userId);
      redisTemplate.delete(redisKey);
  }
  ```

- 重构 `findUserById` 方法，首先调用 `getCache`从缓存获取用户信息，如果获取为 null 则调用 `initCache`。先更新数据库在清除缓存。

  ```java
  public User findUserById(int id) {
  //        return userMapper.selectById(id);
          User user = getCache(id);
          if (user == null) {
              user = initCache(id);
          }
          return user;
      }
  ```

## 7.Kafka

### 发送系统通知

![image-20221113112556500](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113112556500.png)

在 CommunityConstant 接口中新增三个常量，代表三个主题：评论、点赞、关注。

创建 Event 类，封装事件对象，包括主题、用户 id、实体类型、实体 id、实体用户 id 以及一个 map 集合存放其它信息。链式调用。

![image-20221113120007570](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113120007570.png)

![image-20221113120040191](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113120040191.png)

#### 触发事件

创建 EventProducer 事件生产者，新增 `fireEvent(Event event)` 方法，通过 Event 获取事件类型，并将其封装成 JSON 数据，然后调用注入的 KafkaTemplate 实例的 send 方法发送。

```java
@Component
public class EventProducer {

    @Autowired
    private KafkaTemplate kafkaTemplate;

    // 处理事件
    public void fireEvent(Event event) {
        // 将事件发布到指定的主题
        kafkaTemplate.send(event.getTopic(), JSONObject.toJSONString(event));
    }

}
```

在 CommentController、LikeControler、FollowController 中注入 EventProducer 实例，分别重构 `addComment` 方法、`like` 方法、`follow` 方法，封装 Event 对象，然后调用 EventProducer 的`fireEvent` 方法发布通知。

#### 消费事件

创建 EventConsumer 事件消费者，消费者是被动触发的，这一步全是系统通知。

- 注入 MessageService 实例。

- 增加 `handleCommentMessage(ConsumerRecord record)` 方法，通过 `@KafkaListener` 注解，topic 包括了评论、点赞和关注。从 recored 中获取信息，封装成 Message 对象然后调用 `addMessage` 方法插入数据库。

  ```java
  @KafkaListener(topics = {TOPIC_COMMENT, TOPIC_LIKE, TOPIC_FOLLOW})
  public void handleCommentMessage(ConsumerRecord record) {
      if (record == null || record.value() == null) {
          logger.error("消息的内容为空!");
          return;
      }
  
      Event event = JSONObject.parseObject(record.value().toString(), Event.class);
      if (event == null) {
          logger.error("消息格式错误!");
          return;
      }
  
      // 发送站内通知
      Message message = new Message();
      message.setFromId(SYSTEM_USER_ID);
      message.setToId(event.getEntityUserId());
      message.setConversationId(event.getTopic());
      message.setCreateTime(new Date());
  
      Map<String, Object> content = new HashMap<>();
      content.put("userId", event.getUserId());
      content.put("entityType", event.getEntityType());
      content.put("entityId", event.getEntityId());
  
      if (!event.getData().isEmpty()) {
          for (Map.Entry<String, Object> entry : event.getData().entrySet()) {
              content.put(entry.getKey(), entry.getValue());
          }
      }
  
      message.setContent(JSONObject.toJSONString(content));
      messageService.addMessage(message);
  }
  ```

【问题】没有向数据库插入系统通知记录，原因是 ServiceLogAspect 类进行日志处理时要获取 ServletRequestAttributes 请求对象，这个对象是controller里面的，Kafka 的消费事件是自动触发的，没有进行新的请求，会产生请求对象的空指针异常。

![image-20221113140611563](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113140611563.png)

------

### 显示系统通知

![image-20221113140748886](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113140748886.png)

#### 通知列表

在 MessageMapper 接口中

- 新增 `selectLatestNotice(int userId, String topic)` 方法，查询某主题最新的通知。

  ```java
  <select id="selectLatestNotice" resultType="com.nowcoder.community.entity.Message">
      select <include refid="selectFields"></include>
      from message
      where id in (
          select max(id) from message
          where status != 2
          and from_id = 1
          and to_id = #{userId}
          and conversation_id = #{topic}
      )
  </select>
  ```

- 新增 `selectNoticeCount(int userId, String topic)` 方法，查询某主题通知的数量。

  ```java
  <select id="selectNoticeCount" resultType="int">
      select count(id) from message
      where status != 2
      and from_id = 1
      and to_id = #{userId}
      and conversation_id = #{topic}
  </select>
  ```

- 新增 `selectNoticeUnreadCount(int userId, String topic)` 方法，查询未读通知的数量，使用 if 动态语句，如果没有传入 topic 就查询未读总量。

  ```xml
  <select id="selectNoticeUnreadCount" resultType="int">
      select count(id) from message
      where status = 0
      and from_id = 1
      and to_id = #{userId}
      <if test="topic!=null">
          and conversation_id = #{topic}
      </if>
  </select>
  ```

在业务层的 MessageService 中

- 新增 `findLatestNotice` 方法，调用 `selectLatestNotice` 方法查询最新通知。	

  ```java
  public Message findLatestNotice(int userId, String topic) {
      return messageMapper.selectLatestNotice(userId, topic);
  }
  ```

- 新增 `findNoticeCount` 方法，调用 `selectNoticeCount` 方法查询某主题通知的数量。

  ```java
  public int findNoticeCount(int userId, String topic) {
      return messageMapper.selectNoticeCount(userId, topic);
  }
  ```

- 新增 `findNoticeUnreadCount` 方法，调用 `selectNoticeUnreadCount` 方法查询未读通知的数量。

  ```java
  public int findNoticeUnreadCount(int userId, String topic) {
      return messageMapper.selectNoticeUnreadCount(userId, topic);
  }
  ```

在表现层的 MessageController 中新增 `getNoticeList` 方法，获取通知列表

- 调用业务层 MessageService 的方法查询评论、点赞、关注的通知，将其封装在一个 HashMap 集合中然后添加到 Model 对象里。

- 调用业务层 MessageService 的方法查询私信和通知的总未读数量，添加到 Model 对象里。

- 返回 `notice.html` 页面。

- ```java
  @RequestMapping(path = "/notice/list", method = RequestMethod.GET)
  public String getNoticeList(Model model) {
      User user = hostHolder.getUser();
  
      // 查询评论类通知
      Message message = messageService.findLatestNotice(user.getId(), TOPIC_COMMENT);
      if (message != null) {
          Map<String, Object> messageVO = new HashMap<>();
          messageVO.put("message", message);
  
          String content = HtmlUtils.htmlUnescape(message.getContent());
          Map<String, Object> data = JSONObject.parseObject(content, HashMap.class);
  
          messageVO.put("user", userService.findUserById((Integer) data.get("userId")));
          messageVO.put("entityType", data.get("entityType"));
          messageVO.put("entityId", data.get("entityId"));
          messageVO.put("postId", data.get("postId"));
  
          int count = messageService.findNoticeCount(user.getId(), TOPIC_COMMENT);
          messageVO.put("count", count);
  
          int unread = messageService.findNoticeUnreadCount(user.getId(), TOPIC_COMMENT);
          messageVO.put("unread", unread);
  
          model.addAttribute("commentNotice", messageVO);
      }
  
      // 查询点赞类通知
      message = messageService.findLatestNotice(user.getId(), TOPIC_LIKE);
      if (message != null) {
          Map<String, Object> messageVO = new HashMap<>();
          messageVO.put("message", message);
  
          String content = HtmlUtils.htmlUnescape(message.getContent());
          Map<String, Object> data = JSONObject.parseObject(content, HashMap.class);
  
          messageVO.put("user", userService.findUserById((Integer) data.get("userId")));
          messageVO.put("entityType", data.get("entityType"));
          messageVO.put("entityId", data.get("entityId"));
          messageVO.put("postId", data.get("postId"));
  
          int count = messageService.findNoticeCount(user.getId(), TOPIC_LIKE);
          messageVO.put("count", count);
  
          int unread = messageService.findNoticeUnreadCount(user.getId(), TOPIC_LIKE);
          messageVO.put("unread", unread);
  
          model.addAttribute("likeNotice", messageVO);
      }
  
      // 查询关注类通知
      message = messageService.findLatestNotice(user.getId(), TOPIC_FOLLOW);
      if (message != null) {
          Map<String, Object> messageVO = new HashMap<>();
          messageVO.put("message", message);
  
          String content = HtmlUtils.htmlUnescape(message.getContent());
          Map<String, Object> data = JSONObject.parseObject(content, HashMap.class);
  
          messageVO.put("user", userService.findUserById((Integer) data.get("userId")));
          messageVO.put("entityType", data.get("entityType"));
          messageVO.put("entityId", data.get("entityId"));
  
          int count = messageService.findNoticeCount(user.getId(), TOPIC_FOLLOW);
          messageVO.put("count", count);
  
          int unread = messageService.findNoticeUnreadCount(user.getId(), TOPIC_FOLLOW);
          messageVO.put("unread", unread);
  
          model.addAttribute("followNotice", messageVO);
      }
  
      // 查询未读消息数量
      int letterUnreadCount = messageService.findLetterUnreadCount(user.getId(), null);
      model.addAttribute("letterUnreadCount", letterUnreadCount);
      int noticeUnreadCount = messageService.findNoticeUnreadCount(user.getId(), null);
      model.addAttribute("noticeUnreadCount", noticeUnreadCount);
  
      return "/site/notice";
  }
  ```

------

#### 显示通知详情

在 MessageMapper 接口新增 `selectNotices` 方法，查询某个主题的通知列表，在 `message-mapper.xml` 配置 SQL。

```xml
<select id="selectNotices" resultType="com.nowcoder.community.entity.Message">
    select <include refid="selectFields"></include>
    from message
    where status != 2
    and from_id = 1
    and to_id = #{userId}
    and conversation_id = #{topic}
    order by create_time desc
    limit #{offset}, #{limit}
</select>
```

在业务层的 MessageService 中新增 `findNotices` 方法，调用 `selectNotices` 方法。

```java
public List<Message> findNotices(int userId, String topic, int offset, int limit) {
    return messageMapper.selectNotices(userId, topic, offset, limit);
}
```

在表现层的 MessageController 中新增 `getNoticeDetail` 方法

- 调用 `findNotices` 方法获取通知列表详情，封装到 List 集合并存入 Model 对象。

- 从通知集合中获取 id 集合，调用 `readMessage` 方法将消息设为已读。

- 返回 `notice-detail.html` 页面。

  ```java
  @RequestMapping(path = "/notice/detail/{topic}", method = RequestMethod.GET)
  public String getNoticeDetail(@PathVariable("topic") String topic, Page page, Model model) {
      User user = hostHolder.getUser();
  
      page.setLimit(5);
      page.setPath("/notice/detail/" + topic);
      page.setRows(messageService.findNoticeCount(user.getId(), topic));
  
      List<Message> noticeList = messageService.findNotices(user.getId(), topic, page.getOffset(), page.getLimit());
      List<Map<String, Object>> noticeVoList = new ArrayList<>();
      if (noticeList != null) {
          for (Message notice : noticeList) {
              Map<String, Object> map = new HashMap<>();
              // 通知
              map.put("notice", notice);
              // 内容
              String content = HtmlUtils.htmlUnescape(notice.getContent());
              Map<String, Object> data = JSONObject.parseObject(content, HashMap.class);
              map.put("user", userService.findUserById((Integer) data.get("userId")));
              map.put("entityType", data.get("entityType"));
              map.put("entityId", data.get("entityId"));
              map.put("postId", data.get("postId"));
              // 通知作者
              map.put("fromUser", userService.findUserById(notice.getFromId()));
  
              noticeVoList.add(map);
          }
      }
      model.addAttribute("notices", noticeVoList);
  
      // 设置已读
      List<Integer> ids = getLetterIds(noticeList);
      if (!ids.isEmpty()) {
          messageService.readMessage(ids);
      }
  
      return "/site/notice-detail";
  }
  ```

------

#### 显示未读通知总数

创建 MessageInterceptor 拦截器

- 注入 MessageService 实例和 HostHolder 实例。

- 重写 `postHandle` 方法，查询私信和通知的未读数量和，然后添加到 ModelAndView 对象。

  ```java
  @Component
  public class MessageInterceptor implements HandlerInterceptor {
  
      @Autowired
      private HostHolder hostHolder;
  
      @Autowired
      private MessageService messageService;
  
      @Override
      public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
          User user = hostHolder.getUser();
          if (user != null && modelAndView != null) {
              int letterUnreadCount = messageService.findLetterUnreadCount(user.getId(), null);
              int noticeUnreadCount = messageService.findNoticeUnreadCount(user.getId(), null);
              modelAndView.addObject("allUnreadCount", letterUnreadCount + noticeUnreadCount);
          }
      }
  }
  ```

在 WebConfig 中注入 MessageInterceptor 实例，并在 `addInterceptors` 方法中添加该拦截器。

## 8.Elasticsearch

### 基本使用

![image-20221113160636492](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113160636492.png)

类型在es7里面已经没用了，六里面也只是个占位的没实际作用了，比如下面put请求里面的_doc就没什么用。

![image-20221113160838751](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113160838751.png)

![image-20221113161008759](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113161008759.png)

### Spring整合es

![image-20221113161603261](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113161603261.png)

* 配置文件

  ```xml
  # ElasticsearchProperties
  spring.data.elasticsearch.cluster-name=nowcoder
  spring.data.elasticsearch.cluster-nodes=127.0.0.1:9300
  ```

* 解决redis和es之间的netty冲突

  ![image-20221113162115230](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113162115230.png)

* 配置帖子的实体类,将之与es中的属性关联。

  ![image-20221113162808798](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113162808798.png)

* 在dao包下创建es子包，在其中创建` discussPostRepository`

  ```java
  @Repository  //泛型中的第二个参数是主键的类型
  public interface DiscussPostRepository extends ElasticsearchRepository<DiscussPost, Integer> {
  
  }
  ```

* 在test中测试增删改查方法。

### 开发社区搜索功能

![image-20221113165032970](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113165032970.png)

* 业务层，创建ElasticSearchService，编写增删改查方法。

  ```java
  @Service
  public class ElasticsearchService {
  
      @Autowired
      private DiscussPostRepository discussRepository;
  
      @Autowired
      private ElasticsearchTemplate elasticTemplate;
  
      public void saveDiscussPost(DiscussPost post) {
          discussRepository.save(post);
      }
  
      public void deleteDiscussPost(int id) {
          discussRepository.deleteById(id);
      }
  
      public Page<DiscussPost> searchDiscussPost(String keyword, int current, int limit) {
          SearchQuery searchQuery = new NativeSearchQueryBuilder()
                  .withQuery(QueryBuilders.multiMatchQuery(keyword, "title", "content"))
                  .withSort(SortBuilders.fieldSort("type").order(SortOrder.DESC))
                  .withSort(SortBuilders.fieldSort("score").order(SortOrder.DESC))
                  .withSort(SortBuilders.fieldSort("createTime").order(SortOrder.DESC))
                  .withPageable(PageRequest.of(current, limit))
                  .withHighlightFields(
                          new HighlightBuilder.Field("title").preTags("<em>").postTags("</em>"),
                          new HighlightBuilder.Field("content").preTags("<em>").postTags("</em>")
                  ).build();
  
          return elasticTemplate.queryForPage(searchQuery, DiscussPost.class, new SearchResultMapper() {
              @Override
              public <T> AggregatedPage<T> mapResults(SearchResponse response, Class<T> aClass, Pageable pageable) {
                  SearchHits hits = response.getHits();
                  if (hits.getTotalHits() <= 0) {
                      return null;
                  }
  
                  List<DiscussPost> list = new ArrayList<>();
                  for (SearchHit hit : hits) {
                      DiscussPost post = new DiscussPost();
  
                      String id = hit.getSourceAsMap().get("id").toString();
                      post.setId(Integer.valueOf(id));
  
                      String userId = hit.getSourceAsMap().get("userId").toString();
                      post.setUserId(Integer.valueOf(userId));
  
                      String title = hit.getSourceAsMap().get("title").toString();
                      post.setTitle(title);
  
                      String content = hit.getSourceAsMap().get("content").toString();
                      post.setContent(content);
  
                      String status = hit.getSourceAsMap().get("status").toString();
                      post.setStatus(Integer.valueOf(status));
  
                      String createTime = hit.getSourceAsMap().get("createTime").toString();
                      post.setCreateTime(new Date(Long.valueOf(createTime)));
  
                      String commentCount = hit.getSourceAsMap().get("commentCount").toString();
                      post.setCommentCount(Integer.valueOf(commentCount));
  
                      // 处理高亮显示的结果
                      HighlightField titleField = hit.getHighlightFields().get("title");
                      if (titleField != null) {
                          post.setTitle(titleField.getFragments()[0].toString());
                      }
  
                      HighlightField contentField = hit.getHighlightFields().get("content");
                      if (contentField != null) {
                          post.setContent(contentField.getFragments()[0].toString());
                      }
  
                      list.add(post);
                  }
  
                  return new AggregatedPageImpl(list, pageable,
                          hits.getTotalHits(), response.getAggregations(), response.getScrollId(), hits.getMaxScore());
              }
          });
      }
  
  }
  ```

* 表现层 在DiscussPostController的`addDiscussPost`方法中添加异步触发发帖事件的代码，利用kafka。

  ```java
  // 触发发帖事件
  Event event = new Event()
          .setTopic(TOPIC_PUBLISH)
          .setUserId(user.getId())
          .setEntityType(ENTITY_TYPE_POST)
          .setEntityId(post.getId());
  eventProducer.fireEvent(event);
  ```

* 在commentController的`addComment`方法中添加代码

  ```
  if (comment.getEntityType() == ENTITY_TYPE_POST) {
              // 触发发帖事件
              event = new Event()
                      .setTopic(TOPIC_PUBLISH)
                      .setUserId(comment.getUserId())
                      .setEntityType(ENTITY_TYPE_POST)
                      .setEntityId(discussPostId);
              eventProducer.fireEvent(event);
              // 计算帖子分数
              String redisKey = RedisKeyUtil.getPostScoreKey();
              redisTemplate.opsForSet().add(redisKey, discussPostId);
          }
  ```

* 在EventConsumer中添加新方法消费将帖子发布到es中的事件

  ```java
  // 消费发帖事件
  @KafkaListener(topics = {TOPIC_PUBLISH})
  public void handlePublishMessage(ConsumerRecord record) {
      if (record == null || record.value() == null) {
          logger.error("消息的内容为空!");
          return;
      }
  
      Event event = JSONObject.parseObject(record.value().toString(), Event.class);
      if (event == null) {
          logger.error("消息格式错误!");
          return;
      }
  
      DiscussPost post = discussPostService.findDiscussPostById(event.getEntityId());
      elasticsearchService.saveDiscussPost(post);
  }
  ```

* 新建SearchController

  ```java
  @Controller
  public class SearchController implements CommunityConstant {
  
      @Autowired
      private ElasticsearchService elasticsearchService;
  
      @Autowired
      private UserService userService;
  
      @Autowired
      private LikeService likeService;
  
      // search?keyword=xxx
      @RequestMapping(path = "/search", method = RequestMethod.GET)
      public String search(String keyword, Page page, Model model) {
          // 搜索帖子
          org.springframework.data.domain.Page<DiscussPost> searchResult =
                  elasticsearchService.searchDiscussPost(keyword, page.getCurrent() - 1, page.getLimit());
          // 聚合数据
          List<Map<String, Object>> discussPosts = new ArrayList<>();
          if (searchResult != null) {
              for (DiscussPost post : searchResult) {
                  Map<String, Object> map = new HashMap<>();
                  // 帖子
                  map.put("post", post);
                  // 作者
                  map.put("user", userService.findUserById(post.getUserId()));
                  // 点赞数量
                  map.put("likeCount", likeService.findEntityLikeCount(ENTITY_TYPE_POST, post.getId()));
  
                  discussPosts.add(map);
              }
          }
          model.addAttribute("discussPosts", discussPosts);
          model.addAttribute("keyword", keyword);
  
          // 分页信息
          page.setPath("/search?keyword=" + keyword);
          page.setRows(searchResult == null ? 0 : (int) searchResult.getTotalElements());
  
          return "/site/search";
      }
  
  }
  ```

* 结果展示
* ![image-20221113171722613](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113171722613.png)

## 9.Spring Security

### 基本使用

<img src="https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113211027597.png" alt="image-20221113211027597" style="zoom:67%;" />

<img src="https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113211550019.png" alt="image-20221113211550019" style="zoom:50%;" />

**Spring Secruity - demo**

* 实体类要实现userdetails接口，实现相关的方法。

  ```java
  // true: 账号未过期.
  @Override
  public boolean isAccountNonExpired() {
      return true;
  }
  
  // true: 账号未锁定.
  @Override
  public boolean isAccountNonLocked() {
      return true;
  }
  
  // true: 凭证未过期.
  @Override
  public boolean isCredentialsNonExpired() {
      return true;
  }
  
  // true: 账号可用.
  @Override
  public boolean isEnabled() {
      return true;
  }
  
  @Override
  public Collection<? extends GrantedAuthority> getAuthorities() {
      List<GrantedAuthority> list = new ArrayList<>();
      list.add(new GrantedAuthority() {
          @Override
          public String getAuthority() {
              switch (type) {
                  case 1:
                      return "ADMIN";
                  default:
                      return "USER";
              }
          }
      });
      return list;
  }
  ```

* UserService也要实现UserDetailService接口

  ```java
  @Service
  public class UserService implements UserDetailsService {
  
      @Autowired
      private UserMapper userMapper;
  
      public User findUserByName(String username) {
          return userMapper.selectByName(username);
      }
  
      @Override
      public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
          return this.findUserByName(username);
      }
  }
  ```

* 配置SpringSecurity，创建SecurityConfig继承WebSecurityConfigureAdapter

  ```
  AuthenticationManager: 认证的核心接口.
  AuthenticationManagerBuilder: 用于构建AuthenticationManager对象的工具.
  ProviderManager: AuthenticationManager接口的默认实现类.
  // 内置的认证规则（不满足需求，自己重写一个providermanager） auth.userDetailsService(userService).passwordEncoder(new Pbkdf2PasswordEncoder("12345"));
  // 自定义认证规则
  AuthenticationProvider: ProviderManager持有一组AuthenticationProvider,每个AuthenticationProvider负责一种认证，有很多种认证管理器的比如账号密码认证器、微信认证器等等，都可以自己写。
  // 委托模式: ProviderManager将认证委托给AuthenticationProvider.
  auth.authenticationProvider(new AuthenticationProvider() {
              // Authentication: 用于封装认证信息的接口,不同的实现类代表不同类型的认证信息.认证成功后会返回一个完整的Authentication，里面包含principal: 主要信息（user）; credentials: 证书（一般是密码）; authorities: 权限。很简单的。
              @Override
              public Authentication authenticate(Authentication authentication) throws AuthenticationException {
                  String username = authentication.getName();
                  String password = (String) authentication.getCredentials();
  
                  User user = userService.findUserByName(username);
                  if (user == null) {
                      throw new UsernameNotFoundException("账号不存在!");
                  }
  
                  password = CommunityUtil.md5(password + user.getSalt());
                  if (!user.getPassword().equals(password)) {
                      throw new BadCredentialsException("密码不正确!");
                  }
  
                  // principal: 主要信息; credentials: 证书; authorities: 权限;
                  return new UsernamePasswordAuthenticationToken(user, user.getPassword(), user.getAuthorities());
              }
              // 当前的AuthenticationProvider支持哪种类型的认证.
              @Override
              public boolean supports(Class<?> aClass) {
                  // UsernamePasswordAuthenticationToken: Authentication接口的常用的实现类.
                  return UsernamePasswordAuthenticationToken.class.equals(aClass);
              }
          });
      }
  ```

  ```java
  配置web相关的，例如登录、退出配置，配置页面访问权限、还可以在provider前在添加一个filter（验证码过滤器），配置记住我（Remember me）。
      @Override
      protected void configure(HttpSecurity http) throws Exception {
          // 登录相关配置
          http.formLogin()
                  .loginPage("/loginpage")
                  .loginProcessingUrl("/login")
                  .successHandler(new AuthenticationSuccessHandler() {
                      @Override
                      public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
                          response.sendRedirect(request.getContextPath() + "/index");
                      }
                  })
                  .failureHandler(new AuthenticationFailureHandler() {
                      @Override
                      public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException e) throws IOException, ServletException {
                          request.setAttribute("error", e.getMessage());
                          request.getRequestDispatcher("/loginpage").forward(request, response);
                      }
                  });
  
          // 退出相关配置
          http.logout()
                  .logoutUrl("/logout")
                  .logoutSuccessHandler(new LogoutSuccessHandler() {
                      @Override
                      public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
                          response.sendRedirect(request.getContextPath() + "/index");
                      }
                  });
  
          // 授权配置
          http.authorizeRequests()
                  .antMatchers("/letter").hasAnyAuthority("USER", "ADMIN")
                  .antMatchers("/admin").hasAnyAuthority("ADMIN")
                  .and().exceptionHandling().accessDeniedPage("/denied");
  
          // 增加Filter,处理验证码
          http.addFilterBefore(new Filter() {
              @Override
              public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
                  HttpServletRequest request = (HttpServletRequest) servletRequest;
                  HttpServletResponse response = (HttpServletResponse) servletResponse;
                  if (request.getServletPath().equals("/login")) {
                      String verifyCode = request.getParameter("verifyCode");
                      if (verifyCode == null || !verifyCode.equalsIgnoreCase("1234")) {
                          request.setAttribute("error", "验证码错误!");
                          request.getRequestDispatcher("/loginpage").forward(request, response);
                          return;
                      }
                  }
                  // 让请求继续向下执行.
                  filterChain.doFilter(request, response);
              }
          }, UsernamePasswordAuthenticationFilter.class);
  
          // 记住我
          http.rememberMe()
                  .tokenRepository(new InMemoryTokenRepositoryImpl())
                  .tokenValiditySeconds(3600 * 24)
                  .userDetailsService(userService);
  
      }
  }
  ```

* Spring Security还有全局的SpringContext对象，我们可以get到认证成功的Authentication对象，里面存有principal（user）、权限、密码（credentials）。

  ![image-20221113221137143](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113221137143.png)



### 项目使用

 	![image-20221114111426436](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114111426436.png)

* 在CommunityConstant常量接口中添加三个新的权限常量

  ```java
  /**
   * 权限: 普通用户
   */
  String AUTHORITY_USER = "user";
  
  /**
   * 权限: 管理员
   */
  String AUTHORITY_ADMIN = "admin";
  
  /**
   * 权限: 版主
   */
  String AUTHORITY_MODERATOR = "moderator";
  ```

* 创建SecurityCofig类，配置SpringSecurity，忽略静态资源，配置各种路径的权限（这里取巧了，很简单的用了一下security，并没有配置provider，实际操作可以看上一节基本使用中的操作）

  ```java
  @Configuration
  public class SecurityConfig extends WebSecurityConfigurerAdapter implements CommunityConstant {
  
      @Override
      public void configure(WebSecurity web) throws Exception {
          web.ignoring().antMatchers("/resources/**");
      }
  
      @Override
      protected void configure(HttpSecurity http) throws Exception {
          // 授权
          http.authorizeRequests()
                  .antMatchers(
                          "/user/setting",
                          "/user/upload",
                          "/discuss/add",
                          "/comment/add/**",
                          "/letter/**",
                          "/notice/**",
                          "/like",
                          "/follow",
                          "/unfollow"
                  )
                  .hasAnyAuthority(
                          AUTHORITY_USER,
                          AUTHORITY_ADMIN,
                          AUTHORITY_MODERATOR
                  )
                  .antMatchers(
                          "/discuss/top",
                          "/discuss/wonderful"
                  )
                  .hasAnyAuthority(
                          AUTHORITY_MODERATOR
                  )
                  .antMatchers(
                          "/discuss/delete",
                          "/data/**",
                          "/actuator/**"
                  )
                  .hasAnyAuthority(
                          AUTHORITY_ADMIN
                  )
                  .anyRequest().permitAll()
                  .and().csrf().disable();
  
          // 权限不够时的处理
          http.exceptionHandling()
                  .authenticationEntryPoint(new AuthenticationEntryPoint() {
                      // 没有登录
                      @Override
                      public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException e) throws IOException, ServletException {
                          String xRequestedWith = request.getHeader("x-requested-with");
                          if ("XMLHttpRequest".equals(xRequestedWith)) {
                              response.setContentType("application/plain;charset=utf-8");
                              PrintWriter writer = response.getWriter();
                              writer.write(CommunityUtil.getJSONString(403, "你还没有登录哦!"));
                          } else {
                              response.sendRedirect(request.getContextPath() + "/login");
                          }
                      }
                  })
                  .accessDeniedHandler(new AccessDeniedHandler() {
                      // 权限不足
                      @Override
                      public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException e) throws IOException, ServletException {
                          String xRequestedWith = request.getHeader("x-requested-with");
                          if ("XMLHttpRequest".equals(xRequestedWith)) {
                              response.setContentType("application/plain;charset=utf-8");
                              PrintWriter writer = response.getWriter();
                              writer.write(CommunityUtil.getJSONString(403, "你没有访问此功能的权限!"));
                          } else {
                              response.sendRedirect(request.getContextPath() + "/denied");
                          }
                      }
                  });
  
          // Security底层默认会拦截/logout请求,进行退出处理.
          // 覆盖它默认的逻辑,才能执行我们自己的退出代码.
          http.logout().logoutUrl("/securitylogout");
      }
  
  }
  ```

* 在UserService中创建获得权限的方法，这里也是取巧，正规的是用User实体继承UserDetail，在其中实现获得权限的方法

  ```java
  public Collection<? extends GrantedAuthority> getAuthorities(int userId) {
      User user = this.findUserById(userId);
  
      List<GrantedAuthority> list = new ArrayList<>();
      list.add(new GrantedAuthority() {
  
          @Override
          public String getAuthority() {
              switch (user.getType()) {
                  case 1:
                      return AUTHORITY_ADMIN;
                  case 2:
                      return AUTHORITY_MODERATOR;
                  default:
                      return AUTHORITY_USER;
              }
          }
      });
      return list;
  }
  ```

* 改造LoginTicketInterceptor，加入存放Authentication到SecurityContext中的步骤（取巧了，这样绕过了SpringSecurity的认证流程，还是采用系统原来的认证方案）

  ![image-20221114113519797](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114113519797.png)

* 关闭csrf攻击检测

### 帖子置顶、加精、删除

![image-20221114134759203](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114134759203.png)

* 导入thymeleaf-springsecurity包

* dao层DiscussPostMapper新增方法updateType、updateStatus修改类型、状态

  ```xml
  <update id="updateType">
      update discuss_post set type = #{type} where id = #{id}
  </update>
  
  <update id="updateStatus">
      update discuss_post set status = #{status} where id = #{id}
  </update>
  ```

* service实现相关的两个方法，调一下dao层中的方法即可

* Discuss PostController中新增置顶setTop方法，加精方法setWonderful，删除方法setDelete（拉黑，设置状态为2）

  ```java
  // 置顶
  @RequestMapping(path = "/top", method = RequestMethod.POST)
  @ResponseBody
  public String setTop(int id) {
      discussPostService.updateType(id, 1);
  
      // 触发发帖事件
      Event event = new Event()
              .setTopic(TOPIC_PUBLISH)
              .setUserId(hostHolder.getUser().getId())
              .setEntityType(ENTITY_TYPE_POST)
              .setEntityId(id);
      eventProducer.fireEvent(event);
  
      return CommunityUtil.getJSONString(0);
  }
  
  // 加精
  @RequestMapping(path = "/wonderful", method = RequestMethod.POST)
  @ResponseBody
  public String setWonderful(int id) {
      discussPostService.updateStatus(id, 1);
  
      // 触发发帖事件
      Event event = new Event()
              .setTopic(TOPIC_PUBLISH)
              .setUserId(hostHolder.getUser().getId())
              .setEntityType(ENTITY_TYPE_POST)
              .setEntityId(id);
      eventProducer.fireEvent(event);
  
      // 计算帖子分数
      String redisKey = RedisKeyUtil.getPostScoreKey();
      redisTemplate.opsForSet().add(redisKey, id);
  
      return CommunityUtil.getJSONString(0);
  }
  
  // 删除
  @RequestMapping(path = "/delete", method = RequestMethod.POST)
  @ResponseBody
  public String setDelete(int id) {
      discussPostService.updateStatus(id, 2);
  
      // 触发删帖事件
      Event event = new Event()
              .setTopic(TOPIC_DELETE)
              .setUserId(hostHolder.getUser().getId())
              .setEntityType(ENTITY_TYPE_POST)
              .setEntityId(id);
      eventProducer.fireEvent(event);
  
      return CommunityUtil.getJSONString(0);
  }
  ```

* 在EventConsumer中新增handleDeleteMessage消费删帖事件

  ```java
  // 消费删帖事件
  @KafkaListener(topics = {TOPIC_DELETE})
  public void handleDeleteMessage(ConsumerRecord record) {
      if (record == null || record.value() == null) {
          logger.error("消息的内容为空!");
          return;
      }
  
      Event event = JSONObject.parseObject(record.value().toString(), Event.class);
      if (event == null) {
          logger.error("消息格式错误!");
          return;
      }
  
      elasticsearchService.deleteDiscussPost(event.getEntityId());
  }
  ```

* 在security的配置中设置权限，版主可以置顶、加精，管理员可以删帖

  ![image-20221114140457246](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114140457246.png)



## 10.网站数据统计

![image-20221114141502030](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114141502030.png)

![image-20221114142456669](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114142456669.png)

* 在RedisKeyUtil中创建uvkey、daukey

  ```java
  // 单日UV
  public static String getUVKey(String date) {
      return PREFIX_UV + SPLIT + date;
  }
  
  // 区间UV
  public static String getUVKey(String startDate, String endDate) {
      return PREFIX_UV + SPLIT + startDate + SPLIT + endDate;
  }
  
  // 单日活跃用户
  public static String getDAUKey(String date) {
      return PREFIX_DAU + SPLIT + date;
  }
  
  // 区间活跃用户
  public static String getDAUKey(String startDate, String endDate) {
      return PREFIX_DAU + SPLIT + startDate + SPLIT + endDate;
  }
  ```

* 创建DataService，编写相应的统计方法

  ```java
  @Service
  public class DataService {
  
      @Autowired
      private RedisTemplate redisTemplate;
  
      private SimpleDateFormat df = new SimpleDateFormat("yyyyMMdd");
  
      // 将指定的IP计入UV
      public void recordUV(String ip) {
          String redisKey = RedisKeyUtil.getUVKey(df.format(new Date()));
          redisTemplate.opsForHyperLogLog().add(redisKey, ip);
      }
  
      // 统计指定日期范围内的UV
      public long calculateUV(Date start, Date end) {
          if (start == null || end == null) {
              throw new IllegalArgumentException("参数不能为空!");
          }
  
          // 整理该日期范围内的key
          List<String> keyList = new ArrayList<>();
          Calendar calendar = Calendar.getInstance();
          calendar.setTime(start);
          while (!calendar.getTime().after(end)) {
              String key = RedisKeyUtil.getUVKey(df.format(calendar.getTime()));
              keyList.add(key);
              calendar.add(Calendar.DATE, 1);
          }
  
          // 合并这些数据
          String redisKey = RedisKeyUtil.getUVKey(df.format(start), df.format(end));
          redisTemplate.opsForHyperLogLog().union(redisKey, keyList.toArray());
  
          // 返回统计的结果
          return redisTemplate.opsForHyperLogLog().size(redisKey);
      }
  
      // 将指定用户计入DAU
      public void recordDAU(int userId) {
          String redisKey = RedisKeyUtil.getDAUKey(df.format(new Date()));
          redisTemplate.opsForValue().setBit(redisKey, userId, true);
      }
  
      // 统计指定日期范围内的DAU
      public long calculateDAU(Date start, Date end) {
          if (start == null || end == null) {
              throw new IllegalArgumentException("参数不能为空!");
          }
  
          // 整理该日期范围内的key
          List<byte[]> keyList = new ArrayList<>();
          Calendar calendar = Calendar.getInstance();
          calendar.setTime(start);
          while (!calendar.getTime().after(end)) {
              String key = RedisKeyUtil.getDAUKey(df.format(calendar.getTime()));
              keyList.add(key.getBytes());
              calendar.add(Calendar.DATE, 1);
          }
  
          // 进行OR运算
          return (long) redisTemplate.execute(new RedisCallback() {
              @Override
              public Object doInRedis(RedisConnection connection) throws DataAccessException {
                  String redisKey = RedisKeyUtil.getDAUKey(df.format(start), df.format(end));
                  connection.bitOp(RedisStringCommands.BitOperation.OR,
                          redisKey.getBytes(), keyList.toArray(new byte[0][0]));
                  return connection.bitCount(redisKey.getBytes());
              }
          });
      }
  
  }
  ```

* 创建DataInterceptor拦截器统计uv和dau，并将该拦截器加入到webmvcconfig中

  ```java
  @Component
  public class DataInterceptor implements HandlerInterceptor {
  
      @Autowired
      private DataService dataService;
  
      @Autowired
      private HostHolder hostHolder;
  
      @Override
      public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
          // 统计UV
          String ip = request.getRemoteHost();
          dataService.recordUV(ip);
  
          // 统计DAU
          User user = hostHolder.getUser();
          if (user != null) {
              dataService.recordDAU(user.getId());
          }
  
          return true;
      }
  }
  ```

* 创建DataController，配置/data路径只有管理员才能访问

  ```java
  @Controller
  public class DataController {
  
      @Autowired
      private DataService dataService;
  
      // 统计页面
      @RequestMapping(path = "/data", method = {RequestMethod.GET, RequestMethod.POST})
      public String getDataPage() {
          return "/site/admin/data";
      }
  
      // 统计网站UV
      @RequestMapping(path = "/data/uv", method = RequestMethod.POST)
      public String getUV(@DateTimeFormat(pattern = "yyyy-MM-dd") Date start,
                          @DateTimeFormat(pattern = "yyyy-MM-dd") Date end, Model model) {
          long uv = dataService.calculateUV(start, end);
          model.addAttribute("uvResult", uv);
          model.addAttribute("uvStartDate", start);
          model.addAttribute("uvEndDate", end);
          return "forward:/data";
      }
  
      // 统计活跃用户
      @RequestMapping(path = "/data/dau", method = RequestMethod.POST)
      public String getDAU(@DateTimeFormat(pattern = "yyyy-MM-dd") Date start,
                           @DateTimeFormat(pattern = "yyyy-MM-dd") Date end, Model model) {
          long dau = dataService.calculateDAU(start, end);
          model.addAttribute("dauResult", dau);
          model.addAttribute("dauStartDate", start);
          model.addAttribute("dauEndDate", end);
          return "forward:/data";
      }
  
  }
  ```

## 11.定时任务

### Quartz基本使用

<img src="https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114150036003.png" alt="image-20221114150036003" style="zoom:50%;" />

![image-20221114150352923](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114150352923.png)

Quartz是要存到数据库里面的，使用Quartz主要是三个组件。1.job：定义一个任务 2.JobDetail：配置Job  3.trigger：触发器，配置运行时间、频率

* 创建Quartz相关的表，有job表、触发器表等

  ![image-20221114151924374](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114151924374.png)

* 创建Quartz包，创建AlphaJob，job组件

  ```java
  public class AlphaJob implements Job {
      @Override
      public void execute(JobExecutionContext context) throws JobExecutionException {
          System.out.println(Thread.currentThread().getName() + ": execute a quartz job.");
      }
  }
  ```

* 创建QuartzConfig配置JobDetails和Trigger

  ```java
  // FactoryBean可简化Bean的实例化过程:
  // 1.通过FactoryBean封装Bean的实例化过程.
  // 2.将FactoryBean装配到Spring容器里.
  // 3.将FactoryBean注入给其他的Bean.
  // 4.该Bean得到的是FactoryBean所管理的对象实例.
  
  // 配置JobDetail
  // @Bean
  public JobDetailFactoryBean alphaJobDetail() {
      JobDetailFactoryBean factoryBean = new JobDetailFactoryBean();
      factoryBean.setJobClass(AlphaJob.class);
      factoryBean.setName("alphaJob");
      factoryBean.setGroup("alphaJobGroup");
      factoryBean.setDurability(true);
      factoryBean.setRequestsRecovery(true);
      return factoryBean;
  }
  
  // 配置Trigger(SimpleTriggerFactoryBean, CronTriggerFactoryBean)
  // @Bean
  public SimpleTriggerFactoryBean alphaTrigger(JobDetail alphaJobDetail) {
      SimpleTriggerFactoryBean factoryBean = new SimpleTriggerFactoryBean();
      factoryBean.setJobDetail(alphaJobDetail);
      factoryBean.setName("alphaTrigger");
      factoryBean.setGroup("alphaTriggerGroup");
      factoryBean.setRepeatInterval(3000);
      factoryBean.setJobDataMap(new JobDataMap());
      return factoryBean;
  }
  ```

* 配置properties文件，配置后会将任务存入到数据库中

  ```xml
  # QuartzProperties
  spring.quartz.job-store-type=jdbc
  spring.quartz.scheduler-name=communityScheduler
  spring.quartz.properties.org.quartz.scheduler.instanceId=AUTO
  spring.quartz.properties.org.quartz.jobStore.class=org.quartz.impl.jdbcjobstore.JobStoreTX
  spring.quartz.properties.org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.StdJDBCDelegate
  spring.quartz.properties.org.quartz.jobStore.isClustered=true
  spring.quartz.properties.org.quartz.threadPool.class=org.quartz.simpl.SimpleThreadPool
  spring.quartz.properties.org.quartz.threadPool.threadCount=5
  ```

### 热帖排行

![image-20221114154242412](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114154242412.png)

* 在RedisKeyUtil中创建key

  ```java
  // 帖子分数
  public static String getPostScoreKey() {
      return PREFIX_POST + SPLIT + "score";
  }
  ```

* 分别在点赞、加精、评论时候，将帖子的id加入到redis中，等待定时任务的处理

  ```java
  // 计算帖子分数
  String redisKey = RedisKeyUtil.getPostScoreKey();
  redisTemplate.opsForSet().add(redisKey, id);
  ```

* 新建PostScoreRefreshJob（里面是刷新帖子分数的操作）

  ```java
  public class PostScoreRefreshJob implements Job, CommunityConstant {
  
      private static final Logger logger = LoggerFactory.getLogger(PostScoreRefreshJob.class);
  
      @Autowired
      private RedisTemplate redisTemplate;
  
      @Autowired
      private DiscussPostService discussPostService;
  
      @Autowired
      private LikeService likeService;
  
      @Autowired
      private ElasticsearchService elasticsearchService;
  
      // 牛客纪元
      private static final Date epoch;
  
      static {
          try {
              epoch = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").parse("2014-08-01 00:00:00");
          } catch (ParseException e) {
              throw new RuntimeException("初始化牛客纪元失败!", e);
          }
      }
  
      @Override
      public void execute(JobExecutionContext context) throws JobExecutionException {
          String redisKey = RedisKeyUtil.getPostScoreKey();
          BoundSetOperations operations = redisTemplate.boundSetOps(redisKey);
  
          if (operations.size() == 0) {
              logger.info("[任务取消] 没有需要刷新的帖子!");
              return;
          }
  
          logger.info("[任务开始] 正在刷新帖子分数: " + operations.size());
          while (operations.size() > 0) {
              this.refresh((Integer) operations.pop());
          }
          logger.info("[任务结束] 帖子分数刷新完毕!");
      }
  
      private void refresh(int postId) {
          DiscussPost post = discussPostService.findDiscussPostById(postId);
  
          if (post == null) {
              logger.error("该帖子不存在: id = " + postId);
              return;
          }
  
          // 是否精华
          boolean wonderful = post.getStatus() == 1;
          // 评论数量
          int commentCount = post.getCommentCount();
          // 点赞数量
          long likeCount = likeService.findEntityLikeCount(ENTITY_TYPE_POST, postId);
  
          // 计算权重
          double w = (wonderful ? 75 : 0) + commentCount * 10 + likeCount * 2;
          // 分数 = 帖子权重 + 距离天数
          double score = Math.log10(Math.max(w, 1))
                  + (post.getCreateTime().getTime() - epoch.getTime()) / (1000 * 3600 * 24);
          // 更新帖子分数
          discussPostService.updateScore(postId, score);
          // 同步搜索数据
          post.setScore(score);
          elasticsearchService.saveDiscussPost(post);
      }
  
  }
  ```

* 在QuartzConfig中配置JobDetail和Trigger

  ```java
  // 刷新帖子分数任务
  @Bean
  public JobDetailFactoryBean postScoreRefreshJobDetail() {
      JobDetailFactoryBean factoryBean = new JobDetailFactoryBean();
      factoryBean.setJobClass(PostScoreRefreshJob.class);
      factoryBean.setName("postScoreRefreshJob");
      factoryBean.setGroup("communityJobGroup");
      factoryBean.setDurability(true);
      factoryBean.setRequestsRecovery(true);
      return factoryBean;
  }
  
  @Bean
  public SimpleTriggerFactoryBean postScoreRefreshTrigger(JobDetail postScoreRefreshJobDetail) {
      SimpleTriggerFactoryBean factoryBean = new SimpleTriggerFactoryBean();
      factoryBean.setJobDetail(postScoreRefreshJobDetail);
      factoryBean.setName("postScoreRefreshTrigger");
      factoryBean.setGroup("communityTriggerGroup");
      factoryBean.setRepeatInterval(1000 * 60 * 5);
      factoryBean.setJobDataMap(new JobDataMap());
      return factoryBean;
  }
  ```

* 重构查看帖子方法，加入一个orderMode参数，0表示按照时间排序，1表示按照热度排序
* ![image-20221114162143949](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114162143949.png)

## 12.生成长图

![image-20221114162626412](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114162626412.png)

* 安装wk

* 在properties文件中配置

  ```xml
  # wk
  wk.image.command=d:/work/wkhtmltopdf/bin/wkhtmltoimage
  wk.image.storage=d:/work/data/wk-images
  ```

* 创建WkConfig配置类（在项目初始化的时候就创建wk文件的目录）

  ```java
  @Configuration
  public class WkConfig {
  
      private static final Logger logger = LoggerFactory.getLogger(WkConfig.class);
  
      @Value("${wk.image.storage}")
      private String wkImageStorage;
  
      @PostConstruct
      public void init() {
          // 创建WK图片目录
          File file = new File(wkImageStorage);
          if (!file.exists()) {
              file.mkdir();
              logger.info("创建WK图片目录: " + wkImageStorage);
          }
      }
  
  }
  ```

* 在EventConsumer中创建消费分享的事件

  ```java
  // 消费分享事件
  @KafkaListener(topics = TOPIC_SHARE)
  public void handleShareMessage(ConsumerRecord record) {
      if (record == null || record.value() == null) {
          logger.error("消息的内容为空!");
          return;
      }
  
      Event event = JSONObject.parseObject(record.value().toString(), Event.class);
      if (event == null) {
          logger.error("消息格式错误!");
          return;
      }
  
      String htmlUrl = (String) event.getData().get("htmlUrl");
      String fileName = (String) event.getData().get("fileName");
      String suffix = (String) event.getData().get("suffix");
  
      String cmd = wkImageCommand + " --quality 75 "
              + htmlUrl + " " + wkImageStorage + "/" + fileName + suffix;
      try {
          Runtime.getRuntime().exec(cmd);
          logger.info("生成长图成功: " + cmd);
      } catch (IOException e) {
          logger.error("生成长图失败: " + e.getMessage());
      }
  }
  ```

* 创建ShareController，实现生成长图和获取图片（废弃了，后面利用七牛云）的方法。

  ```java
  @Controller
  public class ShareController implements CommunityConstant {
  
      private static final Logger logger = LoggerFactory.getLogger(ShareController.class);
  
      @Autowired
      private EventProducer eventProducer;
  
      @Value("${community.path.domain}")
      private String domain;
  
      @Value("${server.servlet.context-path}")
      private String contextPath;
  
      @Value("${wk.image.storage}")
      private String wkImageStorage;
  
      @Value("${qiniu.bucket.share.url}")
      private String shareBucketUrl;
  
      @RequestMapping(path = "/share", method = RequestMethod.GET)
      @ResponseBody
      public String share(String htmlUrl) {
          // 文件名
          String fileName = CommunityUtil.generateUUID();
  
          // 异步生成长图
          Event event = new Event()
                  .setTopic(TOPIC_SHARE)
                  .setData("htmlUrl", htmlUrl)
                  .setData("fileName", fileName)
                  .setData("suffix", ".png");
          eventProducer.fireEvent(event);
  
          // 返回访问路径
          Map<String, Object> map = new HashMap<>();
  //        map.put("shareUrl", domain + contextPath + "/share/image/" + fileName);
          map.put("shareUrl", shareBucketUrl + "/" + fileName);
  
          return CommunityUtil.getJSONString(0, null, map);
      }
      
      // 废弃后续利用七牛云
      // 获取长图
      @RequestMapping(path = "/share/image/{fileName}", method = RequestMethod.GET)
      public void getShareImage(@PathVariable("fileName") String fileName, HttpServletResponse response) {
          if (StringUtils.isBlank(fileName)) {
              throw new IllegalArgumentException("文件名不能为空!");
          }
  
          response.setContentType("image/png");
          File file = new File(wkImageStorage + "/" + fileName + ".png");
          try {
              OutputStream os = response.getOutputStream();
              FileInputStream fis = new FileInputStream(file);
              byte[] buffer = new byte[1024];
              int b = 0;
              while ((b = fis.read(buffer)) != -1) {
                  os.write(buffer, 0, b);
              }
          } catch (IOException e) {
              logger.error("获取长图失败: " + e.getMessage());
          }
      }
  }
  ```

## 13.将文件上传到云服务器

![image-20221114165312419](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114165312419.png)

* 在properties文件中添加配置

  ```xml
  # qiniu
  qiniu.key.access=LniTyd17oFvp_Sa6oY96fPXwMV9-8cMnH0-_JS_M
  qiniu.key.secret=Famwup60RHwU2n_ZS_hbqMJLWk8LfMvwexpFPCA
  qiniu.bucket.header.name=miyukiheader
  quniu.bucket.header.url=http://header.miyuki.ink
  qiniu.bucket.share.name=miyukishare
  qiniu.bucket.share.url=http://share.miyuki.ink
  ```

* 在UserController中注入相应的属性

  ![image-20221114180305648](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114180305648.png)



* 废弃以前的上传图片和获得图片的方法

* 重写getSetting方法，每次上传都要有凭证

  ```java
  @LoginRequired
  @RequestMapping(path = "/setting", method = RequestMethod.GET)
  public String getSettingPage(Model model) {
      // 上传文件名称
      String fileName = CommunityUtil.generateUUID();
      // 设置响应信息
      StringMap policy = new StringMap();
      policy.put("returnBody", CommunityUtil.getJSONString(0));
      // 生成上传凭证
      Auth auth = Auth.create(accessKey, secretKey);
      String uploadToken = auth.uploadToken(headerBucketName, fileName, 3600, policy);
  
      model.addAttribute("uploadToken", uploadToken);
      model.addAttribute("fileName", fileName);
  
      return "/site/setting";
  }
  ```

* 重写updateHeader方法

  ```java
  // 更新头像路径
  @RequestMapping(path = "/header/url", method = RequestMethod.POST)
  @ResponseBody
  public String updateHeaderUrl(String fileName) {
      if (StringUtils.isBlank(fileName)) {
          return CommunityUtil.getJSONString(1, "文件名不能为空!");
      }
  
      String url = headerBucketUrl + "/" + fileName;
      userService.updateHeader(hostHolder.getUser().getId(), url);
  
      return CommunityUtil.getJSONString(0);
  }
  ```

* 通过前端js上传图片，这是客户端上传

  ![image-20221114181211264](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114181211264.png)

* 在ShareController中是从服务端传图片到七牛云

  ![image-20221114181524309](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114181524309.png)

* 废弃从本地获得图片

* 在EventConsumer的消费分享方法中，新增异步定时器监视文件，传输文件到七牛云

  ![image-20221114182503255](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114182503255.png)

  下面这个方法很精彩，将future设置到线程中，如果超时会去取消任务，如果次数超过3次也会取消任务，调用future.cancle()。

  ```java
  class UploadTask implements Runnable {
  
      // 文件名称
      private String fileName;
      // 文件后缀
      private String suffix;
      // 启动任务的返回值
      private Future future;
      // 开始时间
      private long startTime;
      // 上传次数
      private int uploadTimes;
  
      public UploadTask(String fileName, String suffix) {
          this.fileName = fileName;
          this.suffix = suffix;
          this.startTime = System.currentTimeMillis();
      }
  
      public void setFuture(Future future) {
          this.future = future;
      }
  
      @Override
      public void run() {
          // 生成失败
          if (System.currentTimeMillis() - startTime > 30000) {
              logger.error("执行时间过长,终止任务:" + fileName);
              future.cancel(true);
              return;
          }
          // 上传失败
          if (uploadTimes >= 3) {
              logger.error("上传次数过多,终止任务:" + fileName);
              future.cancel(true);
              return;
          }
  
          String path = wkImageStorage + "/" + fileName + suffix;
          File file = new File(path);
          if (file.exists()) {
              logger.info(String.format("开始第%d次上传[%s].", ++uploadTimes, fileName));
              // 设置响应信息
              StringMap policy = new StringMap();
              policy.put("returnBody", CommunityUtil.getJSONString(0));
              // 生成上传凭证
              Auth auth = Auth.create(accessKey, secretKey);
              String uploadToken = auth.uploadToken(shareBucketName, fileName, 3600, policy);
              // 指定上传机房
              UploadManager manager = new UploadManager(new Configuration(Zone.zone1()));
              try {
                  // 开始上传图片
                  Response response = manager.put(
                          path, fileName, uploadToken, null, "image/" + suffix, false);
                  // 处理响应结果
                  JSONObject json = JSONObject.parseObject(response.bodyString());
                  if (json == null || json.get("code") == null || !json.get("code").toString().equals("0")) {
                      logger.info(String.format("第%d次上传失败[%s].", uploadTimes, fileName));
                  } else {
                      logger.info(String.format("第%d次上传成功[%s].", uploadTimes, fileName));
                      future.cancel(true);
                  }
              } catch (QiniuException e) {
                  logger.info(String.format("第%d次上传失败[%s].", uploadTimes, fileName));
              }
          } else {
              logger.info("等待图片生成[" + fileName + "].");
          }
      }
  }
  ```

## 14.优化网站性能

![image-20221114183847718](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114183847718.png)

![image-20221114190249998](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114190249998.png)

### caffeine缓存

使用的是Caffeine中LoadingCache（同步缓存），即多个线程线程访问缓存时，若缓存没有对应的数据，只允许一个线程去数据库中取数据，其他线程则排队等候。由于LoadingCache是线程同步的，所以即使Caffeine中的缓存同时失效了，有大量请求访问热点帖子的时候，它也不会出现缓存雪崩的问题。因此认为对于缓存热点帖子这一功能而言，应该只需要用Caffeine做缓存就足够了，可以不使用Redis作二级缓存，毕竟热点贴的更新频率慢。如果有其他合适的需求，可以再考虑使用多级缓存的策略。

* 配置properties文件

  ```
  # caffeine 缓存15条数据
  caffeine.posts.max-size=15
  caffeine.posts.expire-seconds=180
  ```

* 使用caffenine优化DiscussPostService，缓存按热度排序的帖子15条（可以根据分页条件查询，最多缓存15条），缓存帖子总数。

  ```java
  @Value("${caffeine.posts.max-size}")
  private int maxSize;
  
  @Value("${caffeine.posts.expire-seconds}")
  private int expireSeconds;
  
  // Caffeine核心接口: Cache, LoadingCache(没有从数据库查), AsyncLoadingCache（异步查数据库）
  
  // 帖子列表缓存
  private LoadingCache<String, List<DiscussPost>> postListCache;
  
  // 帖子总数缓存
  private LoadingCache<Integer, Integer> postRowsCache;
  
  @PostConstruct
  public void init() {
      // 初始化帖子列表缓存
      postListCache = Caffeine.newBuilder()
              .maximumSize(maxSize)
              .expireAfterWrite(expireSeconds, TimeUnit.SECONDS)
              .build(new CacheLoader<String, List<DiscussPost>>() {
                  @Nullable
                  @Override
                  public List<DiscussPost> load(@NonNull String key) throws Exception {
                      if (key == null || key.length() == 0) {
                          throw new IllegalArgumentException("参数错误!");
                      }
  
                      String[] params = key.split(":");
                      if (params == null || params.length != 2) {
                          throw new IllegalArgumentException("参数错误!");
                      }
  
                      int offset = Integer.valueOf(params[0]);
                      int limit = Integer.valueOf(params[1]);
  
                      // 二级缓存: Redis -> mysql
  
                      logger.debug("load post list from DB.");
                      return discussPostMapper.selectDiscussPosts(0, offset, limit, 1);
                  }
              });
      // 初始化帖子总数缓存
      postRowsCache = Caffeine.newBuilder()
              .maximumSize(maxSize)
              .expireAfterWrite(expireSeconds, TimeUnit.SECONDS)
              .build(new CacheLoader<Integer, Integer>() {
                  @Nullable
                  @Override
                  public Integer load(@NonNull Integer key) throws Exception {
                      logger.debug("load post rows from DB.");
                      return discussPostMapper.selectDiscussPostRows(key);
                  }
              });
  }
  ```

* 改造findDiscussPost方法，userid = 0说明访问首页，并且选择热度排序，就从caffeine缓存中取出数据，userid不为0是查询某个用户的帖子。

  ```java
  public List<DiscussPost> findDiscussPosts(int userId, int offset, int limit, int orderMode) {
      if (userId == 0 && orderMode == 1) {
          return postListCache.get(offset + ":" + limit);
      }
  
      logger.debug("load post list from DB.");
      return discussPostMapper.selectDiscussPosts(userId, offset, limit, orderMode);
  }
  ```

* 改造findDiscussPostRows方法，从缓存中拿出来总的贴子数，方便分页。

  ```java
  public int findDiscussPostRows(int userId) {
      if (userId == 0) {
          return postRowsCache.get(userId);
      }
  
      logger.debug("load post rows from DB.");
      return discussPostMapper.selectDiscussPostRows(userId);
  }
  ```

* 使用Jmeter进行压测

  ![image-20221114200250584](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114200250584.png)

  ![image-20221114200412508](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114200412508.png)

  ![image-20221114200432378](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114200432378.png)

  ![image-20221114200518278](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114200518278.png)

  ![image-20221114200543458](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114200543458.png)

  ![image-20221114200622778](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114200622778.png)

  ![image-20221114200656561](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114200656561.png)

  **关注吞吐量**

  ![image-20221114200746403](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114200746403.png)

  * 不加入caffeine缓存吞吐量仅为每秒10个左右

    ![image-20221114200920185](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114200920185.png)

  * 加入caffeine缓存吞吐量达到190是之前的接近20倍

    ![image-20221114201112200](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114201112200.png)

## 15.项目监控

![image-20221114205721044](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114205721044.png)

结果是json格式的数据，可以查看各种各样的信息，如bean信息等

![image-20221114210135052](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114210135052.png)

* 配置xml文件（一共有20多个断电可以配置，访问路径就是/community/actuator/health (caches等等)）

  ```xml
  # actuator
  management.endpoints.web.exposure.include=*
  management.endpoints.web.exposure.exclude=info,caches
  ```

* 示例（/community/actuator/database即可访问）

  ```java
  @Component
  @Endpoint(id = "database")
  public class DatabaseEndpoint {
  
      private static final Logger logger = LoggerFactory.getLogger(DatabaseEndpoint.class);
  
      @Autowired
      private DataSource dataSource;
  
      @ReadOperation
      public String checkConnection() {
          try (
                  Connection conn = dataSource.getConnection();
          ) {
              return CommunityUtil.getJSONString(0, "获取连接成功!");
          } catch (SQLException e) {
              logger.error("获取连接失败:" + e.getMessage());
              return CommunityUtil.getJSONString(1, "获取连接失败!");
          }
      }
  
  }
  ```

* 只有管理员可以访问

  ![image-20221114210100128](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114210100128.png)

## 16.项目部署



## 项目优化

1. 将验证码从session移入到redis中，使用cookie，传给客户端。

2. 将登录凭证存入到redis中，修改登陆凭证也是修改redis中的状态，不再麻烦数据库了，登录凭证表可以废弃了。

3. 一二两条具体看第6节redis中的优化登录模块

4. Kafka的使用，发送系统消息用到kafka（用户xxx给你点了个赞、关注、评论），消息数据格式也不一样（普通的内容就是类似于：你好啊，然后系统的信息在数据库中是一条json格式的数据，存储各种信息，userid呀，评论类型呀等等），使用过滤器messageInterceptor，访问任何一个页面都经过过滤器来实时显示未读的消息。

   ![image-20221113113911288](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113113911288.png)

   ![image-20221113154053276](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113154053276.png)

5. es中存储帖子，通过es提供的方法可以高亮关键字可以看上面的代码，es极大的优化了搜索效率。

6. SpringSecurity改造，使用Spring+jwt的方法替代ticket（就是将ticket变成jwt，更加实用安全了，毕竟jwt是很多人在用的工具了，而ticket是我们自己生成的随机字符串而已），还有就是SpringSecurity配置的更加彻底，之前项目中是取巧的，只是用了自己实现的HostHolder，存放用户（user），改造了LoginTicketInterceptor这个类在之中手动添加了Authentication对象保存到SecurityContext中，没有真正的实现一个AuthenticationProvider来实现认证的逻辑，相当于不是在Filter中实现的登录认证，没有用到SpringSecurity而是在Interceptor中自行实现的，后续改进了，使得SpringSecurity使用的更完整了。项目中权限设计也十分简陋，只有3中角色，后续能进行更细的权限分配。

   <img src="https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221113211550019.png" alt="image-20221113211550019" style="zoom:50%;" />

7. 使用Quartz结合redis（每次评论、点赞、加精帖子都会存放一条redis数据，使用了set集合类型，存放了所有有变动的帖子id）进行定时统计帖子的分数，公式的原则就是时间越长分数越低，点赞、评论越多分数越高。

8. 使用定时任务线程池（ThreadPoolTaskScheduler），在EventConsumer的消费分享方法中，新增异步定时器监视文件，传输文件到七牛云

   ![image-20221114182503255](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114182503255.png)

   下面这个方法很精彩，将future设置到线程中，如果超时会去取消任务，如果次数超过3次也会取消任务，调用future.cancle()。

9. 将头像存在七牛云而不是本地

