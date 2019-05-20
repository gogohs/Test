# Member
<ul>
 <li> 09231 강영훈 </li>
 <li> 09232 고현숙 </li>
 <li> 09277 박한주 </li>
 <li> 09340 이호영 </li>
</ul>

![Alt text](https://github.com/gogohs/skccBigData/blob/master/kangyounghun2.png)
![Alt text](https://github.com/gogohs/skccBigData/blob/master/HyunSook.PNG)
![Alt text](https://github.com/gogohs/skccBigData/blob/master/Hanjoo.PNG)
![Alt text](https://github.com/Lee-Ho-Young/Bigdata_0416/blob/master/picture.png)



## 실습목표
<ul>
 <li> Cloudera Manager를 사용한 Hadoop Cluster 구성 </li>
</ul>


## 환경구성
<ul>
 <li> CENTOS 7 </li>
 <li> JDK 1.78 </li>
 <li> CDH 5.16</li>
</ul>

## AWS Instance 접속
<ul>
 <li> Git bash 접속하여 AWS 접속을 위한 key파일 디렉토리로 이동 [skcc.pem] </li>
 <li> AWS CentOs Instance 접속계정명 : centos </li>
</ul>

```
$ ssh -i skcc.pem centos@13.124.227.184
```

## CentOS 7 관련
<ul>
 <li> service -> systemctl </li>
 <li> chkconfig -> systemctl </li>
 <li> iptables -> firewalld </li>
</ul>

```
<CentOS 6>
$ service cloudera-scm-server start  //cloudera-scm-server 서비스 시작
$ service cloudera-scm-server stop   //cloudera-scm-server 서비스 종료
$ service cloudera-scm-server status //cloudera-scm-server 서비스 상태
$ chkconfig cloudera-scm-server on   //서버 부팅 시 자동실행 설정
$ chkconfig cloudera-scm-server off  //서버 부팅 시 자동실행 해제

<CentOS 7>
$ systemctl start cloudera-scm-server  //cloudera-scm-server 서비스 시작
$ systemctl stop cloudera-scm-server   //cloudera-scm-server 서비스 종료
$ systemctl status cloudera-scm-server //cloudera-scm-server 서비스 상태 
$ systemctl enable cloudera-scm-server //서버 부팅 시 자동실행 설정
$ systemctl disable cloudera-scm-server//서버 부팅 시 자동실행 해제
$ systemctl is-enable 서비스명          //서비스가 자동실행 설정상태인지 확인
$ systemctl list-unit-files --type=service //자동실행 설정된 서비스 리스트 
```

## Pre-Install Step

#### 1. yum update

```
$ sudo yum update
$ sudo yum install -y wget
```

#### 2. firewall 정지 [방화벽 정지, CentOs 7부터 iptables -> firewalld 변경]
** 대상 : Cluster 전체 Host **
<ul>
 <li> stop : 현재 작동중인 firewall 서비스 종료 </li>
 <li> disable : OS 부팅 시 firewall 자동실행 해제 </li>
</ul>

```
$ systemctl stop firewalld.service
$ systemctl disable firewalld.service
```

#### 3. Selinux 정지 [보안 프로그램]
** 대상 : Cluster 전체 Host **
<ul>
 <li> sestatus : Selinux 동작모드 확인 [Default : enforcing] </li>
 <li> /etc/selinux/config 파일을 수정하여 상태를 변경 [enforcing -> disabled] </li>
</ul>

```
$ sestatus
$ sudo  vi /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
# enforcing - SELinux security policy is enforced.
# permissive - SELinux prints warnings instead of enforcing.
# disabled - SELinux is fully disabled.
SELINUX=disabled
# SELINUXTYPE= type of policy in use. Possible values are:
# targeted - Only targeted network daemons are protected.
# strict - Full SELinux protection.
SELINUXTYPE=targeted

$ sudo reboot
```

#### 4. NTP 설정 [Cluster host 시간 동기화]
** 대상 : Cluster 전체 Host **
<ul>
 <li> NTP 설치 </li>
 <li> NTP 서버 설정 [/etc/ntp.conf 파일에서 기본설정 서버를 주석처리 후 한국의 ntp서버 추가]</li>
 <li> NTP 서비스 시작</li>
 <li> NTP 서비스 시작프로그램에 등록</li>
 <li> NTP 서비스 작동여부 확인</li>
</ul>

```
$ yum install -y ntp

$ sudo vi /etc/ntp.conf
# Use public servers from the pool.ntp.org project. 
# Please consider joining the pool (http://www.pool.ntp.org/join.html). 
#server 0.centos.pool.ntp.org 
#server 1.centos.pool.ntp.org 
#server 2.centos.pool.ntp.org
server kr.pool.ntp.org 
server time.bora.net
server time.kornet.net

$ systemctl start ntpd
$ systemctl enable ntpd
$ ntpq -p
```

#### 5. 추가적인 설정 [성능이슈]
[참고]https://www.cloudera.com/documentation/enterprise/latest/topics/cdh_admin_performance.html
## 5-1. Disable the tuned Service [시스템 모니터링 및 시스템 설정에 대한 동적튜닝을 제공하는 데몬]

```
$ systemctl start tuned        // Ensure that the tuned service is started
$ tuned-adm off                // Turn the tuned service off
$ tuned-adm list               // Ensure that there are no active profiles
-> No current active profile   // The output should contain the following line

$ systemctl stop tuned         // shutdown the service
$ systemctl disable tuned      // disable the service
```

#5-1. transparent huge page 설정
disable tuned 설정

#### 3. swappiness 설정
```
$ sysctl -w vm.swappiness=0
$ echo 'vm.swappiness=0' >> /etc/sysctl.conf
```


## CDH install
#### Configure a repository for cloudera manager
1. Download the cloudera-manager.repo file for your OS version to the /etc/yum.repos.d/ directory on the Cloudera Manager Server host
```
$sudo wget https://archive.cloudera.com/cdh5/redhat/7/x86_64/cdh/cloudera-cdh5.repo -P /etc/yum.repos.d/
```
2. Import the repository signing GPG key [RHEL 7]
```
$sudo rpm --import https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/RPM-GPG-KEY-cloude 
```


#### Configure a repository for cloudera manager
[참고] https://zetawiki.com/wiki/CentOS_JDK_%EC%84%A4%EC%B9%98

#### yum 설치가능 java 리스트 확인
```
yum list java*jdk-devel
```

#### Install Java
```
yum install java-1.7.0-openjdk-devel.x86_64
```
##### 설치확인 :
[centos@ip-172-31-46-77 ~]$ java -version
java version "1.7.0_221"
OpenJDK Runtime Environment (rhel-2.6.18.0.el7_6-x86_64 u221-b02)
OpenJDK 64-Bit Server VM (build 24.221-b02, mixed mode)
[centos@ip-172-31-46-77 ~]$ javac -version
javac 1.7.0_221


## 시스템 재부팅(sudo reboot)
#### Cloudera-manager server설치 [수동버전]
1.cloudera manager를 설치할 호스트에서 아래의 명령어 실행
```
$sudo yum install cloudera-manager-daemons cloudera-manager-server
```
2.Cluster 모든 호스트에 대해 cloudera agent 설치
```
$sudo yum install cloudera-manager-daemons cloudera-manager-agent
```

#### Cloudera-manager 설치 [자동버전]
```
1.  $ wget http://archive.cloudera.com/cm5/installer/5.15.2/cloudera-manager-installer.bin
```
> a. 개인정보 입력 후 다운로드 URL확인  
> b. 5.15.0 설치파일로 설치하니까 5.16버전이 깔렸음  
> c. 설치제거 후 재시도  $ sudo /usr/share/cmf/uninstall-cloudera-manager.sh


```
2.  $ wget http://archive.cloudera.com/cm5/installer/5.15.0/cloudera-manager-installer.bin
```  
> a. 현재 디렉토리에 설치 파일 다운로드 됨


```
3.  $ chmod u+x cloudera-manager-installer.bin
```  
> a. 설치파일의 권한 변경

```
4.  $ sudo ./cloudera-manager-installer.bin
```  
> a. 설치 실행

5. 설치 완료 후 접속정보 안내가 뜨고, 접속이 안될경우 iptable 같은 리눅스 방화벽 확인이 필요하다는 메시지 출력
> a. 접속정보 : 13.124.227.184:7180 [ admin / admin]


#### Database  [MariaDB 설치]
[참고]https://www.cloudera.com/documentation/enterprise/5-15-x/topics/cm_ig_installing_configuring_dbs.html
<ol>
<li>MariaDB 설치</li>
<li>JDBC Connector 설치</li>
<li>필요한 계정 생성</li> 
</ol>

```
Table
CREATE DATABASE scm DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE amon DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE rman DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE hue DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE metastore DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE sentry DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE nav DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE navms DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE oozie DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
```

```
권한
GRANT ALL ON scm.* TO  'scm'@'%' IDENTIFIED BY 'scm';   
GRANT ALL ON amon.* TO  'amon'@'%' IDENTIFIED BY 'amon';   
GRANT ALL ON rman.* TO  'rman'@'%' IDENTIFIED BY 'rman';  
GRANT ALL ON hue.* TO  'hue'@'%' IDENTIFIED BY 'hue';  
GRANT ALL ON metastore.* TO  'hive'@'%' IDENTIFIED BY 'hive';  
GRANT ALL ON sentry.* TO  'sentry'@'%' IDENTIFIED BY 'sentry';  
GRANT ALL ON nav.* TO  'nav'@'%' IDENTIFIED BY 'nav';  
GRANT ALL ON navms.* TO  'navms'@'%' IDENTIFIED BY 'navms';  
GRANT ALL ON oozie.* TO  'oozie'@'%' IDENTIFIED BY 'oozie';  
```

#### Set up the Cloudera Manager Database
1.scm_prepare_database.sh 사용하여 configuration 진행  
[MariaDB의 경우 세팅할때 mysql 옵션으로 넣어준다.]
```
파일명 :  /usr/share/cmf/schema/scm_prepare_database.sh 
실행명 : sudo ./scm_prepare_database.sh mysql scm scm
```

2.cloudera-scm-manager Database 설정을 완료한 후 서버 재기동을 해보면 에러없이 서버가 정상적으로 켜짐을 확인할 수 있다.    
> a. $ service cloudera-scm-server restart / status / stop / start / reset  
> b. 로그확인 : tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log  

3. Cloudera manager default port에 서비스가 떠 있는 것을 확인할 수 있다.  
> a. $ netstat -antp | grep 7180

#### Cloudera Manager Web UI : Install CDH and Other Software
1.Specify hosts for your CDH cluster installation
2.Select Repository
3.Accept JDK License
 > a. cloudera manager 호스트에만 JDK를 깔았기 때문에 설치하기로 설정해야 함
4. Single User Mode
 > a. 체크하지 않음
5. Enter Login Credentials
 > a. root 계정으로 접속하기 위한 비밀번호를 설정해야 함
 > b. root로 접속 / 비밀번호 : admin [설정이 그렇게 되었음]
6. Install Agents
> a. 5개 서버의 [Private IP FQDN] 정보 5줄을 입력하고 다음으로 넘어가면 연결이 실패했다고 나온다.
7.Install Parcels 
> a. Hadoop cluster serivce 선택하여 설치 [HDFS / YARN / Zookeeper  3개만 ]


## MASTER 및 SLAVE 노드 SSH 연결 PROCESS

#### 1. local에서 master 및 slave 서버 ssh로 붙기
-- skcc.pem 파일이 있는 디렉토리로 이동. 
-- $sudo ssh -i skcc.pem 계정(centos)@ip  로 연결

  
#### 2.  모든 서버의 root계정 passwd 설정.
```
$sudo su 
```
-- passwd 를 admin으로 입력.


#### 3.  모든 서버에서 host 파일에 서버정보입력
-- /etc로 이동.
```
sudo vi etc/hosts
```
-- public ip hostname 설정 후 저장.

#### 4.   모든 서버에서 sshd_config 파일 편집
```
$ sudo vi /etc/ssh/sshd_config 
```
-- PasswordAuthentication yes로 설정 및 저장.

#### 5.  설정 후 서비스 restart
```
$sudo service sshd restart
```

#### 6. (master노드만) ssh 용 public key 생성하기     
``` 
$ssh-keygen
```

#### 7. 1~5번까지 각 노드에서 완료 후 master노드에서 slave노드로 public key 정보 복사하기.
```
$cd /.ssh 이동
$sudo ssh-copy-id -i ~/.ssh/id_rsa.pub slave4.
```
-- (permission denied 문제가 발생하면 앞의 config 파일 및 서비스 재시작 여부 확인)
     
#### 8.  master root계정에서 ssh로 slave에 접속하기.
```
$sudo su
$ssh hostname
```

## Maria DB Installation
https://www.cloudera.com/documentation/enterprise/latest/topics/install_cm_mariadb.html#install_cm_mariadb_install

#### 1.     Maria DB 설치
```
$ sudo yum install mariadb-server
```
 
#### 2.     마리아디비 서버 스탑
```
$ sudo systemctl stop mariadb
``` 
 
#### 3. 	컨피그 파일 세팅
my.cnf (/etc/my.cnf by default).(가이드 권장 사항이나 수정할 필요없음)
===========================================================


```

[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
transaction-isolation = READ-COMMITTED
# Disabling symbolic-links is recommended to prevent assorted security risks;
# to do so, uncomment this line:
symbolic-links = 0
# Settings user and group are ignored when systemd is used.
# If you need to run mysqld under a different user or group,
# customize your systemd unit file for mariadb according to the
# instructions in http://fedoraproject.org/wiki/Systemd
 
key_buffer = 16M
key_buffer_size = 32M
max_allowed_packet = 32M
thread_stack = 256K
thread_cache_size = 64
query_cache_limit = 8M
query_cache_size = 64M
query_cache_type = 1
 
max_connections = 550
#expire_logs_days = 10
#max_binlog_size = 100M
 
#log_bin should be on a disk with enough free space.
#Replace '/var/lib/mysql/mysql_binary_log' with an appropriate path for your
#system and chown the specified folder to the mysql user.
log_bin=/var/lib/mysql/mysql_binary_log
 
#In later versions of MariaDB, if you enable the binary log and do not set
#a server_id, MariaDB will not start. The server_id must be unique within
#the replicating group.
server_id=1
 
binlog_format = mixed
 
read_buffer_size = 2M
read_rnd_buffer_size = 16M
sort_buffer_size = 8M
join_buffer_size = 8M
 
# InnoDB settings
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit  = 2
innodb_log_buffer_size = 64M
innodb_buffer_pool_size = 4G
innodb_thread_concurrency = 8
innodb_flush_method = O_DIRECT
innodb_log_file_size = 512M
 
[mysqld_safe]
log-error=/var/log/mariadb/mariadb.log
pid-file=/var/run/mariadb/mariadb.pid
 
#
# include all files from the config directory
#
!includedir /etc/my.cnf.d

```
 ===========================================================


#### 4.   부팅시 마리아 디비 시작 확인
```
$ sudo systemctl enable mariadb
```
 
#### 5.   마리아디비 실행
```
$ sudo systemctl start mariadb
```
 
#### 6.     Maria DB rootpassword 및 보안 세팅 실행
```
/usr/bin/mysql_secure_installation
$ sudo /usr/bin/mysql_secure_installation
```
```
아래 실행 화면 따라 설치
[...]
Enter current password for root (enter for none):
OK, successfully used password, moving on...
[...]
Set root password? [Y/n] Y
New password:
Re-enter new password:
[...]
Remove anonymous users? [Y/n] Y
[...]
Disallow root login remotely? [Y/n] N
[...]
Remove test database and access to it [Y/n] Y
[...]
Reload privilege tables now? [Y/n] Y
[...]
All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.
 
Thanks for using MariaDB!
```

####  7. MariaDB를 위한 Mysql JDBC 드라이버 설치
JDBC 드라이버 다운로드
```
$ wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.46.tar.gz
```

JDBC 드라이버 압축해제
```
$ tar zxvf mysql-connector-java-5.1.46.tar.gz
```
/usr/share/java로 JDBC드라이버 복사 경로 없을시 생성
```
$ sudo mkdir -p /usr/share/java/
$ cd mysql-connector-java-5.1.46
$ sudo cp mysql-connector-java-5.1.46-bin.jar /usr/share/java/mysql-connector-java.jar
 ```
 
####  8.     데티어베이스 생성
8-1  root 유저 로그인
```
$ mysql -u root -p
Enter password:
```

