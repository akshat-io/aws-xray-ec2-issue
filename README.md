# AWS X-Ray Issue

## Below is the tech-stack of our application:

1.  Programming Language: Java
2.  Framework: Spring (Version: 5.3.29)
3.  Server: Tomcat
4.  Java Servlet Container: Catalina (Tomcat's Servlet Container)

## Our Goal:

1. To enable end-to-end AWS X-Ray traces in our application.
2. To add custom **AWS X-Ray User Annotations** to existing AWS X-Ray traces.

_Note: By end-to-end, we mean from "Incoming HTTP Request" through Java processes/classes and up to "SQL Queries"._

### Goal 1: Enable end-to-end AWS X-Ray traces

For the first goal, here are the steps/changes we performed:

1.  Configured AWS X-Ray daemon on Amazon EC2 as per the following doc.
    _Note: Currently, this is manually started and not with "User data script" which runs the daemon automatically when we launch the instance._
    https://docs.aws.amazon.com/xray/latest/devguide/xray-daemon-ec2.html

2.  Instead of using an instance profile to grant the daemon permission to upload trace data to X-Ray. We have configured the local `~/.aws/credentials` file on EC2 so that AWS X-Ray Daemon can collect the traces and send to AWS Xray Service. We have not added the role. Now, we perform following command to start the AWS X-Ray daemon manually before starting the application each time, and get following logs:

    ```
    [ec2-user@ec2-machine.dev ~]$ xray

    2024-02-06T00:12:42-08:00 [Info] Initializing AWS X-Ray daemon 3.3.10
    2024-02-06T00:12:42-08:00 [Info] Using buffer memory limit of 158 MB
    2024-02-06T00:12:42-08:00 [Info] 2528 segment buffers allocated
    2024-02-06T00:12:42-08:00 [Info] Using region: us-east-2
    2024-02-06T00:12:42-08:00 [Info] HTTP Proxy server using X-Ray Endpoint : https://xray.us-east-2.amazonaws.com
    2024-02-06T00:12:42-08:00 [Info] Starting proxy http server on 127.0.0.1:2000

    ```


3.  For Tomcat, We added a listener `AWSXRayServletContextListener` and registered the listener in the `web.xml`. As shown in this doc, in "Sampling Rules" section. https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-java-configuration.html#xray-sdk-java-configuration-sampling

    Below is our implementation:

    _Note: We are using **CentralizedSamplingStrategy**_. Let us know if it is not correct and if we need to use some other sampling strategy.
    ```
    # File: AWSXRayServletContextListener.java
    --------------------------------------------

    public class AWSXRayServletContextListener implements ServletContextListener {

        @Override
        public void contextInitialized(ServletContextEvent event) {
            final AWSXRayRecorderBuilder builder = AWSXRayRecorderBuilder.standard().withPlugin(new EC2Plugin());
            builder.withSamplingStrategy(new CentralizedSamplingStrategy());
            AWSXRay.setGlobalRecorder(builder.build());
        }

        @Override
        public void contextDestroyed(ServletContextEvent event) { }
    }


    ```

    ```
    # File: WEB-INF/web.xml
    -------------------------
        ...

        // More listeners here...

        <listener>
            <listener-class>com.some.package.AWSXRayServletContextListener</listener-class>
        </listener>
        
        ...



    ```


4.  For auto-instrumenting our application, we are using "AWS X-Ray auto-instrumentation agent for Java". We have followed this guide:
https://docs.aws.amazon.com/xray/latest/devguide/aws-x-ray-auto-instrumentation-agent-for-java.html

    - We have added the JVM arguments for X-Ray Java Agent in our `CATALINA_OPTS` variable, as we are using Tomcat with Catalina servlet container. The JVM argument we used:
        ```
        -javaagent:/xray/disco/disco-java-agent.jar=pluginPath=/xray/disco/disco-plugins -Dcom.amazonaws.xray.configFile=/xray/xray-config.json
        ```
    
    - The path givens are proper and contain the jars as shown in the doc and below:

        ```
        xray
        ├── xray-config.json
        └── disco 
            ├── disco-java-agent.jar 
            └── disco-plugins 
                ├── aws-xray-agent-plugin.jar 
                ├── disco-java-agent-aws-plugin.jar 
                ├── disco-java-agent-sql-plugin.jar 
                └── disco-java-agent-web-plugin.jar
        ```

    - Here is the config file, we currently provide:
        ```
        # File: /xray/xray-config.json
        --------------------------------

        {
            "serviceName": "my_app",
            "tracingEnabled": true,
            "awsServiceHandlerManifest": "/xray/DefaultOperationParameterWhitelist.json",     
            "awsSdkVersion": 1,     
            "collectSqlQueries": true
        }
        ```

**As per our understanding of the provided AWS X-Ray docs, the above steps should have been enough to instrument the application automatically and collect/send all of the traces to appear in AWS X-Ray Console.**

But we are getting below logs, when we start the tomcat server, hence, the application.

#### During server initialization:
```
...
INFO [main] com.amazonaws.xray.agent.runtime.config.XRaySDKConfiguration.init Reading X-Ray Agent config file at: /xray/xray-agent.json
INFO [main] com.amazonaws.xray.agent.runtime.config.XRaySDKConfiguration.init Initializing the X-Ray Agent Recorder
INFO [main] com.amazonaws.xray.AWSXRayRecorderBuilder.lambda$build$1 Collecting trace metadata from com.amazonaws.xray.plugins.EC2Plugin.
WARNING [main] com.amazonaws.xray.plugins.EC2Plugin.populateLogReferences CloudWatch Agent log configuration file not found at /opt/aws/amazon-cloudwatch-agent/etc/log-config.json. Install the CloudWatch Agent on this instance to record log references in X-Ray.
WARNING [main] org.apache.tomcat.util.digester.SetPropertiesRule.begin Match [Server/Service/Engine/Host/Valve] failed to set property [resolveHosts] to [false]
INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server version name:   Apache Tomcat/9.0.76
INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server built:          Jun 5 2023 07:17:04 UTC
INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server version number: 9.0.76.0
...
```

#### After a few seconds:
```
...
SEVERE [C3P0PooledConnectionPoolManager[identityToken->2tu9e4b1ysxgw8858ldk|78e71d05]-HelperThread-#4] com.amazonaws.xray.strategy.LogErrorContextMissingStrategy.contextMissing Suppressing AWS X-Ray context missing exception (SegmentNotFoundException): Failed to begin subsegment named 'qaedge1': segment cannot be found.
SEVERE [C3P0PooledConnectionPoolManager[identityToken->2tu9e4b1ysxgw8858ldk|78e71d05]-HelperThread-#5] com.amazonaws.xray.strategy.LogErrorContextMissingStrategy.contextMissing Suppressing AWS X-Ray context missing exception (SegmentNotFoundException): Failed to begin subsegment named 'qaedge1': segment cannot be found.
SEVERE [C3P0PooledConnectionPoolManager[identityToken->2tu9e4b1ysxgw8858ldk|78e71d05]-HelperThread-#0] com.amazonaws.xray.strategy.LogErrorContextMissingStrategy.contextMissing Suppressing AWS X-Ray context missing exception (SegmentNotFoundException): Failed to begin subsegment named 'qaedge1': segment cannot be found.
SEVERE [C3P0PooledConnectionPoolManager[identityToken->2tu9e4b1ysxgw8858ldk|78e71d05]-HelperThread-#4] com.amazonaws.xray.strategy.LogErrorContextMissingStrategy.contextMissing Suppressing AWS X-Ray context missing exception (SegmentNotFoundException): No segment in progress.
SEVERE [C3P0PooledConnectionPoolManager[identityToken->2tu9e4b1ysxgw8858ldk|78e71d05]-HelperThread-#4] com.amazonaws.xray.strategy.LogErrorContextMissingStrategy.contextMissing Suppressing AWS X-Ray context missing exception (SubsegmentNotFoundException): Failed to end subsegment: subsegment cannot be found.
SEVERE [C3P0PooledConnectionPoolManager[identityToken->2tu9e4b1ysxgw8858ldk|78e71d05]-HelperThread-#0] com.amazonaws.xray.strategy.LogErrorContextMissingStrategy.contextMissing Suppressing AWS X-Ray context missing exception (SegmentNotFoundException): No segment in progress.
SEVERE [C3P0PooledConnectionPoolManager[identityToken->2tu9e4b1ysxgw8858ldk|78e71d05]-HelperThread-#0] com.amazonaws.xray.strategy.LogErrorContextMissingStrategy.contextMissing Suppressing AWS X-Ray context missing exception (SubsegmentNotFoundException): Failed to end subsegment: subsegment cannot be found.
SEVERE [C3P0PooledConnectionPoolManager[identityToken->2tu9e4b1ysxgw8858ldk|78e71d05]-HelperThread-#5] com.amazonaws.xray.strategy.LogErrorContextMissingStrategy.contextMissing Suppressing AWS X-Ray context missing exception (SegmentNotFoundException): No segment in progress.
SEVERE [C3P0PooledConnectionPoolManager[identityToken->2tu9e4b1ysxgw8858ldk|78e71d05]-HelperThread-#5] com.amazonaws.xray.strategy.LogErrorContextMissingStrategy.contextMissing Suppressing AWS X-Ray context missing exception (SubsegmentNotFoundException): Failed to end subsegment: subsegment cannot be found.
...
```

---

### Goal 2: Custom AWS X-Ray User Annotations

To achieve our second goal of adding custom AWS X-Ray user annotations, we performed these additional steps after the "Goal 1" steps.

1. We have added following maven dependencies to add custom X-Ray user annotations.
    

    ```
    # File: pom.xml
    ----------------

	<dependencyManagement>
		<dependencies>
			<!-- AWS Xray Dependency BOM -->
			<dependency>
				<groupId>com.amazonaws</groupId>
				<artifactId>aws-xray-recorder-sdk-bom</artifactId>
				<version>2.11.1</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<dependencies>
        <dependency>
			<groupId>com.amazonaws</groupId>
			<artifactId>aws-xray-recorder-sdk-core</artifactId>
		</dependency>
	</dependencies>

    ```

2. We have created a custom Java filter, to add custom user annotations. Here is the Java filter code for reference:
    ```
    # File: AwsXRayCustomFilter.java
    ----------------------------------

    public class AwsXRayCustomFilter implements Filter {

        private static final Logger LOG = LogManager.getLogger(AwsXRayCustomFilter.class);

        @Override
        public void init(FilterConfig filterConfig) throws ServletException {
        }

        @Override
        public void destroy() {
        }

        @Override
        public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
                throws IOException, ServletException {
            final HttpServletRequest httpRequest = (HttpServletRequest) request;
            final HttpServletResponse httpResponse = (HttpServletResponse) response;

            try {
                final Segment document = AWSXRay.getCurrentSegment();
                if (document != null) {
                    LOG.info("AWSXRay current segment has been acquired!");
                    LOG.info("Putting custom user annotation into XRay");
                    document.putAnnotation("hello", "world");
                } else {
                    LOG.info("Cannot acquire AWSXRay current segment!");
                }
                LOG.info("Continuing to chain.doFilter() call");
                chain.doFilter(httpRequest, httpResponse);
            } catch (Exception e) {
                LOG.error("An error occurred in the filter:", e);
                LOG.info("Continuing to chain.doFilter() call");
                chain.doFilter(httpRequest, httpResponse);
            }
        }
    }
    ```

3. We have changed our `WEB-INF/web.xml` to enable this filter as follows:

    ```
    # File: WEB-INF/web.xml
    -------------------------
        ...

        // More filters here...
        
        <filter>
            <filter-name>AwsXRayCustomFilter</filter-name>
            <filter-class>com.some.package.AwsXRayCustomFilter</filter-class>
        </filter>
        <filter-mapping>
            <filter-name>AwsXRayCustomFilter</filter-name>
            <url-pattern>/App/*</url-pattern>
        </filter-mapping>
        ...

    ```

4. This code is not working as well, the `AWSXRay.getCurrentSegment()` method call returns `null` and the code flow is going through the `else` condition and giving output on the log as follows:

    ```
    Cannot acquire AWSXRay current segment!
    ```


## Help

Let us know what is your analysis of above, and what we are doing wrong or what we should correct to make it work.
