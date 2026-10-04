# TAK Server XFF Logging Patch Procedure

TAK Server's default HTTP access log records only the address of the upstream device that connects to it. When the server sits behind a load balancer or reverse web proxy, such as a BIG-IP, every HTTP log entry shows only the proxy's address, even though the proxy might be sending the X-Forwarded-For header to TAK. The log format is fixed in compiled Java code and cannot be changed through TAK `CoreConfig.xml` or startup script options. This process modifies the one class that defines the Tomcat log format so that every entry also records any X-Forwarded-For value, allowing every request to be traced to its originating client. This process is useful for organizations that download pre-built RPMs from tak.gov and do not maintain their own TAK build server. This process has been validated against TAK Server [5.5-RELEASE-82](https://github.com/TAK-Product-Center/Server/releases/tag/5.5-RELEASE-82) and should work for subsequent releases, provided that the TAK developers do not change the Tomcat architecture.

## Tak build server or a non-Tak Rhel server tools preparation

1. Install the required packages

   ```sh
   dnf install -y java-21-openjdk-devel unzip binutils
   ```

## Windows client tools preparation

2. Download Recaf, a Java bytecode analyzer and disassembler: https://github.com/Col-E/Recaf

3. Download a Java Development Kit (required for Recaf): https://www.oracle.com/java/technologies/downloads/

## Back up the current unmodified war

4. Copy the default war to a backup version

   ```sh
   cp -a /opt/tak/takserver.war /opt/tak/takserver.war.backup
   ```

## Process and modify the takserver.war and export a single modified class file

5. Copy `takserver.war` to the Windows client machine via `scp`

6. Open Recaf by double-clicking the Recaf jar file

<img width="1350" height="944" alt="Image" src="https://github.com/user-attachments/assets/e0c0e041-b1d6-4bc2-a6cb-346a0475c244" />

7. Drag and drop the unmodified `takserver.war` into the Recaf window and wait until war analysis is complete

<img width="1355" height="944" alt="Image" src="https://github.com/user-attachments/assets/3c44b5da-68ee-4e15-b9bc-b5ac18e8ee20" />

8. In the left workspace window, navigate to **Classes > tak > server > CustomizeEmbeddedTomcatContainer**

9. CustomizeEmbeddedTomcatContainer is the class that instructs Tomcat on how to log HTTP connection information. In the right panel find the line:

   ```java
   accessLogValve.setPattern("request: method=%m uri=\"%U\" response: statuscode=%s bytes=%b duration=%D(ms) client: remoteip=%a user=%{username}s useragent=\"%{User-Agent}i\"");
   ```

10. And modify it to include the incoming X-Forwarded-For header via: `xff=%{X-Forwarded-For}i`

    ```java
    accessLogValve.setPattern("request: method=%m uri=\"%U\" response: statuscode=%s bytes=%b duration=%D(ms) client: remoteip=%a xff=%{X-Forwarded-For}i user=%{username}s useragent=\"%{User-Agent}i\"");
    ```

11. Do not make any other changes. Press Control-S to save the now modified class file.

<img width="1353" height="944" alt="Image" src="https://github.com/user-attachments/assets/451b5a8c-eaa8-4fc8-94ec-a681c6c4d2d9" />

<img width="1353" height="944" alt="Image" src="https://github.com/user-attachments/assets/451b5a8c-eaa8-4fc8-94ec-a681c6c4d2d9" />

12. **Do not export the entire war using Recaf**, this repackages the war in a layout the TAK Server cannot load. Right click on 'CustomizeEmbeddedTomcatContainer' and select "Export class". Recaf will ask you where to save the 'CustomizeEmbeddedTomcatContainer.class' file. Ensure you **do not edit** this file in notepad or other text editor or you will cause bytecode corruption of the class file.

<img width="1356" height="947" alt="Image" src="https://github.com/user-attachments/assets/7ed621ea-00da-40bd-86a4-e3c56d38cd4d" />

13. Copy `takserver.war` and the modified class file to the Tak build server or a non-Tak Rhel server, into the same directory

## Patch the modified class file into the war

14. Run these commands in the directory holding `takserver.war` and the modified class file

    `jar` places a file in the war at the same relative path it has on disk, so the class **must** be moved into a temporary **`WEB-INF/classes/tak/server`** folder which builds the necessary internal pathing in the war during the patch

    ```sh
    mkdir -p WEB-INF/classes/tak/server
    mv CustomizeEmbeddedTomcatContainer.class WEB-INF/classes/tak/server/
    jar uf takserver.war WEB-INF/classes/tak/server/CustomizeEmbeddedTomcatContainer.class
    rm -rf WEB-INF
    ```

15. Verify the patch – The output must be `1` before proceeding further

    ```sh
    unzip -p takserver.war WEB-INF/classes/tak/server/CustomizeEmbeddedTomcatContainer.class | strings | grep -c 'xff='
    ```

## Install the patched war

16. On the TAK server, stop the service

    ```sh
    systemctl stop takserver
    ```

17. From the Tak build server or a non-Tak Rhel server, copy the patched `takserver.war` to the TAK server at `/opt/tak/takserver.war`

    ```sh
    scp takserver.war root@<tak-server>:/opt/tak/takserver.war
    ```

18. On the TAK server, set ownership and start the service

    ```sh
    chown tak:tak /opt/tak/takserver.war
    systemctl start takserver
    ```

## Logging Verification

19. Review the access log

    ```sh
    tail -f /opt/tak/logs/takserver-api-access.log
    ```

    Expected entry:

    ```
    ... client: remoteip=<BIG-IP address> xff=<client address> user=... useragent="..."
    ```

## Rollback

20. Restore the backup and restart TAK Server

    ```sh
    systemctl stop takserver
    cp -a /opt/tak/takserver.war.backup /opt/tak/takserver.war
    systemctl start takserver
    ```
