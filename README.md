# ELK-Stack-With-Mule deployment in local environment
Exporting application logs from Mulesoft application to ELK stack

Exporting application logs from Anypoint Platform to Third Party systems can be in any of the three approaches.
1) Telemetry Exporter
2) CloudHub API
3) log4j Appenders

Floow along the youtube video provided in the references. Few of the changes are needed to the files mentioned in the video, which is being documented here.

Installation of Elasticsearch
-----------------------------
1. Make the following changes in the D:\elasticsearch-9.2.2-windows-x86_64\elasticsearch-9.2.2\config\elasticsearch.yml
   - Uncomment following lines:
         
      path.data: /path/to/data
     
      path.logs: /path/to/logs
     
      http.port: 9200
     
     The elasticsearch will be running on port: https://localhost:9200.

2. Go to bin folder in cmd and execute this command:
   D:\elasticsearch-9.2.2-windows-x86_64\elasticsearch-9.2.2\bin>elasticsearch.bat
   From the console, note the password generated for the user 'elastic'.
   Open https://localhost:9200 in the browser and sign in with user:elastic and password:generated password. This password can be reset using another command.

Installation of Kibana
----------------------
1. Make the following changes in the D:\kibana-9.2.2-windows-x86_64\kibana-9.2.2\config\kibana.yml
   - Uncomment following lines:
         
      server.port: 5601
     
      server.host: "localhost"
     
      elasticsearch.hosts: ["https://localhost:9200"]
     
      elasticsearch.username: "kibana_system"
     
      elasticsearch.password: "PASSWORD_FOR_KIBANA_USER...if not regenerate the password using command"
     
      elasticsearch.requestTimeout: 30000
     
      elasticsearch.ssl.certificateAuthorities: [ "D:/elasticsearch-9.2.2-windows-x86_64/elasticsearch-9.2.2/config/certs/http_ca.crt" ]
     
      elasticsearch.ssl.verificationMode: certificate
     
      pid.file: D:/kibana-9.2.2-windows-x86_64/run/kibana/kibana.pid
     
      ops.interval: 5000

2. Create the below folder structure in the D:\kibana-9.2.2-windows-x86_64.
   kibana-9.2.2-windows-x86_64
   |--->run/kibana
   The kibana.pid file will be stored in this folder.
   
3. Go to bin folder in cmd and execute this command:
   D:\kibana-9.2.2-windows-x86_64\kibana-9.2.2\bin>kibana.bat
   This will take time to run. 
   Open http://localhost:5601/ in the browser and sign in with user:elastic and password:generated for elastic user. 

Installation of Logstash
------------------------
1. Make the following changes in the D:\logstash-9.2.2-windows-x86_64\logstash-9.2.2\config\logstash.yml
   - Uncomment following lines:
     
      path.data: D:/Log Export/logstash_logs_path

2. Make the following changes in the D:\logstash-9.2.2-windows-x86_64\logstash-9.2.2\config\pipelines.yml
   - Uncomment following lines:
          
    - pipeline.id: test
      
      pipeline.workers: 1
      
      pipeline.batch.size: 1
      
      config.string: "input { generator {} } filter { sleep { time => 1 } } output { stdout { codec => dots } }"
      
    # - pipeline.id: another_test
   
    #   queue.type: persisted
   
      path.config: "/tmp/logstash/*.config"

4. Create a new config file and place it in D:\logstash-9.2.2-windows-x86_64\logstash-9.2.2\bin folder with following data.
   input {
    tcp {
        port => 4560
        codec => json
    }
  }
  filter {
      date {
          match => [ "timeMillis", "UNIX_MS"]
      }
  }
  output {
      elasticsearch {
          hosts => ["https://localhost:9200"]
          user => "elastic"
          password => "generated_password_for_elastic_user"
          ssl_enabled => true
          ssl_certificate_authorities => "D:/elasticsearch-9.2.2-windows-x86_64/elasticsearch-9.2.2/config/certs/http_ca.crt"
          index => "mule-logs-%{+YYYY.MM.dd}"  #used for indexing in elastic search, can be unique / same for all applications
      }
  }
   
5. Go to bin folder in cmd and execute this command:
   D:\logstash-9.2.2-windows-x86_64\logstash-9.2.2\bin>logstash.bat -f yourconfigfilename.conf
    

Creation of Mule Application
---------------------------
Create a simple mule application and modify the log4j2.xml

====================================app.impl========================================
<?xml version="1.0" encoding="UTF-8"?>
<mule xmlns:http="http://www.mulesoft.org/schema/mule/http" xmlns="http://www.mulesoft.org/schema/mule/core"
	xmlns:doc="http://www.mulesoft.org/schema/mule/documentation"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.mulesoft.org/schema/mule/core http://www.mulesoft.org/schema/mule/core/current/mule.xsd
http://www.mulesoft.org/schema/mule/http http://www.mulesoft.org/schema/mule/http/current/mule-http.xsd">
	<http:listener-config name="HTTP_Listener_config" doc:name="HTTP Listener config" doc:id="ebd3ab1f-3042-4424-8735-f033c1fe9ad3" >
		<http:listener-connection host="0.0.0.0" port="${http.port}" />
	</http:listener-config>
	<configuration-properties doc:name="Configuration properties" doc:id="9269a8e2-aef4-4ed0-9f0f-fc2be9e19cdc" file="properties.yaml" />
	<flow name="test-projectFlow" doc:id="caa9ebe9-5ef0-4cea-bde4-b34649cf00c9" >
		<http:listener doc:name="Listener" doc:id="8458f136-a78b-42f5-a74e-c13eacd24d65" config-ref="HTTP_Listener_config" path="/api/*"/>
		<logger level="INFO" doc:name="Logger" doc:id="5d0b579b-4040-4918-a9ce-a42372585f2a" message="Started logging application"/>
	</flow>
</mule>

==================================properties.yaml=====================================
http:
  port: "8081"

HTTP Listener Config:
<img width="294" height="293" alt="image" src="https://github.com/user-attachments/assets/577871b8-d5c0-4b07-8498-1922a48955ac" />

Global Element:
<img width="300" height="156" alt="image" src="https://github.com/user-attachments/assets/9f316167-eb11-4a01-a056-6fffc59919d0" />

================================log4j2.xml=============================================
Add these changes to the file

    <Appenders>
        <RollingFile name="file" fileName="${sys:mule.home}${sys:file.separator}logs${sys:file.separator}test-project.log"
                 filePattern="${sys:mule.home}${sys:file.separator}logs${sys:file.separator}test-project-%i.log">
            <PatternLayout pattern="%-5p %d [%t] [processor: %X{processorPath}; event: %X{correlationId}] %c: %m%n"/>
            <SizeBasedTriggeringPolicy size="10 MB"/>
            <DefaultRolloverStrategy max="10"/>
        </RollingFile>
        <!-- Exports logs to Kibana -->
        <Socket name="socket" host="localhost" port="4560" protocol="TCP"> 
       	<JsonLayout compact="true" eventEol="true"/>
       </Socket>
    </Appenders> 

    <Loggers>
        <!-- Http Logger shows wire traffic on DEBUG -->
        <!--AsyncLogger name="org.mule.service.http.impl.service.HttpMessageLogger" level="DEBUG"/-->
        <AsyncLogger name="org.mule.service.http" level="WARN"/>
        <AsyncLogger name="org.mule.extension.http" level="WARN"/>

        <!-- Mule logger -->
        <AsyncLogger name="org.mule.runtime.core.internal.processor.LoggerMessageProcessor" level="INFO"/>

        <AsyncRoot level="INFO">
            <AppenderRef ref="file"/>
            <!-- Exports logs to Kibana -->
           <AppenderRef ref="socket"/> 
        </AsyncRoot>
    </Loggers>


Deploy the mule application locally.
Send a POST request to http://0.0.0.0:8081/api/hello, go to http://localhost:5601/ > Analytics > Discover > 
<img width="1916" height="567" alt="image" src="https://github.com/user-attachments/assets/bef53498-8c96-4f21-880e-aba461bb953c" />


References:
https://www.youtube.com/watch?v=FJL8zRQzNSk : The video is dated 4 years back hence some of the configs might not be the same.
