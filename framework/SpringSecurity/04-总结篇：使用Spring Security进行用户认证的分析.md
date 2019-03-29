撰写该篇时，我也算是初步认识Spring Security进行用户登录认证了。要对比以前我自己通过拦截器去实现这个认证简单许多，这个框架的运用还是挺方便的。  
Spring Security默认认证只有username和password，基于这点我们上篇需要加一个验证码，我们也通过Spring Security去验证，因此重写了Spring Security默认的验证类。我们该篇就负责总结下前几篇涉及的用户认证的内容。  
### 1.重要的类
继承自WebSecurityConfigurerAdapter 的类是我们使用Spring Security的核心配置类。
该类一般有如下三个注解

```
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
```
作用分别是标记该类是Spring管理的配置类，开启 Security 服务，开启全局 Securtiy 注解。  
其中开启的全局注解包括以下四个。

```
@PreAuthorize //在方法调用之前,基于表达式的计算结果来限制对方法的访问
@PostAuthorize //允许方法调用,但是如果表达式计算结果为false,将抛出一个安全性异常
@PostFilter //允许方法调用,但必须按照表达式来过滤方法的结果
@PreFilter //允许方法调用,但必须在进入方法之前过滤输入值
```
如下方法便使用了@PreAuthorize ，意思判断是否有该角色，具体内部怎么实现的，后面源码分析再分析。

```
	@GetMapping("/admin")
	@ResponseBody
	@PreAuthorize("hasRole('ROLE_ADMIN')")
	public String printAdmin() {
		return "如果你看见这句话，说明你有ROLE_ADMIN角色";
	}
```
#### 注意点：如果想用Spring Security默认的用户和密码认证，页面的name必须传递username和password  
其中重要的方法之一。

```
	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.authorizeRequests().antMatchers("/", "/home", "/getVerifyCode").permitAll() // 定义不需要拦截的路径
				.anyRequest().authenticated().and().formLogin().loginPage("/login")// 设值登录页
				.permitAll()
				// 指定authenticationDetailsSource
				.authenticationDetailsSource(authenticationDetailsSource)
				// 登出
				.and().logout().permitAll()
				// 自动登录
				.and().rememberMe();
	}
```
主要的用法我也在注释中有标注。  
接下来是另一个方法。
```
	@Override
	protected void configure(AuthenticationManagerBuilder auth) throws Exception {
		auth.authenticationProvider(customAuthenticationProvider);
		auth.userDetailsService(userDetailService).passwordEncoder(new PasswordEncoder() {
			// 密码加密，由各自系统决定，该处明文
			@Override
			public String encode(CharSequence charSequence) {
				return charSequence.toString();
			}

			@Override
			public boolean matches(CharSequence charSequence, String s) {
				return s.equals(charSequence.toString());
			}
		});
	}
```
这一步主要是配置密码加密以及指定自己验证类。

### 2.指定自己的额外表单内容获取（以验证码为例）
针对用户名和密码，我们按规则办事，Spring Security是可以默认处理的，那显然我们现在要加个验证码就不行了。那我们就得考虑两个地方了，怎么把默认获取表单内容的地方修改，怎么把默认验证的地方修改。这里我们先讲讲怎么取表单的额外内容。  

