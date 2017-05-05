#问题的引出-- 

	为什么我们通过DriverManager.getConnection(url, username, password)获取到的是Connection是一个接口,却能调用方法?不是说接口只是一个抽象的东西?不能做具体的事情么?
	那我们通过DriverManager获取到的Connection接口为什么可以做获取Statement对象之类的事情?


首先我们看我们获取到的对象是怎么样获取的:

	Connection	conn = DriverManager.getConnection(url, username, password);

这种获取方法是不是很眼熟?我们看一下从小学到大的一个创建对象的写法:
	
	Person person=new Student();

这样来看是不是很熟悉?是不是我们的父类引用指向子类对象?

###所以我们实际上通过DriverManager.getConnection(url, username, password)获取到的是实现了Connection接口的实体对象

我们接着看DriverManager是怎么获取到Connection实体对象的,先通过DriverManager.getConnection(url, username, password)跳到源码位置


DvierManamger的方法


	 	public static Connection getConnection(String url,
	        String user, String password) throws SQLException {
	        java.util.Properties info = new java.util.Properties();//创建了一个Properties对象来存放信息
	
	        if (user != null) {
	            info.put("user", user);//如果user不为空,将user信息存入info对象
	        }
	        if (password != null) {
	            info.put("password", password);//如果password不为空,将password信息存入info对象
	        }
	
			//再通过另一个重载的getConnection方法获取连接
	        return (getConnection(url, info, Reflection.getCallerClass()));
	    }

我们在第一次跳转进来的方法中是没法直接获取实现了Connection接口的实体类的,这里还要通过DriverManger的重载方法继续做做事情,接着我们进到getConnection(url, info, Reflection.getCallerClass())方法


		private static Connection getConnection(
		        String url, java.util.Properties info, Class<?> caller) throws SQLException {
		        /*
		         * When callerCl is null, we should check the application's
		         * (which is invoking this class indirectly)
		         * classloader, so that the JDBC driver class outside rt.jar
		         * can be loaded from here.
		         */
		        ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
		        synchronized(DriverManager.class) {
		            // synchronize loading of the correct classloader.
		            if (callerCL == null) {
		                callerCL = Thread.currentThread().getContextClassLoader();
		            }
		        }
		
		        if(url == null) {
		            throw new SQLException("The url cannot be null", "08001");
		        }
		
		        println("DriverManager.getConnection(\"" + url + "\")");
		
		        // Walk through the loaded registeredDrivers attempting to make a connection.
		        // Remember the first exception that gets raised so we can reraise it.
		        SQLException reason = null;
		
		        for(DriverInfo aDriver : registeredDrivers) {
		            // If the caller does not have permission to load the driver then
		            // skip it.
		            if(isDriverAllowed(aDriver.driver, callerCL)) {
		                try {
		                    println("    trying " + aDriver.driver.getClass().getName());
		                    Connection con = aDriver.driver.connect(url, info);
		                    if (con != null) {
		                        // Success!
		                        println("getConnection returning " + aDriver.driver.getClass().getName());
		                        return (con);
		                    }
		                } catch (SQLException ex) {
		                    if (reason == null) {
		                        reason = ex;
		                    }
		                }
		
		            } else {
		                println("    skipping: " + aDriver.getClass().getName());
		            }
		
		        }
		
		        // if we got here nobody could connect.
		        if (reason != null)    {
		            println("getConnection failed: " + reason);
		            throw reason;
		        }
		
		        println("getConnection: no suitable driver found for "+ url);
		        throw new SQLException("No suitable driver found for "+ url, "08001");
		    }


这一整个方法我们不用全部关心,找到切入点,我们现在知道自己在找Connection,和Connection息息相关的是Driver,那我们现在就针对这两个对象来查找,可以看到这个方法里的这段关键代码

	 for(DriverInfo aDriver : registeredDrivers) {
	            // If the caller does not have permission to load the driver then
	            // skip it.
	            if(isDriverAllowed(aDriver.driver, callerCL)) {
	                try {
	                    println("    trying " + aDriver.driver.getClass().getName());
	                    Connection con = aDriver.driver.connect(url, info);
	                    if (con != null) {
	                        // Success!
	                        println("getConnection returning " + aDriver.driver.getClass().getName());
	                        return (con);
	                    }
	                } catch (SQLException ex) {
	                    if (reason == null) {
	                        reason = ex;
	                    }
	                }
	
	            } else {
	                println("    skipping: " + aDriver.getClass().getName());
	            }
	
	        }
	
这个增强for循环中有一个registeredDrivers对象,我们通过查看可以知道他是一个ArrayList,存在这个List的对象是DriverInfo,现在先看一下DriverInfo

	final Driver driver;
    DriverAction da;
    DriverInfo(Driver driver, DriverAction action) {
        this.driver = driver;
        da = action;
    }

可以看到他接收了一个成员变量driver,那我们初步可以判断,DriverManager是通过registeredDrivers这个集合来获取已经注册好的驱动,那registeredDrivers这个集合是在什么地方添加对象的呢?

    public static synchronized void registerDriver(java.sql.Driver driver,
            DriverAction da)
        throws SQLException {

        /* Register the driver if it has not already been added to our list */
        if(driver != null) {
            registeredDrivers.addIfAbsent(new DriverInfo(driver, da));
        } else {
            // This is for compatibility with the original DriverManager
            throw new NullPointerException();
        }

        println("registerDriver: " + driver);

    }

根据关键字"registeredDrivers"查找,我们发现了registerDriver这个方法,如果driver对象不为空就尝试进行添加,如果没有添加过,我们就会将该driver对象加入registerDriver集合中,而注册驱动是我们第一步做的事情,所以在我们注册驱动的时候,已经将com.mysql.jdbc.Driver加到了该集合中

再回到增强for循环看他做了什么事

		 Connection con = aDriver.driver.connect(url, info);
            if (con != null) {
                // Success!
                println("getConnection returning " + aDriver.driver.getClass().getName());
                return (con);
            }

DriverManger通过每次遍历到的aDriver获取里面的driver对象,再调用它的connect方法,将url(数据库地址),info(访问用户名,访问密码)拿来使用,获取实现了Connection接口的实现类,到此,我们初步可以看到DriverManager是如何获取Connection实现类的了

去到我们注册的com.mysql.jdbc.Driver类中找connect方法,发现没找到,那子类没有的方法一般父类会有,我们继续往父类com.mysql.jdbc.NonRegisteringDriver那查找connect(url,info)方法

我们在父类com.mysql.jdbc.NonRegisteringDriver中可以找到下面方法:

	public java.sql.Connection connect(String url, Properties info)
			throws SQLException 

在这个方法里我们就能看到它在返回ConnectionImpl对象了