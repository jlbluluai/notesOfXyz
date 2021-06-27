1.添加jar

    pagehelper-5.1.2.jar ,jsqlparser-1.1.jar

2.添加mybatis-conf.xml

    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE configuration
            PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
            "http://mybatis.org/dtd/mybatis-3-config.dtd">
    <configuration>
    	<plugins>
    		<plugin interceptor="com.github.pagehelper.PageInterceptor"/>
    	</plugins>
    </configuration>

3.修改spring配置文件，关联 mybatis-conf.xml

    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    	<property name="dataSource" ref="dataSource" />
    	<property name="mapperLocations" value="classpath:com/wanho/java98/*/mapping/*.xml" />
    	<property name="configLocation" value="classpath:mybatis/conf/mybatis-conf.xml"/>
    </bean>

4.修改业务层代码

4.1 业务方法返回值类型：PageInfo<News>

    @Override
    	public PageInfo<Pubnews> getPubnews(int module, int begin, int nums) {
    		/*
    		 * Pubnews pubnews = new Pubnews(); pubnews.setModule((byte) module);
    		 * List<Pubnews> list = null; if (begin >= 0) { list =
    		 * pubnewsMapper.selectPagesPubnewsByModule(pubnews, begin, nums); } else { list
    		 * = pubnewsMapper.selectPagesPubnewsByModule(pubnews, 0, nums + begin); } list
    		 * = Utils.reverseList(list); return list;
    		 */
    		Pubnews pubnews = new Pubnews();
    		pubnews.setModule((byte) module);
    		PageHelper.startPage(begin, nums);
    
    		return new PageInfo<Pubnews>(pubnewsMapper.selectPagesPubnewsByModule(pubnews));
    
    	}

5.修改控制器代码

    /**
    	 * 根据模块（公告或是新闻）进行相应的分页查询
    	 * 
    	 * @author 叶灬黎
    	 * @param harvest
    	 * @param response
    	 * @throws Exception
    	 */
    	@RequestMapping("/getPubNews")
    	public void getPubNews(@RequestBody String harvest, HttpServletResponse response) throws Exception {
    		JSONObject js = JSONObject.fromObject(harvest);
    		int module = (int) js.get("module");
    		int currentPage = (int) js.get("currentPage");
    
    		int limitPage = 10;
    		//int totalRecords = pubnewsService.getCounts(module);
    
    		// 计算总页数
    		/*int totalpage;
    		if (totalRecords <= limitPage) {
    			totalpage = 1;
    		} else if ((totalRecords % limitPage) == 0) {
    			totalpage = totalRecords / limitPage;
    		} else {
    			totalpage = (totalRecords / limitPage) + 1;
    		}*/
    
    		//List<Pubnews> list = pubnewsService.getPubnews(module, totalRecords - limitPage * currentPage, limitPage);
    		
    		PageInfo<Pubnews> pageInfo =  pubnewsService.getPubnews(module, currentPage, limitPage);
    		
    		int totalpages = pageInfo.getPages();
    		currentPage = pageInfo.getPageNum();
    
    		JSONObject json = new JSONObject();
    		json.put("currentPage", currentPage);
    		json.put("totalpages", totalpages);
    		json.put("pubnews", pageInfo.getList());
    		response.setContentType("text/html;charset=UTF-8");
    		response.getWriter().write(json.toString());
    	}

6.修改映射文件



    	<!-- 根据模块查询公告新闻总记录数 -->
      <!-- <select id="selectPubnewsCountByModule" resultType="java.lang.Integer">
      		select 
      			count(*)
      		from
      			t_pubnews
      		where
      			module = #{module}
      </select> -->
      
      <!-- 根据模块分页查询公告新闻记录 -->
      <select id="selectPagesPubnewsByModule" resultMap="ResultMapWithBLOBs">
      		select
      			id,title
      		from
      		 	t_pubnews
      		 where
      		 	module = #{pubnews.module}
      		 order by
      		 	create_time desc
      		<!--  limit
      		 	#{begin},#{nums} -->
      </select>



至此，pageHelper在项目中的基本运用就完成了，可以看出其优势，从控制器层开始说，我们只需要获取当前页调用一个业务逻辑即可，而不是像自己写，还得先获取总记录数，自己再算总页数再调用查询记录业务。然后看业务层，我们取消了自己还得判断limit开始的位置防止越界，直接把当前页和每页记录数作为参数给PageInfo即可，调用相关的mapper，返回一个PageInfo对象，这个对象内容就丰富了，当前页的记录，总页数，当前页都有，通过对应的getter方法就能获得。最后看映射文件，由之前的操作我们可以看到，我们不需要自己去获得总记录数了，所以有一条sql就可以注释了，而且我们获取记录的sql也不需要limit了，我们就按照怎么查这些记录就行了，分页pageHelper会解决。
