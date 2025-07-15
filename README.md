
# Oracle SOA Suite Installation & Configuration Guide

## Default Domain & Installation

### Java JDK
- **Version**: 1.8.321

### Configuration
1. Navigate to `Oracle_Home\jdeveloper\ide\bin` folder.
2. Edit `ide.conf` file.
3. Add the following line (under any section, e.g., OSGi options):
   ```
   AddVMOption  -Djdk.lang.Process.allowAmbiguousCommands=true
   ```

### Install Quickstart
Ensure both `soa_quickstart` files are in the same folder. Use the full path of installed Java JDK 1.8.321:

```bash
"C:\Program Files\Java\jdk-1.8\bin\java.exe" -jar fmw_12.2.1.4.0_soa_quickstart.jar
```

### Standalone Example (Oracle Home at `D:\apps\oracle\fmw`):

```bash
set QS_TEMPLATES=D:\apps\oracle\fmw\soa\common\templates\wls\oracle.soa_template.jar,D:\apps\oracle\fmw\osb\common\templates\wls\oracle.osb_template.jar
D:\apps\oracle\fmw\oracle_common\common\bin\qs_config.cmd
```

### Admin Console Access
- URL: `http://localhost:17001/console/login/LoginForm.jsp`
- Username: `weblogic`
- Password: `weblogic123`

---

## Install XDE on Oracle Linux 9

```bash
yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
yum update
yum grouplist
yum groupinstall "xfce"
yum update
yum upgrade
touch /root/.Xauthority
chmod 600 /root/.Xauthority
```

Reference: [Oracle Docs](https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.3/wldpu/creating-and-starting-managed-server-remote-system.html)

---

## Silent Install SOA Domain

Create `oraInst.loc`:
```bash
echo "inventory_loc=/apps/oracle/oraInventory
inst_group=oracle" > ./oraInst.loc
```

Run installation:
```bash
java -jar ./fmw_12.2.1.4.0_infrastructure.jar -silent -responseFile /home/oracle/Downloads/fmw_infra.response -invPtrLoc /home/oracle/Downloads/oraInst.loc
```

---

## Pack/Unpack WebLogic Domain

### Pack
```bash
/apps/oracle/products/fmw12c/oracle_common/common/bin/pack.sh \
-domain=/apps/oracle/domains/SOADEV \
-template=/apps/oracle/SOADEV_MS_template.jar \
-template_name="SOADEV_MS_template" \
-managed=true
```

### Unpack
```bash
/apps/oracle/products/fmw12c/oracle_common/common/bin/unpack.sh \
-domain=/apps/oracle/domains/SOADEV \
-template=/apps/oracle/SOADEV_MS_template.jar \
-user_name=weblogic \
-password=welcome1 \
-app_dir=/apps/oracle/applications/SOADEV \
-java_home=/apps/java/jdk-8
```

References:
- [WLST & Node Manager](https://docs.oracle.com/middleware/12213/wls/NODEM/tutorial.htm#NODEM267)
- [Remote Server Setup](https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.3/wldpu/creating-and-starting-managed-server-remote-system.html#GUID-84608557-0BA5-4238-A350-7A802E4F1A47)

### Node Manager Enrollment
```python
nmEnroll('C:/oracle/user_projects/domains/prod_domain',
         'C:/oracle/user_projects/domains/prod_domain/nodemanager')
```

---

# Oracle SOA Suite Cluster Installation & Configuration Guide

## Architecture Overview

| Component     | Nodes | Cluster Name   |
|---------------|-------|----------------|
| Admin Server  | 1     | -              |
| OSB Server    | 2     | `osb_cluster`  |
| ESS Server    | 2     | `ess_cluster`  |
| BPEL Server   | 2     | `soa_cluster`  |

---

## Step 1: Prerequisites

### Hardware/OS
- OS: Oracle Linux / RHEL 7+
- RAM: ≥16GB per node
- CPU: 4+ cores
- Storage: ≥50GB free/node
- Shared storage (optional but recommended)

### Software
- JDK 1.8+
- Oracle WebLogic 12.2.1.4
- Oracle SOA Suite 12.2.1.4
- Oracle DB 12c+

### Network
- Static IP/DNS for each node
- Open ports: 7001 (Admin), 8001–8100 (Managed), 5556 (Node Manager)

---

## Step 2: Install Software

### Create User & Group
```bash
groupadd oinstall
useradd -g oinstall oracle
passwd oracle
```

### Set Environment (add to `.bash_profile`)
```bash
export ORACLE_BASE=/u01/app/oracle
export JAVA_HOME=/usr/java/jdk1.8.0_291
export PATH=$JAVA_HOME/bin:$PATH
```

### Install WebLogic & SOA Suite
```bash
java -jar fmw_12.2.1.4.0_wls.jar -silent -responseFile /path/to/wls_response.rsp
java -jar fmw_12.2.1.4.0_soa.jar -silent -responseFile /path/to/soa_response.rsp
```

---

## Step 3: Prepare Database (RCU)

Run on Admin Node:
```bash
rcu -silent -createRepository \
  -databaseType ORACLE -connectString db_host:1521/service \
  -dbUser sys -dbRole SYSDBA \
  -schemaPrefix DEV \
  -component SOAINFRA -component MDS -component ORASDPM -component ESS
```

---

## Step 4: Create Domain

### Launch Domain Configuration Wizard
```bash
$ORACLE_HOME/oracle_common/common/bin/config.sh
```

### Templates to Select
- Oracle SOA Suite
- Oracle Enterprise Scheduler (ESS)
- Oracle Service Bus (OSB)

### Server Configuration

| Type  | Server Name  | Port | Node       |
|-------|--------------|------|------------|
| OSB   | osb_server1  | 8001 | osb-node1  |
| OSB   | osb_server2  | 8001 | osb-node2  |
| ESS   | ess_server1  | 8002 | ess-node1  |
| ESS   | ess_server2  | 8002 | ess-node2  |
| BPEL  | bpel_server1 | 8003 | bpel-node1 |
| BPEL  | bpel_server2 | 8003 | bpel-node2 |

- Create Clusters: `osb_cluster`, `ess_cluster`, `soa_cluster`
- Assign respective servers
- Target SOA/ESS/OSB to corresponding clusters

### Node Manager Setup
- Type: Per Domain
- Username/Password: `weblogic / Welcome1`

---

## Step 5: Configure Node Manager (on all nodes)

### Copy Domain from Admin Node
```bash
scp -r oracle@admin-node:/u01/app/oracle/user_projects/domains/soa_domain /u01/app/oracle/user_projects/domains/
```

### Start Node Manager
```bash
cd $DOMAIN_HOME/bin
./startNodeManager.sh
```

### Add Machines in Admin Console
- Go to Environment → Machines → New
- Assign Node Manager to correct machine
- Map servers to machines accordingly

---

## Step 6: Start Servers

### Start Admin Server
```bash
cd $DOMAIN_HOME/bin
./startWebLogic.sh
```

### Start Managed Servers
- Via Admin Console → Environment → Servers → Start

### Verify Clusters
- Admin Console → Environment → Clusters

### Deploy Sample App to Validate

---

## Post-Configuration

- **Load Balancer**:
  - osb.example.com → osb_cluster
  - ess.example.com → ess_cluster
  - soa.example.com → soa_cluster

- **JMS/Persistence**:
  - Set up JDBC persistence and stores

- **Monitoring**:
  - Enable WLDF
  - Use Coherence for ESS caching

---

## Troubleshooting

- **Node Manager**:
  - Check: `$DOMAIN_HOME/nodemanager/nodemanager.log`
  - Validate: `nodemanager.properties` → `ListenAddress=0.0.0.0`

- **Startup Failures**:
  - Review: `$DOMAIN_HOME/servers/<server>/logs/*.out`

- **Cluster Issues**:
  - Check multicast/unicast port availability (7001–9000)

