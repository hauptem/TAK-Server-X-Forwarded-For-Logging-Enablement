# TAK Server 'X-Forwarded-For' Logging Enablement

**This process was created for organizations that use compiled RPMs from tak.gov and do not maintain a local TAK build server.** 

TAK Server's https://github.com/TAK-Product-Center/Server default HTTP access log records only the address of the upstream device that connects to it as the 'remoteip' entity. When the TAK server sits behind any load balancer or reverse web proxy, such as a BIG-IP, every tomcat HTTP log entry shows only the proxy's address, even though the proxy might be sending the X-Forwarded-For header to TAK. The tomcat log format is unfortunately fixed in compiled Java code and cannot be manipulated through TAK's `CoreConfig.xml` or startup script options. This process extracts the single java class that defines the embedded Tomcat log format from the war, modifies its log pattern with a short Python script, and places it back into the war, enabling every entry to also record the X-Forwarded-For value. This permits every request to be traced to its originating client within TAK's syslog. This process does not require a full gradle rebuild of takserver.war or any third-party tools, only `jar` and `python3` on a Rhel server.

This process has been validated against TAK Server 5.5-RELEASE-82 and should work for subsequent releases, provided that the TAK developers do not change the embedded Tomcat architecture.

## Non-TAK-prod Rhel server tools preparation (used for patching the war)

1. Install the required packages on the Rhel server or equivalent

   ```sh
   dnf install -y java-21-openjdk-devel python3 unzip binutils
   ```

2. Create a working directory

   ```sh
   mkdir -p /var/tmp/takserver-xff
   ```

## Record and back up the original files

3. On the TAK server, save the md5 hashes of the original files

   ```sh
   cd /opt/tak
   md5sum takserver.war CoreConfig.xml > takserver-original.md5
   cat takserver-original.md5
   ```

4. On the TAK server, back up the original files

   ```sh
   cp -a /opt/tak/takserver.war /opt/tak/takserver.war.backup
   cp -a /opt/tak/CoreConfig.xml /opt/tak/CoreConfig.xml.backup
   ```

5. On the TAK server, verify the backups against the saved hashes – Both lines must report `OK`

   ```sh
   cd /opt/tak
   sed 's/$/.backup/' takserver-original.md5 | md5sum -c -
   ```

## Copy the war to the Rhel server

6. From the TAK server, copy the war and the hash file to the Tak build server or a non-Tak Rhel server

   ```sh
   cd /opt/tak
   scp takserver.war takserver-original.md5 root@<rhel-server>:/var/tmp/takserver-xff/
   ```

7. On the Tak build server or a non-Tak Rhel server, verify the copied war is the unmodified original – The line must report `OK`

   ```sh
   cd /var/tmp/takserver-xff
   grep 'takserver.war$' takserver-original.md5 | md5sum -c -
   ```

## Patch the access log pattern in the war

8. Extract the class file that defines the Tomcat log format from the war

   `jar` recreates the `WEB-INF/classes/tak/server` path on disk, which is the path it needs to place the class back into the war

   ```sh
   cd /var/tmp/takserver-xff
   jar xf takserver.war WEB-INF/classes/tak/server/CustomizeEmbeddedTomcatContainer.class
   ```

9. Add the X-Forwarded-For field to the log pattern in the class file – **Run once only**

   The class file is binary and **must not** be edited in vi or another text editor. The script inserts the field, corrects the stored length of the pattern string, and stops without writing if the stock pattern is not found exactly once

   ```sh
   python3 - <<'EOF'
   import struct
   p = 'WEB-INF/classes/tak/server/CustomizeEmbeddedTomcatContainer.class'
   d = open(p, 'rb').read()
   old = b'remoteip=%a user='
   new = b'remoteip=%a xff=%{X-Forwarded-For}i user='
   assert d.count(old) == 1
   i = d.index(old)
   s = d.rfind(b'request: method=', 0, i)
   assert d[s-3] == 1
   n = struct.unpack('>H', d[s-2:s])[0]
   d = d[:s-2] + struct.pack('>H', n + len(new) - len(old)) + d[s:i] + new + d[i+len(old):]
   open(p, 'wb').write(d)
   EOF
   ```

10. Place the modified class file back into the war and remove the extracted copy

    ```sh
    jar uf takserver.war WEB-INF/classes/tak/server/CustomizeEmbeddedTomcatContainer.class
    rm -rf WEB-INF
    ```

11. Verify the patch – The output must be `1` before proceeding further

    ```sh
    unzip -p takserver.war WEB-INF/classes/tak/server/CustomizeEmbeddedTomcatContainer.class | strings | grep -c 'xff='
    ```

12. Save the md5 hash of the patched war

    ```sh
    cd /var/tmp/takserver-xff
    md5sum takserver.war > takserver-xff.md5
    cat takserver-xff.md5
    ```

## Transfer the patched war to the TAK server

13. On the TAK server, create a staging directory

    ```sh
    mkdir -p /var/tmp/takserver-xff
    ```

14. From the Tak build server or a non-Tak Rhel server, copy the patched war and its hash file to the TAK server

    ```sh
    cd /var/tmp/takserver-xff
    scp takserver.war takserver-xff.md5 root@<tak-server>:/var/tmp/takserver-xff/
    ```

15. On the TAK server, verify the transfer – The line must report `OK`

    ```sh
    cd /var/tmp/takserver-xff
    md5sum -c takserver-xff.md5
    ```

## Install the patched war

16. On the TAK server, stop the service

    ```sh
    systemctl stop takserver
    ```

17. Install the patched war and set ownership

    ```sh
    cp /var/tmp/takserver-xff/takserver.war /opt/tak/takserver.war
    chown tak:tak /opt/tak/takserver.war
    ```

18. Verify the installed war – The line must report `OK`

    ```sh
    cd /opt/tak
    md5sum -c /var/tmp/takserver-xff/takserver-xff.md5
    ```

19. Check whether the HTTP access log is enabled

    ```sh
    grep -n '<logging' /opt/tak/CoreConfig.xml
    ```

20. If the previous step printed nothing, add the logging element

    If it printed a `logging` line without `httpAccessEnabled="true"`, add that attribute to the existing line by hand and skip this command

    ```sh
    sed -i 's#</Configuration>#    <logging httpAccessEnabled="true"/>\n</Configuration>#' /opt/tak/CoreConfig.xml
    ```

21. Start the service

    ```sh
    systemctl start takserver
    ```

## Logging Verification

22. Confirm the API process reports the access log as enabled

    ```sh
    grep 'http access logging' /opt/tak/logs/takserver-api.log | tail -1
    ```

    Expected output:

    ```
    ... http access logging enabled: true
    ```

23. Review the access log

    ```sh
    tail -f /opt/tak/logs/takserver-api-access.log
    ```

    Expected entry:

    ```
    ... client: remoteip=<BIG-IP address> xff=<client address> user=... useragent="..."
    ```

## Rollback

24. On the TAK server, restore the original files and restart the service

    ```sh
    systemctl stop takserver
    cp -a /opt/tak/takserver.war.backup /opt/tak/takserver.war
    cp -a /opt/tak/CoreConfig.xml.backup /opt/tak/CoreConfig.xml
    systemctl start takserver
    ```

25. Verify the restored files against the original hashes – Both lines must report `OK`

    ```sh
    cd /opt/tak
    md5sum -c takserver-original.md5
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
