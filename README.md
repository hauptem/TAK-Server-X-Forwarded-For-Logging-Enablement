# TAK Server 'X-Forwarded-For' Logging Enablement

**This process was created for organizations that use compiled RPMs from tak.gov and do not maintain a local TAK build server.** 

TAK Server's https://github.com/TAK-Product-Center/Server default HTTP access log records only the address of the upstream device that connects to it as the 'remoteip' entity. When the TAK server sits behind any load balancer or reverse web proxy, such as a BIG-IP, every tomcat HTTP log entry shows only the proxy's address, even though the proxy might be sending the X-Forwarded-For header to TAK. The tomcat log format is unfortunately fixed in compiled Java code and cannot be manipulated through TAK's `CoreConfig.xml` or startup script options. This process modifies the single java class that defines the embedded Tomcat log format enabling every entry to also record the X-Forwarded-For value. This permits every request to be traced to its originating client within TAK's syslog. This process does not require a full grable rebuild of takserver.war.

This process has been validated against TAK Server 5.5-RELEASE-82 and should work for subsequent releases, provided that the TAK developers do not change the embedded Tomcat architecture.

## Non-TAK-prod Rhel server tools preparation (used for patching the java class into the war)

1. Install the required packages

   ```sh
   dnf install -y java-21-openjdk-devel unzip binutils
   ```

## Windows client tools preparation (used for modifying and exporting a java class from the war)

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

<img width="1351" height="946" alt="Image" src="https://github.com/user-attachments/assets/4f6e358e-5d32-4daf-bd37-1fd2d2a04f92" />

12. **Do not export the entire war using Recaf**, this repackages the war in a layout the TAK Server cannot load. Right click on 'CustomizeEmbeddedTomcatContainer' and select "Export class". Recaf will ask you where to save the 'CustomizeEmbeddedTomcatContainer.class' file. Ensure you **do not edit** this file in notepad or other text editor or you will cause bytecode corruption of the class file rendering it useless.

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
    
## Disclaimer

- This solution is **NOT** officially endorsed, supported, or maintained by TAK Product Center.
- TAK Product Center retains all rights to their trademarks, including but not limited to "TAK", "Team Awareness Kit", and related marks.
- This is an independent, community-developed solution that utilizes TAK Server but is not affiliated with TAK Product Center.
- For official TAK support and solutions, please contact TAK Product Center directly.

**Technical Disclaimer:**
- This process is provided "AS IS" without warranty of any kind.
- The authors and contributors are not responsible for any damages or issues that may arise from its use.
- Always test thoroughly in non-production environments before deployment.
- Backup your TAK configuration before implementing any changes.
- Review and understand all code before deploying to production systems.

By using this software, you acknowledge that you have read and understood these disclaimers and agree to use this solution at your own risk.
