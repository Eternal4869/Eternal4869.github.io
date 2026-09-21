# Controller测试

```java
package com.heima.admin.controller.v1;

import com.heima.admin.service.AdUserLoginService;
import com.heima.model.admin.dtos.AdUserDto;
import com.heima.model.common.dtos.ResponseResult;
import com.heima.model.common.enums.AppHttpCodeEnum;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

/**
 * 管理端登录控制器单元测试。
 * 控制器只做参数空/空白校验，校验通过后原样透传 service.login 的返回结果，
 * 因此校验分支断言返回 PARAM_INVALID 且不调用 service，合法参数断言透传结果与调用行为。
 */
@DisplayName("AdUserLoginController 管理端登录控制器")
@ExtendWith(MockitoExtension.class)
class AdUserLoginControllerTest {

    @Mock
    private AdUserLoginService adUserLoginService;

    private AdUserLoginController adUserLoginController;

    @BeforeEach
    void setUp() {
        // 控制器使用 Lombok @RequiredArgsConstructor 生成带 final 依赖的构造器，手动 new 并注入 mock
        adUserLoginController = new AdUserLoginController(adUserLoginService);
    }

    /**
     * 场景：登录参数合法，控制器应把请求透传给 service，并原样返回 service 的成功结果
     */
    @Test
    @DisplayName("login 参数合法：透传 service 并返回其成功结果")
    void login_validDto_shouldCallServiceAndReturnItsResult() {
        // 构造合法登录参数
        AdUserDto dto = new AdUserDto();
        dto.setName("admin");
        dto.setPassword("123456");

        // service 返回一个成功结果（含 user + token 的 map）
        java.util.Map<String, Object> data = new java.util.HashMap<>();
        data.put("token", "mock-token");
        ResponseResult serviceResult = ResponseResult.okResult(data);
        when(adUserLoginService.login(dto)).thenReturn(serviceResult);

        ResponseResult result = adUserLoginController.login(dto);

        // 控制器不重新包装，直接返回 service 的同一个实例
        assertThat(result).isSameAs(serviceResult);
        assertThat(result.getCode()).isEqualTo(AppHttpCodeEnum.SUCCESS.getCode());
        assertThat(result.getData()).isEqualTo(data);
        // 确认 service 恰好被调用一次，且传入的就是该 DTO
        verify(adUserLoginService).login(dto);
    }

    /**
     * 场景：登录参数合法但 service 校验失败（如密码错误），控制器应原样透传错误结果，不做额外包装
     */
    @Test
    @DisplayName("login service 返回错误结果：控制器原样透传")
    void login_serviceReturnsError_shouldPassThroughUnchanged() {
        AdUserDto dto = new AdUserDto();
        dto.setName("admin");
        dto.setPassword("wrong-password");

        ResponseResult serviceResult = ResponseResult.errorResult(AppHttpCodeEnum.LOGIN_PASSWORD_ERROR);
        when(adUserLoginService.login(dto)).thenReturn(serviceResult);

        ResponseResult result = adUserLoginController.login(dto);

        assertThat(result).isSameAs(serviceResult);
        assertThat(result.getCode()).isEqualTo(AppHttpCodeEnum.LOGIN_PASSWORD_ERROR.getCode());
        assertThat(result.getErrorMessage()).isEqualTo(AppHttpCodeEnum.LOGIN_PASSWORD_ERROR.getErrorMessage());
        verify(adUserLoginService).login(dto);
    }

    /**
     * 场景：请求体为 null，控制器应直接返回参数无效，不调用 service
     */
    @Test
    @DisplayName("login 请求体为 null：返回 PARAM_INVALID 且不调用 service")
    void login_nullDto_shouldReturnParamInvalid() {
        ResponseResult result = adUserLoginController.login(null);

        assertThat(result.getCode()).isEqualTo(AppHttpCodeEnum.PARAM_INVALID.getCode());
        assertThat(result.getErrorMessage()).isEqualTo(AppHttpCodeEnum.PARAM_INVALID.getErrorMessage());
        verify(adUserLoginService, never()).login(any(AdUserDto.class));
    }

    /**
     * 场景：用户名为空字符串，控制器应返回参数无效，不调用 service
     */
    @Test
    @DisplayName("login 用户名为空：返回 PARAM_INVALID 且不调用 service")
    void login_blankName_shouldReturnParamInvalid() {
        AdUserDto dto = new AdUserDto();
        dto.setName("   ");
        dto.setPassword("123456");

        ResponseResult result = adUserLoginController.login(dto);

        assertThat(result.getCode()).isEqualTo(AppHttpCodeEnum.PARAM_INVALID.getCode());
        assertThat(result.getErrorMessage()).isEqualTo(AppHttpCodeEnum.PARAM_INVALID.getErrorMessage());
        verify(adUserLoginService, never()).login(any(AdUserDto.class));
    }

    /**
     * 场景：密码为空白(null 或空串都算 blank)，控制器应返回参数无效，不调用 service
     */
    @Test
    @DisplayName("login 密码为空：返回 PARAM_INVALID 且不调用 service")
    void login_blankPassword_shouldReturnParamInvalid() {
        AdUserDto dto = new AdUserDto();
        dto.setName("admin");
        dto.setPassword("");

        ResponseResult result = adUserLoginController.login(dto);

        assertThat(result.getCode()).isEqualTo(AppHttpCodeEnum.PARAM_INVALID.getCode());
        assertThat(result.getErrorMessage()).isEqualTo(AppHttpCodeEnum.PARAM_INVALID.getErrorMessage());
        verify(adUserLoginService, never()).login(any(AdUserDto.class));
    }
}
```

# Repository测试

- `H2MapperSupport.java`
```java
package com.heima.admin.mapper.support;

import com.baomidou.mybatisplus.core.MybatisConfiguration;
import com.baomidou.mybatisplus.extension.spring.MybatisSqlSessionFactoryBean;
import org.apache.ibatis.session.SqlSessionFactory;
import org.h2.jdbcx.JdbcDataSource;
import org.h2.tools.RunScript;

import java.io.InputStreamReader;
import java.nio.charset.StandardCharsets;
import java.sql.Connection;

/**
 * Mapper 层 H2 集成测试支持类。
 * 以 H2 内存库 + MyBatis-Plus 真实 SqlSessionFactory 构建会话，
 * 供 AdUserMapperTest 直接获取真实 BaseMapper 能力进行 CRUD 验证。
 * 注意：类名不以 Test 结尾，避免被 surefire 误当成测试类执行。
 */
public final class H2MapperSupport {

    private static volatile SqlSessionFactory factory;

    private H2MapperSupport() {
    }

    public static synchronized SqlSessionFactory factory() {
        if (factory == null) {
            factory = build();
        }
        return factory;
    }

    private static SqlSessionFactory build() {
        try {
            JdbcDataSource ds = new JdbcDataSource();
            ds.setURL("jdbc:h2:mem:admin_test;MODE=MySQL;DB_CLOSE_DELAY=-1;DATABASE_TO_LOWER=TRUE");
            ds.setUser("sa");
            ds.setPassword("");
            try (Connection conn = ds.getConnection()) {
                RunScript.execute(conn, new InputStreamReader(
                        H2MapperSupport.class.getResourceAsStream("/db/schema-admin.sql"), StandardCharsets.UTF_8));
            }
            MybatisSqlSessionFactoryBean fb = new MybatisSqlSessionFactoryBean();
            fb.setDataSource(ds);
            MybatisConfiguration cfg = new MybatisConfiguration();
            cfg.setMapUnderscoreToCamelCase(true);
            fb.setConfiguration(cfg);
            return fb.getObject();
        } catch (Exception e) {
            throw new IllegalStateException(e);
        }
    }

    /**
     * 获取指定 Mapper 的真实实现。
     * 由于手工构建的 SqlSessionFactory 不会像 MapperScannerConfigurer 那样自动注册 Mapper 接口，
     * 首次使用时需将其注册进 MybatisPlusMapperRegistry（等价于 @MapperScan 的行为），
     * 否则会抛 BindingException: ... is not known to the MybatisPlusMapperRegistry。
     */
    public static <T> T mapper(Class<T> type) {
        SqlSessionFactory f = factory();
        if (!f.getConfiguration().hasMapper(type)) {
            f.getConfiguration().addMapper(type);
        }
        return f.openSession(true).getMapper(type);
    }
}
```

- `AdUserMapperTest.java`

```java
package com.heima.admin.mapper;

import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.heima.admin.mapper.support.H2MapperSupport;
import com.heima.model.admin.pojos.AdUser;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * AdUserMapper(BaseMapper&lt;AdUser&gt;) 的 H2 内存库集成测试。
 * 通过 H2MapperSupport 构建真实 MyBatis-Plus SqlSessionFactory 与 Mapper，
 * 验证 BaseMapper 的 selectById / selectOne / selectList / insert / updateById / deleteById 能力，
 * 以及下划线列名到驼峰属性的自动映射（如 login_time -> loginTime）。
 *
 * 说明：所有会写库的用例都使用独立的高位 id 并自清理（finally 中删除），
 * 只读用例也只按种子记录名过滤，保证用例相互独立、可乱序执行。
 */
@DisplayName("AdUserMapper H2 集成测试")
class AdUserMapperTest {

    /**
     * 场景：按主键查询存在的种子数据（id=2 admin 记录），字段应与种子一致
     */
    @Test
    @DisplayName("selectById 命中种子记录：返回完整字段")
    void selectById_existingId_shouldReturnSeedUser() {
        AdUser admin = H2MapperSupport.mapper(AdUserMapper.class).selectById(2);

        assertThat(admin).isNotNull();
        assertThat(admin.getName()).isEqualTo("admin");
        assertThat(admin.getPassword()).isEqualTo("5d4e1a406d4a9edbf7b4f10c2a390405");
        assertThat(admin.getSalt()).isEqualTo("123abc");
        assertThat(admin.getNickname()).isEqualTo("ad");
        assertThat(admin.getPhone()).isEqualTo("13320325528");
        assertThat(admin.getEmail()).isEqualTo("admin@qq.com");
        assertThat(admin.getStatus()).isTrue();
        // 下划线列 login_time/created_time 应映射为驼峰属性
        assertThat(admin.getLoginTime()).isNotNull();
        assertThat(admin.getCreatedTime()).isNotNull();
    }

    /**
     * 场景：按主键查询不存在的 id，应返回 null
     */
    @Test
    @DisplayName("selectById 未命中：返回 null")
    void selectById_notExistId_shouldReturnNull() {
        AdUser notExist = H2MapperSupport.mapper(AdUserMapper.class).selectById(9999);
        assertThat(notExist).isNull();
    }

    /**
     * 场景：按用户名(wrapper eq)查询，应命中唯一种子记录
     */
    @Test
    @DisplayName("selectOne 按用户名查询：命中 wukong 记录")
    void selectOne_byName_shouldReturnMatchedUser() {
        AdUser wukong = H2MapperSupport.mapper(AdUserMapper.class)
                .selectOne(Wrappers.<AdUser>lambdaQuery().eq(AdUser::getName, "wukong"));

        assertThat(wukong).isNotNull();
        assertThat(wukong.getId()).isEqualTo(1);
        assertThat(wukong.getPassword()).isEqualTo("123");
        assertThat(wukong.getSalt()).isNull();
        assertThat(wukong.getNickname()).isEqualTo("mo");
        assertThat(wukong.getStatus()).isTrue();
    }

    /**
     * 场景：按不存在的用户名查询，应返回 null
     */
    @Test
    @DisplayName("selectOne 按不存在用户名查询：返回 null")
    void selectOne_byNameNotExist_shouldReturnNull() {
        AdUser notExist = H2MapperSupport.mapper(AdUserMapper.class)
                .selectOne(Wrappers.<AdUser>lambdaQuery().eq(AdUser::getName, "ghost_user"));

        assertThat(notExist).isNull();
    }

    /**
     * 场景：selectList 按种子用户名集合过滤，应返回全部 3 条种子记录
     */
    @Test
    @DisplayName("selectList 按种子用户名过滤：返回 3 条记录")
    void selectList_bySeedNames_shouldReturnAllSeedUsers() {
        List<AdUser> users = H2MapperSupport.mapper(AdUserMapper.class)
                .selectList(Wrappers.<AdUser>lambdaQuery()
                        .in(AdUser::getName, "admin", "wukong", "guest"));

        assertThat(users).hasSize(3);
        assertThat(users)
                .extracting(AdUser::getName)
                .containsExactlyInAnyOrder("admin", "wukong", "guest");
    }

    /**
     * 场景：insert 新增管理员（id 由 AUTO_INCREMENT 自增生成），返回影响行数 1，
     * 随后按唯一名称查回实体，验证字段已完整落库
     */
    @Test
    @DisplayName("insert 新增记录：落库后可查回")
    void insert_newUser_shouldPersistAndBeSelectable() {
        AdUserMapper mapper = H2MapperSupport.mapper(AdUserMapper.class);
        AdUser user = new AdUser();
        user.setName("mapper_insert_user");
        user.setPassword("insert_pwd_md5");
        user.setSalt("ins");
        user.setNickname("ins");
        user.setPhone("13900000001");
        user.setStatus(true);
        user.setEmail("insert@heima.com");
        try {
            int rows = mapper.insert(user);

            assertThat(rows).isEqualTo(1);
            AdUser saved = mapper.selectOne(Wrappers.<AdUser>lambdaQuery().eq(AdUser::getName, "mapper_insert_user"));
            assertThat(saved).isNotNull();
            assertThat(saved.getId()).isNotNull();
            assertThat(saved.getName()).isEqualTo("mapper_insert_user");
            assertThat(saved.getPassword()).isEqualTo("insert_pwd_md5");
            assertThat(saved.getSalt()).isEqualTo("ins");
            assertThat(saved.getNickname()).isEqualTo("ins");
            assertThat(saved.getPhone()).isEqualTo("13900000001");
            assertThat(saved.getEmail()).isEqualTo("insert@heima.com");
            assertThat(saved.getStatus()).isTrue();
        } finally {
            // 清理，避免污染共享内存库中的其它用例
            mapper.delete(Wrappers.<AdUser>lambdaQuery().eq(AdUser::getName, "mapper_insert_user"));
        }
    }

    /**
     * 场景：updateById 修改已存在记录的可变字段，应只影响该行且落库生效
     */
    @Test
    @DisplayName("updateById 修改字段：重新查询可见变更")
    void updateById_shouldPersistChanges() {
        AdUserMapper mapper = H2MapperSupport.mapper(AdUserMapper.class);
        AdUser user = new AdUser();
        user.setName("mapper_update_user");
        user.setPassword("update_pwd_md5");
        user.setSalt("upd");
        user.setNickname("original");
        user.setPhone("13900000010");
        user.setStatus(true);
        mapper.insert(user);
        try {
            // 通过唯一名称查回实体，取得自增生成的真实 id
            AdUser persisted = mapper.selectOne(Wrappers.<AdUser>lambdaQuery().eq(AdUser::getName, "mapper_update_user"));
            assertThat(persisted).isNotNull();
            // 修改若干字段后执行 updateById
            persisted.setNickname("renamed_nick");
            persisted.setPhone("13900000011");
            persisted.setPassword("new_pwd_md5");
            persisted.setStatus(false);

            int rows = mapper.updateById(persisted);

            assertThat(rows).isEqualTo(1);
            AdUser reloaded = mapper.selectById(persisted.getId());
            assertThat(reloaded).isNotNull();
            assertThat(reloaded.getNickname()).isEqualTo("renamed_nick");
            assertThat(reloaded.getPhone()).isEqualTo("13900000011");
            assertThat(reloaded.getPassword()).isEqualTo("new_pwd_md5");
            assertThat(reloaded.getStatus()).isFalse();
            // 未修改字段保持不变
            assertThat(reloaded.getName()).isEqualTo("mapper_update_user");
            assertThat(reloaded.getSalt()).isEqualTo("upd");
        } finally {
            mapper.delete(Wrappers.<AdUser>lambdaQuery().eq(AdUser::getName, "mapper_update_user"));
        }
    }

    /**
     * 场景：deleteById 删除新增记录，删除后按 id 查不到
     */
    @Test
    @DisplayName("deleteById 删除记录：删除后查不到")
    void deleteById_shouldRemoveRow() {
        AdUserMapper mapper = H2MapperSupport.mapper(AdUserMapper.class);
        AdUser user = new AdUser();
        user.setName("mapper_delete_user");
        user.setPassword("delete_pwd_md5");
        user.setNickname("del");
        mapper.insert(user);

        AdUser persisted = mapper.selectOne(Wrappers.<AdUser>lambdaQuery().eq(AdUser::getName, "mapper_delete_user"));
        assertThat(persisted).isNotNull();
        assertThat(persisted.getId()).isNotNull();

        int rows = mapper.deleteById(persisted.getId());

        assertThat(rows).isEqualTo(1);
        assertThat(mapper.selectById(persisted.getId())).isNull();
    }
}
```

# Service测试

```java
package com.heima.admin.service.impl;

import com.baomidou.mybatisplus.core.conditions.Wrapper;
import com.heima.admin.mapper.AdUserMapper;
import com.heima.model.admin.dtos.AdUserDto;
import com.heima.model.admin.pojos.AdUser;
import com.heima.model.common.dtos.ResponseResult;
import com.heima.model.common.enums.AppHttpCodeEnum;
import com.heima.utils.common.AppJwtUtil;
import io.jsonwebtoken.Claims;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.test.util.ReflectionTestUtils;
import org.springframework.util.DigestUtils;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

/**
 * 管理端登录服务实现单元测试（纯 Mockito，不启动 Spring 容器）。
 * 被测类 AdUserLoginServiceImpl extends ServiceImpl&lt;AdUserMapper, AdUser&gt;，
 * 其父类 ServiceImpl 通过 @Autowired 注入字段 baseMapper（声明在父类，字段名就叫 baseMapper），
 * login 内部 getOne(...) 实际走 baseMapper.selectOne(...)，
 * 因此手动 new 被测对象后，用 ReflectionTestUtils 把 AdUserMapper mock 注入 baseMapper 字段。
 */
@DisplayName("AdUserLoginServiceImpl 管理端登录服务")
@ExtendWith(MockitoExtension.class)
class AdUserLoginServiceImplTest {

    @Mock
    private AdUserMapper adUserMapper;

    private AdUserLoginServiceImpl adUserLoginService;

    @BeforeEach
    void setUp() {
        adUserLoginService = new AdUserLoginServiceImpl();
        ReflectionTestUtils.setField(adUserLoginService, "baseMapper", adUserMapper);
    }

    /**
     * 场景：登录成功——用户存在且密码(明文+salt 的 MD5)匹配，
     * 应返回 code=200，data 中含 user(脱敏：password/salt 被清空)与可解析的 token
     */
    @Test
    @DisplayName("login 用户存在且密码正确：返回 200，data 含脱敏 user 与 token")
    void login_userExistsAndPasswordMatch_shouldReturnOkWithUserAndToken() {
        // 与 SQL 种子 admin 记录口径一致：salt = 123abc，原始密码 123456
        String salt = "123abc";
        String rawPassword = "123456";
        AdUser adUser = new AdUser();
        adUser.setId(2);
        adUser.setName("admin");
        adUser.setSalt(salt);
        // 密文 = MD5(明文 + salt)，与被测实现里 DigestUtils.md5DigestAsHex((password + salt).getBytes()) 计算方式一致
        adUser.setPassword(DigestUtils.md5DigestAsHex((rawPassword + salt).getBytes()));
        adUser.setNickname("ad");
        adUser.setPhone("13320325528");

        // 数据库中存在该用户
        when(adUserMapper.selectOne(any(Wrapper.class))).thenReturn(adUser);

        AdUserDto dto = new AdUserDto();
        dto.setName("admin");
        dto.setPassword(rawPassword);

        ResponseResult result = adUserLoginService.login(dto);

        // 成功响应
        assertThat(result.getCode()).isEqualTo(AppHttpCodeEnum.SUCCESS.getCode());
        // data 为 {user, token} 的 map
        assertThat(result.getData()).isInstanceOf(Map.class);
        Map<String, Object> data = (Map<String, Object>) result.getData();
        assertThat(data).containsKeys("user", "token");
        // 返回的 user 就是查询出的同一实例，且敏感字段 password/salt 已被清空
        assertThat(data.get("user")).isSameAs(adUser);
        assertThat(adUser.getPassword()).isEmpty();
        assertThat(adUser.getSalt()).isEmpty();
        assertThat(adUser.getName()).isEqualTo("admin");
        // token 非空且用真实 AppJwtUtil 可解析，id 载荷为 userId
        String token = (String) data.get("token");
        assertThat(token).isNotBlank();
        Claims claims = AppJwtUtil.getClaimsBody(token);
        assertThat(claims).isNotNull();
        assertThat(((Number) claims.get("id")).longValue()).isEqualTo(adUser.getId().longValue());
        // 确认只查询了一次用户
        verify(adUserMapper).selectOne(any(Wrapper.class));
    }

    /**
     * 场景：用户名在库中不存在，应返回 DATA_NOT_EXIST，不进入密码校验与签发流程
     */
    @Test
    @DisplayName("login 用户不存在：返回 DATA_NOT_EXIST")
    void login_userNotExist_shouldReturnDataNotExist() {
        // 数据库查无此人
        when(adUserMapper.selectOne(any(Wrapper.class))).thenReturn(null);

        AdUserDto dto = new AdUserDto();
        dto.setName("nobody");
        dto.setPassword("whatever");

        ResponseResult result = adUserLoginService.login(dto);

        assertThat(result.getCode()).isEqualTo(AppHttpCodeEnum.DATA_NOT_EXIST.getCode());
        assertThat(result.getErrorMessage()).isEqualTo(AppHttpCodeEnum.DATA_NOT_EXIST.getErrorMessage());
        assertThat(result.getData()).isNull();
        verify(adUserMapper).selectOne(any(Wrapper.class));
    }

    /**
     * 场景：用户存在但密码不匹配（MD5 不一致），应返回 LOGIN_PASSWORD_ERROR 且不签发 token
     */
    @Test
    @DisplayName("login 密码错误：返回 LOGIN_PASSWORD_ERROR")
    void login_passwordMismatch_shouldReturnLoginPasswordError() {
        String salt = "123abc";
        AdUser adUser = new AdUser();
        adUser.setId(2);
        adUser.setName("admin");
        adUser.setSalt(salt);
        // 库中存的是 123456 的加盐密文
        adUser.setPassword(DigestUtils.md5DigestAsHex(("123456" + salt).getBytes()));

        when(adUserMapper.selectOne(any(Wrapper.class))).thenReturn(adUser);

        AdUserDto dto = new AdUserDto();
        dto.setName("admin");
        dto.setPassword("wrong-password");

        ResponseResult result = adUserLoginService.login(dto);

        assertThat(result.getCode()).isEqualTo(AppHttpCodeEnum.LOGIN_PASSWORD_ERROR.getCode());
        assertThat(result.getErrorMessage()).isEqualTo(AppHttpCodeEnum.LOGIN_PASSWORD_ERROR.getErrorMessage());
        assertThat(result.getData()).isNull();
        verify(adUserMapper).selectOne(any(Wrapper.class));
    }
}
```