---
title: MyBatis实战（一)
date: 2017-06-26 10:07:15
tags:
    - MyBatis3
---
### 一、MyBatis框架简介

> MyBatis 是支持定制化 SQL、存储过程以及高级映射的优秀的持久层框架。MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。MyBatis 可以对配置和原生Map使用简单的 XML 或注解，将接口和 Java 的 POJOs(Plain Old Java Objects,普通的 Java对象)映射成数据库中的记录。

<!-- more -->

### 二、使用MyBatis框架与原生开发方式对比

1. 数据库连接配置：

    * 使用MyBatis框架：

        ```XML
        <environments default="development">  
            <environment id="development">  
                <transactionManager type="JDBC"/>  
                <dataSource type="POOLED">  
                    <property name="driver" value="com.mysql.jdbc.Driver"/>  
                    <property name="url" value="jdbc:mysql://115.28.61.171:3306/xx"/>  
                    <property name="username" value="root"/>  
                    <property name="password" value="********"/>  
                </dataSource>  
            </environment>  
        </environments>
        ```
    * 使用JDBC：

        ```Java
        public Connection conn = null;   
        public String url="jdbc:mysql://134.78.21.143:3306/xx";  
        public String password="********";  
        public String dbName="root";  
        public String driverName="com.mysql.jdbc.Driver";  
        public Connection getConnection()throws ClassNotFoundException,SQLException   
        {  
           try {  
                   Class.forName(driverName);//指定连接类型  
                   conn = DriverManager.getConnection(url, dbName, password);//获取连接  

            }catch (SQLException e)  
            {  
                e.printStackTrace();  
            }  
           return conn;  
        }  
        ```

2. 查询数据库

    * 使用MyBatis框架：

        ```XML
        <mapper namespace="com.mb.interfaces.IWcUserOperation">  
            <resultMap type="WcUser" id="resultListUser">  
                <id column="id" property="id"/>  
                <result column="openid" property="openid"/>  
                <result column="nickname" property="nickname"/>  
                <result column="province" property="province"/>  
            </resultMap>  
            <select id="selectUserById" parameterType="int"  resultType="WcUser">  
                select * from WXUSER where id= #{id}  
            </select>  
        </mapper>
        ```

        ```Java
        IWcUserOperation userOperation  = session.getMapper(IWcUserOperation.class);  
        WcUser wcUser = userOperation.selectUserById(15);  
        ```
    * 使用JDBC：

        ```Java
        public void getArticle(Connection conn, ArtisvrInitPara yjsvrInitPara, JSONObject jo)  
        {  
            String sql = "select * from "+yjsvrInitPara.getTabname()+" where "+yjsvrInitPara.getExp();    

            try {  
                st= conn.createStatement();  
                rs= st.executeQuery(sql);  
                System.out.println(rs);  

                while(rs.next())  
                {  
                    Article ar= new Article();  

                    ar.setId(       rs.getInt("id"));  
                    ar.setClasses(  rs.getString("classes"));  
                    ar.setContent(  rs.getString("content"));  
                    ar.setClirate(  rs.getInt("clirate"));  
                    ar.setTitle(    rs.getString("title"));  
                    ar.setFbtime(   rs.getString("fbtime"));  

                    list.add(ar);  
                }  
            } catch (SQLException e)   
            {  
                e.printStackTrace();  
            }  
            JSONArray ja=JSONArray.fromObject(list);  
            jo.put("ret", ja);  
            ms.close(rs, st, conn);  
        }  
        ```
### 三、总结

> 使用Mybatis框架可以直接将数据表中每个字段映射到实体类的属性，简化了使用JDBC带来的复杂度。


### 附录：（完整Demo）

（暂时没有时间整理Demo，后期提供）

----
持续更新中。。。。。。
