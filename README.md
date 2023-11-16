# Apache-Tomcat
Apache와 Tomcat을 이용한 3Tier구조의 서버 구현에 대한 정보


### Apache 수동 설치에 필요한 의존성

- Apache v2.4.46
- apr v1.7.2
- apr-util v1.6.3
- pcre v8.43
- tomcat-connectors-1.2.48-src.tar

---

## Aapache 파일 수정

### workers.properties

worker.list=lb,jkstatus <br/>
worker.lmsJvm1.port={{port}} <br/>
worker.lmsJvm1.host={{ip}} <br/>
worker.lmsJvm1.type=ajp13 <br/>
worker.lmsJvm1.lbfactor=1 <br/>
worker.lmsJvm1.socket_timeout=300 <br/>
worker.lmsJvm1.retries=0 <br/>
worker.lmsJvm1.socket_keepalive=True <br/>
worker.lmsJvm1.connection_pool_timeout=60 <br/>
worker.lmsJvm1.ping_mode=C,P <br/>
worker.lmsJvm1.ping_timeout=100 <br/>
worker.lmsJvm1.connect_timeout=100 <br/>
 
worker.lb.type=lb <br/>
worker.lb.balance_workers=lmsJvm1 <br/>
worker.lb.sticky_session=1 <br/>

worker.jkstatus.type=status <br/>
worker.jkstatus.read_only=True <br/>
worker.jkstatus.bad=s,e,r <br/>

---
### uri.properties

/jkstatus/*=jkstatus <br/>
/*=lb <br/>

---
### http.conf MOD_JK

LoadModule jk_module modules/mod_jk.so <br/>

JkWorkersFile    conf/workers.properties <br/>
JkLogFile "| /usr/local/apache2.4/bin/rotatelogs /usr/local/apache2.4/logs/mod_jk-%Y-%m-%d.log 86400" <br/>
JkLogLevel info <br/> 
JkLogStampFormat "[%a %b %d %H:%M:%S %Y] " <br/>
JkMountFile      conf/uri.properties <br/>

### Aapache tomcat server.xml 구성
<?xml version="1.0" encoding="UTF-8"?>

<Server port="8004" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  
  <GlobalNamingResources>
    
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <Service name="Catalina">
	<Executor name="lmsThreadPool" namePrefix="lms-exec-" maxThreads="500" minSpareThreads="100"/>
    <Connector address="0.0.0.0" port="8009" executor="lmsThreadPool" protocol="AJP/1.3" redirectPort="8443" secretRequired="fale"/>
   
<Engine name="Catalina" defaultHost="localhost" jvmRoute="lmsJvm1">

<Cluster 
        channelSendOptions="8" 
        channelStartOptions="3" 
        className="org.apache.catalina.ha.tcp.SimpleTcpCluster">
    <Manager 
        className="org.apache.catalina.ha.session.DeltaManager" 
        expireSessionsOnShutdown="false" 
        notifyListenersOnReplication="true"
    />
    <Channel className="org.apache.catalina.tribes.group.GroupChannel">
        <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter">
            <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender" />
        </Sender>
        <Receiver 
            address="10.39.71.30" 
            autoBind="0" 
            className="org.apache.catalina.tribes.transport.nio.NioReceiver" 
            maxThreads="6" 
            port="4000" 
            selectorTimeout="5000"
        /> 

        <Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpPingInterceptor" staticOnly="true"/>
        <Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpFailureDetector" />
        <Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatchInterceptor"/>
        <Interceptor className="org.apache.catalina.tribes.group.interceptors.ThroughputInterceptor"/>
        <Interceptor className="org.apache.catalina.tribes.group.interceptors.StaticMembershipInterceptor">
	    <LocalMember className="org.apache.catalina.tribes.membership.StaticMember"
                  uniqueId="{0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,1}}"/>

            <Member 
                className="org.apache.catalina.tribes.membership.StaticMember" 
                port="4001" 
                host="10.39.71.30" 
                uniqueId="{0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,2}" 
            />
        </Interceptor>
        <Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatchInterceptor" />
    </Channel>
    <Valve 
        className="org.apache.catalina.ha.tcp.ReplicationValve" 
        filter=".*\.gif;.*\.js;.*\.jpg;.*\.png;.*\.htm;.*\.html;.*\.css;.*\.txt;" 
    />
    <Valve className="org.apache.catalina.ha.session.JvmRouteBinderValve"/>
    <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener" />
</Cluster>



		
     
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>

	<Host name="localhost"  appBase="/nas/app/"
            unpackWARs="false" autoDeploy="false">
        <Context path="" docBase="." reloadable="false"/>
        <Valve className="org.apache.catalina.valves.RemoteIpValve" />
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%{org.apache.catalina.AccessLog.RemoteAddr}r %l %u %t &quot;%r&quot; %s %b" />

      </Host>
    </Engine>
  </Service>
</Server>

---


