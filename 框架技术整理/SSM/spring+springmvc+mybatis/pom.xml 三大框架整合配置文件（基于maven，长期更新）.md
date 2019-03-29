#### pom.xml
```
<!-- 初始化框架的版本号 -->
	<properties>
		<jdk.version>1.8</jdk.version>
		<spring.version>4.3.7.RELEASE</spring.version>
		<mybatis.version>3.4.1</mybatis.version>
		<mybatis.spring.version>1.3.0</mybatis.spring.version>
		<pageHelper.version>5.1.1</pageHelper.version>
		<mysql.version>5.1.38</mysql.version>
		<c3p0.version>0.9.2.1</c3p0.version>
		<json.lib.version>2.4</json.lib.version>
		<fastjson.version>1.2.30</fastjson.version>
		<commons.fileupload.version>1.3.1</commons.fileupload.version>
		<log4j.version>1.2.17</log4j.version>
		<slf4j.version>1.7.2</slf4j.version>
		<jstl.version>1.2</jstl.version>
		<servlet.api.version>2.5</servlet.api.version>
		<jsp.api.version>2.0</jsp.api.version>
		<junit.version>4.12</junit.version>
	</properties>

	<dependencies>
		<!-- spring 依赖 start -->

		<!-- spring context  -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-context</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<!-- spring context-support -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-context-support</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<!-- spring oxm -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-oxm</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<!-- spring jdbc -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-jdbc</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<!-- spring tx -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-tx</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<!-- spring web -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-web</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<!-- spring webmvc -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-webmvc</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<!-- spring test -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-test</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<!-- spring 依赖 end -->
		
		<!-- mybatis 依赖 -->
		<dependency>
			<groupId>org.mybatis</groupId>
			<artifactId>mybatis</artifactId>
			<version>${mybatis.version}</version>
		</dependency>

		<!-- mybatis整合spring 依赖 -->
		<dependency>
			<groupId>org.mybatis</groupId>
			<artifactId>mybatis-spring</artifactId>
			<version>${mybatis.spring.version}</version>
		</dependency>
		
		<!-- pageHelper 基于mybatis的分页工具 依赖 -->
		<dependency>
			<groupId>com.github.pagehelper</groupId>
			<artifactId>pagehelper</artifactId>
			<version>${pageHelper.version}</version>
		</dependency>
		
		<!-- mysql 依赖 -->
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>${mysql.version}</version>
			<scope>runtime</scope>
		</dependency>
		
		<!-- c3p0连接池 依赖 -->
		<dependency>
			<groupId>com.mchange</groupId>
			<artifactId>c3p0</artifactId>
			<version>${c3p0.version}</version>
		</dependency>
		
		<!-- json-lib -->
		<dependency>
			<groupId>net.sf.json-lib</groupId>
			<artifactId>json-lib</artifactId>
			<version>${json.lib.version}</version>
			<classifier>jdk15</classifier>
		</dependency>
		
		<!-- fastjson -->
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>fastjson</artifactId>
			<version>${fastjson.version}</version>
		</dependency>
		
		<!-- 文件上传 -->
		<dependency>
			<groupId>commons-fileupload</groupId>
			<artifactId>commons-fileupload</artifactId>
			<version>${commons.fileupload.version}</version>
		</dependency>
		
		<!-- Log4j -->
		<dependency>
			<groupId>log4j</groupId>
			<artifactId>log4j</artifactId>
			<version>${log4j.version}</version>
		</dependency>

		<!-- slf4j -->
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-api</artifactId>
			<version>${slf4j.version}</version>
		</dependency>

		<!-- slf4j -->
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-log4j12</artifactId>
			<version>${slf4j.version}</version>
		</dependency>
		
		<!-- jstl -->
		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>jstl</artifactId>
			<version>${jstl.version}</version>
		</dependency>

		<!-- 加入ServletAPI -->
		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>servlet-api</artifactId>
			<version>${servlet.api.version}</version>
			<scope>provided</scope>
		</dependency>
		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>jsp-api</artifactId>
			<version>${jsp.api.version}</version>
			<scope>provided</scope>
		</dependency>
		
		<!-- junit -->
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>${junit.version}</version>
			<scope>test</scope>
		</dependency>
	</dependencies>


	<build>
		<finalName>tinyDemo_paging</finalName>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>3.3</version>
				<configuration>
					<!-- 指定source和target的版本 -->
					<source>${jdk.version}</source>
					<target>${jdk.version}</target>
				</configuration>
			</plugin>
		</plugins>
	</build>
```
