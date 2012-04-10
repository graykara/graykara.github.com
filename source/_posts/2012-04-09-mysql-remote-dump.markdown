---
layout: post
title: "MySQL DB 덤프 원격 자동 백업"
date: 2012-04-09 17:15
comments: true
categories: 
---

주기적으로(최소한 하루 한 번) MySQL database 를 백업하기 위해 cron job 을 사용하여 매일 마다 백업 후, ftp 를 통해 특정 서버에 덤프파일을 업로드하도록 설정합니다.

## 실행환경

* Ubuntu Server
* LFTP
* crontab

## LFTP Install

커맨드라인 ftp 툴인 lftp 패키지를 사용하기 위해서 우선 lftp 를 설치합니다.

```
$ sudo apt-get install lftp
```

## MySQL 서버의 백업을 위한 shell script

쉘 스크립트는 아래의 코드와 같습니다.
이를 통해서 /home/*username*/mysqldump/ 폴더에 특정 db 를 제외한 모든 데이터베이스의 덤프를 기록하고 FTP 서버에 업로드 됩니다. 
**사용자명**과 **비밀번호**는 스크립트를 사용하기 전에 설정해야 합니다.

{% codeblock mysql.backup.sh lang:sh %}
#!/bin/bash
### MySQL Server Login Info ###
MUSER="root"
MPASS=""
MHOST="localhost"
MYSQL="$(which mysql)"
MYSQLDUMP="$(which mysqldump)"
BAK="/home/username/mysqldump"
GZIP="$(which gzip)"
### FTP SERVER Login info ###
FTPU="ftp_user_name"
FTPP="ftp_password"
FTPS="ftp_address"
NOW=$(date +"%d-%m-%Y")
 
[ ! -d $BAK ] && mkdir -p $BAK || /bin/rm -f $BAK/*
 
DBS="$($MYSQL -u $MUSER -h $MHOST -Bse 'show databases')"
for db in $DBS
do
 if [ $db != "information_schema" ] && [ $db != "mysql" ] && [ $db != "dbname" ]
 then
   FILE=$BAK/$db.$NOW-$(date +"%T").gz
   $MYSQLDUMP -u $MUSER -h $MHOST $db | $GZIP -9 > $FILE
 fi
done
 
lftp -u $FTPU,$FTPP -e "mkdir ./$NOW;cd ./$NOW; mput ./mysqldump/*; quit" $FTPS
{% endcodeblock %}

/home/*username*/mysql.backup.sh 로 저장한 후에, 실행 권한을 줍니다.

```
$ chmod +x ~/mysql.backup.sh
```

최초 백업을 위해서 쉘 스크립트를 실행합니다.

```
$ ./mysql.backup.sh
```

## cron job 으로 MySQL 백업 자동화

cron job 에 작업을 등록하여 매일 정해진 시간에 위의 작업이 이루어질 수 있도록 합니다.

crontab 편집기를 실행하여 아래와 같이 설정합니다.

```
$ crontab -e
```

``` bash crontab
00 04 * * /home/username/mysql.backup.sh > /dev/null 2>&1
```

## 결과물

이로써 MySQL database Server 의 데이터베이스를 매일 백업해서 압축된 덤프파일을 특정 FTP 서버로 업로드 되도록 하는 작업이 완료되었습니다.

1. SQL database Server 의 덤프폴더에는 데이터베이스명.%d-%m-%Y-%T.gz 형태의 압축된 백업 파일이 생성될 됩니다.
2. 압축된 백업파일은 특정 FTP 서버에 날짜별 폴더를 만든 후, 각각 업로드 됩니다.
3. 다음 날, 새롭게 백업 파일을 만들 때 이전 백업파일이 존재하면 해당 내용은 삭제하여 MySQL database Server 상에는 데이터베이스 별로 최신 백업파일 하나만이 존재하게 됩니다.

이후, 특정 FTP 서버로 전송된 백업 파일은 외부 저장소에 주기적으로 2차 백업을 하여 데이터베이스 보존에 심혈을 기울여야할 것 입니다.