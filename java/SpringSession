
由于项目中使用了Nginx做了负载均衡，同时有多个服务器运行tomcat提供HTTP服务，由于有登录功能，涉及到了会话共享问题。会话共享采用的方案为，使用阿里的高可用redis服务存储会话然后进行共享。会话由SpringSession来创建，抛弃了tomcat创建的会话。

SpringSession的原理为，创建一个过滤器，把该过滤器放在所有可能涉及会话操作的前面。在该过滤器中，对request进行装饰，替换掉原来所有的会话操作方法为目标方法，后续的过滤器对会话进行操作实际上是使用目标方法进行操作。目标方法因存储的方式而不同，可以选择关系型数据库，也可以使用redis、mangoDB等NoSQL数据库。

下面通过阅读SpringSession的源码学习其原理。

### SpringSession引入
SpringSession的引入需要在web.xml中配置一个过滤器代理，被代理对象名为springSessionRepositoryFilter。该对象需要引入配置对象进行创建，Redis对应的是RedisHttpSessionConfiguration：
```
	<bean class="org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration">
		<property name="maxInactiveIntervalInSeconds" value="${maxInactiveIntervalInSeconds}"></property>
	</bean>
```
属性maxInactiveIntervalInSeconds表示会话的有效时长。下面分析RedisHttpSessionConfiguration都干了些什么事。

RedisHttpSessionConfiguration类带有@Configuration注解，说明该类是Spring配置类，为了给IOC容器中生成某些对象，同时继承SpringHttpSessionConfiguration，后者有两个@Bean注解，由于@Bean注解可以被继承，所以RedisHttpSessionConfiguration有以下6个带有@Bean注解的方法：
```
// 方法1
RedisMessageListenerContainer org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration.redisMessageListenerContainer(RedisConnectionFactory connectionFactory, RedisOperationsSessionRepository messageListener);
// 方法2
RedisTemplate<Object, Object> org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration.sessionRedisTemplate(RedisConnectionFactory connectionFactory);
// 方法3
RedisOperationsSessionRepository org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration.sessionRepository(@Qualifier(value="sessionRedisTemplate") RedisOperations<Object, Object> sessionRedisTemplate, ApplicationEventPublisher applicationEventPublisher);
// 方法4
InitializingBean org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration.enableRedisKeyspaceNotificationsInitializer(RedisConnectionFactory connectionFactory);
// 方法5
SessionEventHttpSessionListenerAdapter org.springframework.session.config.annotation.web.http.SpringHttpSessionConfiguration.sessionEventHttpSessionListenerAdapter();
// 方法6
<S extends ExpiringSession> SessionRepositoryFilter<? extends ExpiringSession> org.springframework.session.config.annotation.web.http.SpringHttpSessionConfiguration.springSessionRepositoryFilter(SessionRepository<S> sessionRepository);
```
由于方法1、4、5暂放一边。从方法名中可以看出，方法2生成了一个RedisTemplate对象，该对象被用做与redis服务器进行通信，入参为RedisConnectionFactory对象，该对象可以在xml进行配置生成；方法3生成一个会话仓库RedisOperationsSessionRepository，顾名思义用来管理会话，入参为方法2生成的对象；方法6返回一个SessionRepositoryFilter对象，对象名为springSessionRepositoryFilter，也就是在web.xml配置的被代理对象，入参被自动注入方法3创建的会话仓库。

有这三个方法，可以看出引入SpringSession来需要配置以下对象：
1. 配置对象RedisHttpSessionConfiguration
2. Redis通信工厂对象RedisConnectionFactory

这两个对象都可以在xml进行配置，配置后对生成的过滤器对象进行代理即可。

### 会话创建
了解了Spring Session如何引入使用后，来分析下SpringSession是如何创建会话的。从上面的配置可以看出，SpringSession提供服务入口为名为springSessionRepositoryFilter的SessionRepositoryFilter的对象。SessionRepositoryFilter类继承OncePerRequestFilter类，而OncePerRequestFilter实现了Filter接口，作为一个Filter的功能入口为doFilter方法：
```
	public final void doFilter(ServletRequest request, ServletResponse response,
			FilterChain filterChain) throws ServletException, IOException {

        // 只支持HTTP协议的请求
		if (!(request instanceof HttpServletRequest)
				|| !(response instanceof HttpServletResponse)) {
			throw new ServletException(
					"OncePerRequestFilter just supports HTTP requests");
		}
		HttpServletRequest httpRequest = (HttpServletRequest) request;
		HttpServletResponse httpResponse = (HttpServletResponse) response;
		
		// 是否已经被该过滤器处理过
		boolean hasAlreadyFilteredAttribute = request
				.getAttribute(this.alreadyFilteredAttributeName) != null;

            
		if (hasAlreadyFilteredAttribute) {
            // 已经处理过那就不再处理
			// Proceed without invoking this filter...
			filterChain.doFilter(request, response);
		}
		else {
			// Do invoke this filter...
			// 设置处理标志
			request.setAttribute(this.alreadyFilteredAttributeName, Boolean.TRUE);
			try {
			    // 处理请求，子类实现
				doFilterInternal(httpRequest, httpResponse, filterChain);
			}
			finally {
				// Remove the "already filtered" request attribute for this request.
				request.removeAttribute(this.alreadyFilteredAttributeName);
			}
		}
	}
```
下面分析SessionRepositoryFilter的doFilterInternal方法：
```
        // 设置request的会话存放的仓库
		request.setAttribute(SESSION_REPOSITORY_ATTR, this.sessionRepository);

        // 包装request类，经过包装后request和response的HTTPSession已经被偷梁换柱为Redis中的session啦，可以看下该request中的getSession方法
		SessionRepositoryRequestWrapper wrappedRequest = new SessionRepositoryRequestWrapper(
				request, response, this.servletContext);
		SessionRepositoryResponseWrapper wrappedResponse = new SessionRepositoryResponseWrapper(
				wrappedRequest, response);
        
        // 使用策略类包装request和response，作用是什么暂时不清楚
		HttpServletRequest strategyRequest = this.httpSessionStrategy
				.wrapRequest(wrappedRequest, wrappedResponse);
		HttpServletResponse strategyResponse = this.httpSessionStrategy
				.wrapResponse(wrappedRequest, wrappedResponse);

		try {
			filterChain.doFilter(strategyRequest, strategyResponse);
		}
		finally {
		    // 提交新session到redis中
			wrappedRequest.commitSession();
		}
```
使用SessionRepositoryRequestWrapper对象对原来的request方法进行了装饰，目的很明显——由SessionRepositoryRequestWrapper创建会话！
下面接着看SessionRepositoryRequestWrapper创建会话的核心方法：
```
		@Override
		public HttpSessionWrapper getSession(boolean create) {
		    // 获取request持有有效会话，如果有了就不再创建新的
			HttpSessionWrapper currentSession = getCurrentSession();
			if (currentSession != null) {
				return currentSession;
			}
			// 从request中获取sessionID
			String requestedSessionId = getRequestedSessionId();
			if (requestedSessionId != null
					&& getAttribute(INVALID_SESSION_ID_ATTR) == null) {
				// 会话尚未被标记无效，那么从会话仓库中获取会话
				S session = getSession(requestedSessionId);
				if (session != null) {
					this.requestedSessionIdValid = true;
					currentSession = new HttpSessionWrapper(session, getServletContext());
					currentSession.setNew(false);
					setCurrentSession(currentSession);
					return currentSession;
				}
				else {
				    // 无效会话
					// This is an invalid session id. No need to ask again if
					// request.getSession is invoked for the duration of this request
					if (SESSION_LOGGER.isDebugEnabled()) {
						SESSION_LOGGER.debug(
								"No session found by id: Caching result for getSession(false) for this HttpServletRequest.");
					}
					setAttribute(INVALID_SESSION_ID_ATTR, "true");
				}
			}
			// request请求头中没有sessionID，或者session被标记为无效，需要创建新的session
			if (!create) {
				return null;
			}
			
			// 为了便于追踪会话的创建，此处抛出一个异常，这个方法值得推荐，便于追踪一些重要对象的创建过程
			if (SESSION_LOGGER.isDebugEnabled()) {
				SESSION_LOGGER.debug(
						"A new session was created. To help you troubleshoot where the session was created we provided a StackTrace (this is not an error). You can prevent this from appearing by disabling DEBUG logging for "
								+ SESSION_LOGGER_NAME,
						new RuntimeException(
								"For debugging purposes only (not an error)"));
			}
			// 调用仓库创建新会话
			S session = SessionRepositoryFilter.this.sessionRepository.createSession();
			// 设置最后访问时间为当前时间，为了计算session的有效剩余时间
			session.setLastAccessedTime(System.currentTimeMillis());
			// 设置当前request的session
			currentSession = new HttpSessionWrapper(session, getServletContext());
			setCurrentSession(currentSession);
			return currentSession;
		}
```		



