cas-addon
=========

### Install and configure eXo Platform
Go to `$PLATFORM_HOME` then run:
```
./addon.sh install exo-cas
```

#### Platform tomcat
- Add the following to the `$PLATFORM_HOME/gatein/conf/exo.properties` file

```
# SSO
gatein.sso.enabled=true
gatein.sso.callback.enabled=${gatein.sso.enabled}
gatein.sso.login.module.enabled=${gatein.sso.enabled}
gatein.sso.login.module.class=org.gatein.sso.agent.login.SSOLoginModule
gatein.sso.server.url=http://localhost:8888/cas
gatein.sso.portal.url=http://localhost:8080
gatein.sso.filter.logout.class=org.gatein.sso.agent.filter.CASLogoutFilter
gatein.sso.filter.logout.url=${gatein.sso.server.url}/logout
gatein.sso.filter.login.sso.url=${gatein.sso.server.url}/login?service=${gatein.sso.portal.url}/@@portal.container.name@@/initiatessologin
```

- Add `<Valve className="org.gatein.sso.agent.tomcat.ServletAccessValve" />` to `$PLATFORM_HOME/conf/server.xml`

```
....
    <Engine name="Catalina" defaultHost="localhost">
        <Host name="localhost" appBase="webapps" startStopThreads="-1"
              unpackWARs="${EXO_TOMCAT_UNPACK_WARS}" autoDeploy="true">
            <Valve className="org.gatein.sso.agent.tomcat.ServletAccessValve" />
            ... 
            <Valve className="org.apache.catalina.authenticator.SingleSignOn" />
            ...
            <Listener className="org.exoplatform.platform.server.tomcat.PortalContainersCreator" />
            ...
        </Host>
    </Engine>
....
```

#### Platform Jboss
- Add the following to the `$PLATFORM_HOME/standalone/configuration/gatein/exo.properties`

```
# SSO
gatein.sso.enabled=true
gatein.sso.callback.enabled=${gatein.sso.enabled}
gatein.sso.login.module.enabled=${gatein.sso.enabled}
gatein.sso.login.module.class=org.gatein.sso.agent.login.SSOLoginModule
gatein.sso.server.url=http://localhost:8888/cas
gatein.sso.portal.url=http://localhost:8080
gatein.sso.filter.logout.class=org.gatein.sso.agent.filter.CASLogoutFilter
gatein.sso.filter.logout.url=${gatein.sso.server.url}/logout
gatein.sso.filter.login.sso.url=${gatein.sso.server.url}/login?service=${gatein.sso.portal.url}/@@portal.container.name@@/initiatessologin
```

- Uncomment the below login module in `$PLATFORM_HOME/standalone/configuration/standalone-exo.xml`, then change `${gatein.sso.login.module.enabled}` and `${gatein.sso.login.module.class}` into `#{gatein.sso.login.module.enabled}` and `#{gatein.sso.login.module.class}` respectively.

```xml
<login-module code="org.gatein.sso.integration.SSODelegateLoginModule" flag="required">
    <module-option name="enabled" value="#{gatein.sso.login.module.enabled}"/>
    <module-option name="delegateClassName" value="#{gatein.sso.login.module.class}"/>
    <module-option name="portalContainerName" value="portal"/>
    <module-option name="realmName" value="gatein-domain"/>
    <module-option name="password-stacking" value="useFirstPass"/>
</login-module>
```

### Build and configure cas server
- Download CAS from: `https://www.apereo.org/cas/download` and extract it to `$CAS_HOME`
- Download tomcat from: `http://tomcat.apache.org/download-70.cgi` amd extract it to `$CAS_TOMCAT_HOME`
- Go to `$CAS_HOME/cas-server-webapp` and execute:
```
mvn clean install -Dmaven.test.skip=true
```
- Deploy CAS to Tomcat by copying `$CAS_HOME/cas-server-webapp/target/cas.war` into `$CAS_TOMCAT_HOME/webapps`
- Change the default port to avoid conflicts with the default eXo Platform (for testing purposes) by replacing the `8080` port with `8888` in `$CAS_TOMCAT_HOME/conf/server.xml`.
- Start cas tomcat

#### Configure authentication plugin for CAS
##### Authentication with callback

In this case, when user authentication, cas server will call an REST api to authenticate username/password
- You will need unzip `cas-plugin.zip` in `$PLATFORM_HOME`, we assume that you extract this file into folder `cas-plugin` 
- If you are using CAS 3.5:
    - Copy `commons-httpclient-3.1.jar`, `sso-common-plugin-1.3.1.Final.jar` and `sso-cas-plugin-1.3.1.Final.jar` from `$PLATFORM_HOME/cas-plugin` into `$CAS_TOMCAT_HOME/webapps/cas/WEB-INF/lib`
    - Open `$CAS_TOMCAT_HOME/webapps/cas/WEB-INF/deployerConfigContext.xml`, then replace:
    
      ```xml
      <bean
          class="org.jasig.cas.authentication.handler.support.SimpleTestUsernamePasswordAuthenticationHandler" />
      ```
      With the following:
      
      ```xml
      <bean class="org.gatein.sso.cas.plugin.AuthenticationPlugin">
          <property name="gateInProtocol"><value>http</value></property>
          <property name="gateInHost"><value>localhost</value></property>
          <property name="gateInPort"><value>8080</value></property>
          <property name="gateInContext"><value>portal</value></property>
          <property name="httpMethod"><value>POST</value></property>
      </bean>
      ```

- If you are using CAS version 4.0
    - Copy `commons-httpclient-3.1.jar`, `sso-common-plugin-1.3.1.Final.jar` and `cas40-plugin-4.2.x-SNAPSHOT.jar` from `$PLATFORM_HOME/cas-plugin` into `$CAS_TOMCAT_HOME/webapps/cas/WEB-INF/lib`
    - Open `$CAS_TOMCAT_HOME/webapps/cas/WEB-INF/deployerConfigContext.xml`, then replace:
    
    ```xml
    <bean id="primaryAuthenticationHandler"
              class="org.jasig.cas.authentication.AcceptUsersAuthenticationHandler">
        <property name="users">
            <map>
                <entry key="casuser" value="Mellon"/>
            </map>
        </property>
    </bean>
    ```
    
    With:
    
    ```xml
    <bean id="primaryAuthenticationHandler" class="org.gatein.sso.cas.plugin.CAS40AuthenticationPlugin">
        <property name="gateInProtocol"><value>http</value></property>
        <property name="gateInHost"><value>localhost</value></property>
        <property name="gateInPort"><value>8080</value></property>
        <property name="gateInContext"><value>portal</value></property>
        <property name="httpMethod"><value>POST</value></property>
    </bean>
    ```

##### Other authentication plugin
please follow do guideline at: https://wiki.jasig.org/display/CASUM/Authentication


