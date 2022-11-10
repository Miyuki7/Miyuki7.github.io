作者：是瑶瑶公主呀
链接：https://www.nowcoder.com/discuss/481089?type=5
来源：牛客网



## 1.数据库表

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

| 字段    | 类型      | 备注                     |
| ------- | --------- | ------------------------ |
| id      | int       | 主键、自增               |
| user_id | int       | 登录用户 id              |
| ticket  | varchar   | 登录凭证，随机字符串     |
| status  | int       | 登录状态：0 有效、1 无效 |
| expired | timestamp | 过期时间                 |

#### 消息表 message

| 字段            | 类型      | 备注                                  |
| --------------- | --------- | ------------------------------------- |
| id              | int       | 主键、自增                            |
| from_id         | int       | 发消息的 id，创建索引                 |
| to_id           | int       | 收消息的 id，创建索引                 |
| conversation_id | varchar   | 会话 id，由通信双方 id 拼接，创建索引 |
| content         | text      | 消息内容                              |
| status          | int       | 消息状态：0 未读、1 已读、2 删除      |
| create_time     | timestamp | 消息发送时间                          |



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
  
  - `<if test="userId!=0">`  userID 为 0 时不使用，按照类型，发帖时间排序。
  
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

拼接后的访问路径是/index?current = 1这种

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

### 显示评论（comment 表）

创建 comment 表对应的实体类 Comment。

创建 CommentMapper 接口

- 新增 `selectCommentsByEntity` 方法，根据实体查询一页的评论数据。 
- 新增 `selectCountByEntity` 方法，根据实体查询评论的数量。 
- 在 `comment-mapper.xml` 配置 select 语句。 

创建 CommentService 类

- 新增 `findCommentByEntity` 方法，调用 CommentMapper 的 `selectCommentByEntity` 方法。 
- 新增 `findCommentCount` 方法，调用 CommentMapper 的 `selectCountByEntity` 方法。 

在 DiscussPostController 的 `getDiscussPost` 方法中增加查询帖子评论和回复的逻辑，将结果存储在 Model 对象。

【问题】sql 的 xml 文件中绑定参数时，应传入实体类属性名，拼错成数据库字段名（entityId 写成 entity_id）。

------

### 添加评论

在 CommentMapper 接口新增 `insertComment` 方法，添加评论数据，在 `comment-mapper` 配置对应 sql。

在 DiscussPostMapper 接口新增 `updateCommentCount` 方法，增加评论数量，在 `discusspost-mapper` 配置对应 sql。

在 DiscussPostService 类新增 `updateCommentCount` 方法，调用 DiscussPostMapper 的 `updateCommentCount` 方法。

在 CommentService 类新增 `addComment` 方法，调用 CommentMapper 的 `insertComment` 新增评论，并调用 DiscussPostService 的 `updateCommentCount` 更新评论数量，使用 `@Transactional` 注解保证事务。

创建 CommentController 类，新增 `addComment` 方法，从 hostHolder 获取用户信息，然后调用 CommentService 的 `addComment` 方法添加评论。

【问题】sql 的 xml 文件中绑定参数时，应传入实体类属性名，拼错成数据库字段名（entityId 写成 entity_id）。

------

### 显示私信列表 （message 表）

创建对应 message 表的实体类 Message。

创建 MessageMapper 接口，增加查询会话列表、会话数量、私信列表、私信数量、未读私信数量等方法，在 `message-mapper.xml` 中配置对应的 sql。

创建 MessageService，调用 MessageMapper 中的方法。

创建 MessgaeController

- 新增 `getLetterList` 方法，将会话列表信息存储到 Model 对象，返回 `letter` 视图。 
- 新增 `getLetterDetail` 方法，将每个会话具体的私信信息存储到 Model 对象，返回 `letter-datail` 视图。 

------

### 发送私信

在 MessageMapper 

- 新增 `insertMessage` 方法插入私信记录，在 `message-mapper.xml`  配置 insert 语句。 
- 新增 `updateMessgae` 方法修改私信状态，在 `message-mapper.xml`  配置 update 语句，利用 foreach 动态 sql。 

在 MessageService

- 新增 `addMessage` 发送私信方法，过滤敏感词后，调用 MessageMapper 的 `insertMessage` 。
- 新增 `readMessage` 方法读取信息，调用MessageMapper 的 `updateMessgae` 更新私信的状态为 1。

在 MessageController 

- 新增 `getLetterIds` 方法，将私信集合中未读私信的 id 添加到 List 集合并返回，在 `getLetterDetail` 方法调用该方法设置已读。
- 新增 `sendLetter` 发送私信方法，设置私信信息后调用 MessageService 的 `addMessage` 发送。

------

### 统一异常处理

在 HomeController 中增加 `getErrorPage` 方法，返回错误页面。

创建 ExceptionAdvice 类

- 加上 `@ControllerAdvice` 注解，表示该类是 Controller 的全局配置类。 
- 创建 `handleException` 方法，加上 `@ExceptionHandler` 注解，该方法在 Controller 出现异常后调用，处理捕获异常。如果是异步请求返回一个 JSON 数据，否则重定向至 HomeController 的 `getErrorPage` 方法。 

------

### 统一日志处理

在 `pom.xml` 引入 aspectj 的依赖。

创建 ServiceLogAspect 类，添加 `@Aspect` 切面注解，配置切入点表达式，拦截所有 service 包下的方法，利用 `@Before` 记录日志。

## 项目优化

1. 将验证码从session移入到redis中
2. 将登录凭证存入到redis中，修改登陆凭证也是修改redis中的状态，不再麻烦数据库了。

3. 将头像存在七牛云而不是本地