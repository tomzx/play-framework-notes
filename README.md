# Play framework

Tips and tricks I've learned while developing a play application targeting `WebLogic 10.3`, `Java 6` and `Oracle Database`.

1. [Disabling logging completely](#disabling-logging)
2. [Writing a custom Ebean plugin to use J2EE data source](#custom-ebean-plugin)
3. [Environment specific configuration file](#environment-specific-configuration)
4. [Custom handler for bad requests, on errors (500) and not found (404)](#custom-handlers)
5. [Helper methods](#helper-methods)

## <a name="disabling-logging"></a>Disabling logging completely

Create `conf/application-logger.xml` with the following

	<configuration>
	    <root level="OFF"></root>
	</configuration>

## <a name="custom-ebean-plugin"></a>Writing a custom Ebean plugin to use J2EE data source

If you develop for `WebLogic 10.3`, chances are you'll want to use the data source it can provide you so that you do not need to hard code credentials in the code.

To do so, you'll want to create a play plugin that will replace the existing Ebean plugin, providing you the same functionalities but using the datasource instead of play's `BoneCP`.

Create
 a file called `play.plugins` inside of your `conf/` folder.

```
10000:com.domain.db.CustomEbeanPlugin
```

Inside app/com/domain/db/CustomEbeanPlugin.java


	package com.domain.db;
	
	import play.Application;
	import play.Configuration;
	import play.Logger;
	import play.Plugin;
	
	import play.*;
	import play.db.*;
	
	import play.api.libs.Files;
	
	import java.io.*;
	import java.util.*;
	
	import com.avaje.ebean.*;
	import com.avaje.ebean.config.*;
	import com.avaje.ebeaninternal.server.ddl.*;
	import com.avaje.ebeaninternal.api.*;
	
	import javax.naming.InitialContext;
	import javax.naming.NamingException;
	import javax.sql.DataSource;
	
	public class CustomEbeanPlugin extends Plugin {
		private final Application application;
	
		private final Map<String,EbeanServer> servers = new HashMap<String,EbeanServer>();
	
		public CustomEbeanPlugin(Application application)
		{
			this.application = application;
		}
	
		@Override
		public void onStart()
		{
			Logger.info("CustomEbeanPlugin has started");
	
			Configuration ebeanConf = Configuration.root().getConfig("customEbean");
	
			if(ebeanConf != null) {
				for(String key: ebeanConf.keys()) {
	
					ServerConfig config = new ServerConfig();
					config.setName(key);
					config.loadFromProperties();
					try {
						try {
							InitialContext initialContext = Helper.getInitialContext();
							DataSource dataSource = (DataSource) initialContext.lookup("jdbc/datasource-name");
							config.setDataSource(new WrappingDatasource(dataSource));
							Logger.info("Datasource set from JNDI");
						} catch (NamingException exception) {
							Logger.error("Could not set datasource from JNDI", exception);
							throw exception;
						}
					} catch(Exception e) {
						throw ebeanConf.reportError(
								key,
								e.getMessage(),
								e
						);
					}
					if(key.equals("default")) {
						config.setDefaultServer(true);
					}
	
					String[] toLoad = ebeanConf.getString(key).split(",");
					Set<String> classes = new HashSet<String>();
					for(String load: toLoad) {
						load = load.trim();
						if(load.endsWith(".*")) {
							classes.addAll(play.libs.Classpath.getTypes(application, load.substring(0, load.length()-2)));
						} else {
							classes.add(load);
						}
					}
					for(String clazz: classes) {
						try {
							config.addClass(Class.forName(clazz, true, application.classloader()));
						} catch(Throwable e) {
							throw ebeanConf.reportError(
									key,
									"Cannot register class [" + clazz + "] in Ebean server",
									e
							);
						}
					}
	
					servers.put(key, EbeanServerFactory.create(config));
				}
			}
		}
	
		@Override
		public void onStop()
		{
			if (!Helper.isProd()) {
				return;
			}
	
			// you may want to tidy up resources here
			Logger.info("CustomEbeanPlugin has stopped");
		}
	
		/**
		 * <code>DataSource</code> wrapper to ensure that every retrieved connection has auto-commit disabled.
		 */
		static class WrappingDatasource implements javax.sql.DataSource {
	
			public java.sql.Connection wrap(java.sql.Connection connection) throws java.sql.SQLException {
				connection.setAutoCommit(false);
				return connection;
			}
	
			// --
	
			final javax.sql.DataSource wrapped;
	
			public WrappingDatasource(javax.sql.DataSource wrapped) {
				this.wrapped = wrapped;
			}
	
			public java.sql.Connection getConnection() throws java.sql.SQLException {
				return wrap(wrapped.getConnection());
			}
	
			public java.sql.Connection getConnection(String username, String password) throws java.sql.SQLException {
				return wrap(wrapped.getConnection(username, password));
			}
	
			public int getLoginTimeout() throws java.sql.SQLException {
				return wrapped.getLoginTimeout();
			}
	
			public java.io.PrintWriter getLogWriter() throws java.sql.SQLException {
				return wrapped.getLogWriter();
			}
	
			public void setLoginTimeout(int seconds) throws java.sql.SQLException {
				wrapped.setLoginTimeout(seconds);
			}
	
			public void setLogWriter(java.io.PrintWriter out) throws java.sql.SQLException {
				wrapped.setLogWriter(out);
			}
	
			public boolean isWrapperFor(Class<?> iface) throws java.sql.SQLException {
				return wrapped.isWrapperFor(iface);
			}
	
			public <T> T unwrap(Class<T> iface) throws java.sql.SQLException {
				return wrapped.unwrap(iface);
			}
	
			public java.util.logging.Logger getParentLogger() {
				return null;
			}
	
		}
	}

Next, you want to follow the [Ebean hack](#ebean-hack) section.

## <a name="environment-specific-configuration"></a>Environment specific configuration file

This will allow you to have separate files for separate environment. The environment available are `dev`, `test` and `production`. 

- `conf/application.conf` (default values)
- `conf/application.dev.conf`
- `conf/application.test.conf`
- `conf/application.prod.conf`

In `app/Global.scala`, add the following
	
	import com.typesafe.config.ConfigFactory
	import java.io.File
	import play.core.j._
	import play.api._
	import play.api.mvc._
	import play.api.mvc.Results._
	import play.i18n.Messages
	
	import helpers.Helper
	
	object Global extends GlobalSettings
	{
		override def onLoadConfig(config: Configuration, path: File, classloader: ClassLoader, mode: Mode.Mode): Configuration = {
			val modeSpecificConfig = config ++ Configuration(ConfigFactory.load(s"application.${mode.toString.toLowerCase}.conf"))
			super.onLoadConfig(modeSpecificConfig, path, classloader, mode)
		}
	}

## <a name="custom-handlers"></a>Custom handler for bad requests, on errors (500) and not found (404)


In `app/Global.scala`, add the following

	import com.typesafe.config.ConfigFactory
	import java.io.File
	import play.core.j._
	import play.api._
	import play.api.mvc._
	import play.api.mvc.Results._
	import play.i18n.Messages
	
	import helpers.Helper
	
	object Global extends GlobalSettings
	{
		// called when a route is found, but it was not possible to bind the request parameters
		override def onBadRequest(request: RequestHeader, error: String) = {
			BadRequest("Bad Request: " + error)
		}
	
		// 500 - internal server error
		override def onError(request: RequestHeader, throwable: Throwable): Result = {
			val jContext = play.core.j.JavaHelpers.createJavaContext(request)
			play.mvc.Http.Context.current.set(jContext);
			InternalServerError(views.html.errors.errors500())
		}
	
		// 404 - page not found error
		override def onHandlerNotFound(request: RequestHeader): Result = {
			val jContext = play.core.j.JavaHelpers.createJavaContext(request)
			play.mvc.Http.Context.current.set(jContext);
			NotFound(views.html.errors.errors404())
		}
	}

## <a name="helper-methods"></a>Helper methods

The following is a list of methods you may find useful at some point during development. Most are small utilities functions that will provide you with some information about the framework.

	public static boolean isDev() {
		return play.api.Play.isDev(play.api.Play.current());
	}

	public static boolean isTest() {
		return play.api.Play.isTest(play.api.Play.current());
	}

	public static boolean isProd() {
		return play.api.Play.isProd(play.api.Play.current());
	}

## Publishing under Weblogic 10.3

### Requirements

[Play 2 war plugin](https://github.com/play2war/play2-war-plugin)

### Instructions

To prepare a war file for deployment onto a `Weblogic` instance, you will want to use play2war. The plugin will provide you with the functionality necessary to generate a .war file. 

If you plan to use Ebean for your model/ORM needs, read up on the [Ebean hack](#ebean-hack), which you will most likely need to do.

## <a name="ebean-hack"></a>Ebean hack

In production mode, Ebean will have difficulties finding the model files due to how reflection works and the fact that your model are packaged in a .war file instead of being directly on the disk.

In a generateWar.sh file, have the following
	
	#!/bin/bash
	
	PLAY=`which play`
	if [ "$PLAY" == "" ]; then
		PLAY=/home/jenkins/libraries/play/play
	fi
	
	echo PLAY=$PLAY
	
	$PLAY clean
	
	# Strip out
	FIND=db.default.driver=org.h2.Driver
	sed -i "s/$FIND//g" conf/application.conf
	# Strip out
	FIND=db.default.url=\"jdbc:h2:mem:play\"
	sed -i "s/$FIND//g" conf/application.conf
	
	# with ebean.models = "models.*"
	$PLAY compile
	
	# Replace models.* with models.User
	FIND=ebean.default=\"models.\\*\"
	REPLACE=customEbean.default=\"models.User\"
	
	sed -i "s/$FIND/$REPLACE/g" conf/application.conf
	
	# with customEbean.models = "models.User"
	$PLAY compile
	
	$PLAY war

This script will make sure to remove the default org.h2.Driver/jdbc:h2:mem:play db details. Furthermore, it will also replace `ebean.default="models.*"` with `customEbean.default="models.User"`. Note that this change will work with the change mentioned in the [Writing a custom Ebean plugin to use J2EE data source](#custom-ebean-plugin) section. If you do not want to use that change, but you do face the problem where it cannot find models, you can replace `customEbean.default` with `ebean.default`.