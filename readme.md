# Spring PetClinic (JNDI + AWS Secrets Manager + Apache mod_proxy_http)

Java Spring 기반의 PetClinic 샘플 앱.  
**DB 비밀번호를 코드/설정에 저장하지 않고**, **Tomcat JNDI + AWS Secrets Manager JDBC**로 **RDS(MySQL)** 에 안전하게 연결합니다.  
Web 계층은 **Apache HTTP Server(mod_proxy_http)** 로 리버스 프록시하고, WAS는 **Tomcat 9** 를 사용합니다.

## 아키텍처 개요
- **3-Tier**
    - **Web**: Apache HTTP Server (mod_proxy_http)
    - **WAS**: Tomcat 9 (Amazon Corretto 8)
    - **DB**: Amazon RDS (MySQL)
- **보안 포인트**
    - DB 자격증명은 **AWS Secrets Manager**에만 저장
    - 애플리케이션은 **JNDI DataSource**로만 DB 접근
    - EC2 IAM Role 최소 권한(`secretsmanager:GetSecretValue`)
    - Web↔WAS는 프라이빗 네트워크 통신(보안그룹으로 포트 최소 개방)
## 로컬/IDE 빌드

> 로컬에서는 DB 접속 테스트 없이 **빌드만** 진행합니다.

빌드:

    mvn -U clean package

- Windows PowerShell에서 Maven/JDK 명령 인식 안 되면:
    - JDK `bin`, Maven `bin`을 PATH에 추가하거나
    - IntelliJ의 Maven 패널(우측)에서 빌드

생성물: `target/petclinic.war`
## 애플리케이션 변경점(요약)

- `pom.xml`: `aws-secretsmanager-jdbc` 의존성 추가 (`runtime`)
- `spring/datasource-config.xml`: **JNDI Lookup만 사용**
- `spring/business-config.xml`: XSD 헤더 정리, 프로필 구조 유지
- `spring/data-access.properties`: `jdbc.url/username/password` 제거
- `META-INF/context.xml`: **Secrets Manager JDBC** 기반 **JNDI DataSource** 정의
- `WEB-INF/web.xml`: `resource-ref` 선언
## AWS 배포 (Tomcat JNDI + Secrets Manager)

### 1) 사전 준비
- **RDS(MySQL)**: DB 생성 (예: `petclinic`)
- **Secrets Manager**: 시크릿 생성(이름 예: `petclinic/rds/mysql`)
    - 최소 키: `username`, `password` (host/dbname은 URL로 전달)
- **IAM Role(EC2 인스턴스 프로파일)** 예시 정책:

        {
          "Version": "2012-10-17",
          "Statement": [{
            "Effect": "Allow",
            "Action": "secretsmanager:GetSecretValue",
            "Resource": "arn:aws:secretsmanager:ap-northeast-2:<ACCOUNT_ID>:secret:petclinic/rds/mysql-*"
          }]
        }

- **보안그룹**
    - Web 인스턴스: 80(또는 443) Inbound from Internet
    - WAS 인스턴스: 8080 Inbound only from Web SG
    - RDS: 3306 Inbound only from WAS SG

### 2) WAR 내부 JNDI 설정 (이미 포함됨)
`src/main/webapp/META-INF/context.xml` 에 **JNDI DataSource**가 정의되어 있으며, 배포 시 자동 인식됩니다.  
아래 값만 환경에 맞게 수정 후 빌드하세요.

    <Resource name="jdbc/petclinic"
              type="javax.sql.DataSource"
              factory="org.apache.tomcat.jdbc.pool.DataSourceFactory"
              driverClassName="com.amazonaws.secretsmanager.sql.AWSSecretsManagerMySQLDriver"
              url="jdbc-secretsmanager:mysql://<RDS_ENDPOINT>:3306/petclinic?useUnicode=true&characterEncoding=utf8"
              username="arn:aws:secretsmanager:ap-northeast-2:<ACCOUNT_ID>:secret:petclinic/rds/mysql-XXXXXXXX"
              password=""
              initialSize="2" maxActive="20" minIdle="2"
              testOnBorrow="true" validationQuery="SELECT 1"
              removeAbandoned="true" removeAbandonedTimeout="60"
              jdbcInterceptors="ConnectionState;StatementFinalizer"/>

> `username`에는 **시크릿 이름** 또는 **ARN**을 사용할 수 있습니다. 운영 고정이면 **ARN 권장**.  
> 비밀번호는 **공란**(드라이버가 시크릿에서 조회).

### 3) Spring 프로필 활성화
JPA 사용을 위해 **WAS에 프로필을 지정**합니다.

- systemd 예 (`tomcat.service`):

        Environment="CATALINA_OPTS=-Dspring.profiles.active=jpa"

- 수동 기동 예:

        export CATALINA_OPTS="$CATALINA_OPTS -Dspring.profiles.active=jpa"

### 4) 배포/기동
1) `target/petclinic.war` → `${CATALINA_BASE}/webapps/` 업로드
2) 기존 `webapps/petclinic/`(exploded) 디렉터리가 있으면 삭제
3) Tomcat 재시작:

        systemctl restart tomcat
        # 또는
        $CATALINA_BASE/bin/catalina.sh restart

### 5) 헬스 체크
- 접속: `http://<WEB or LB>` (Apache가 80/443으로 프록시)
- 로그:
    - `${CATALINA_BASE}/logs/catalina.out`
- 성공 신호:
    - JNDI lookup 성공, `SELECT 1` 검증 OK
- 실패 힌트:
    - `AccessDeniedException` → IAM/시크릿 권한 확인
    - `Communications link failure` → SG/네트워크 확인
    - `No suitable driver` → `driverClassName`/`jdbc-secretsmanager:` 접두어 확인
## Apache (Web) 설정 – mod_proxy_http

> Web 인스턴스(Apache)가 WAS(Tomcat:8080)로 프록시합니다.  
> SSL을 사용할 경우 `SSLCertificateFile/KeyFile` 추가와 `VirtualHost *:443` 구성으로 확장하세요.

### 모듈 활성화(Amazon Linux 예)

    sudo dnf install -y httpd
    sudo systemctl enable --now httpd

모듈 로드(배포판에 따라 다를 수 있음):  
`/etc/httpd/conf.modules.d/00-proxy.conf` 등에 다음이 포함되어야 함

    LoadModule proxy_module modules/mod_proxy.so
    LoadModule proxy_http_module modules/mod_proxy_http.so
    # (SSL 쓸 때)
    LoadModule ssl_module modules/mod_ssl.so
    # (보안 헤더)
    LoadModule headers_module modules/mod_headers.so

### 가상호스트(리버스 프록시) 예시

    <VirtualHost *:80>
        ServerName <YOUR_DOMAIN_OR_PUBLIC_IP>

        # 보안 헤더(옵션)
        Header always set X-Content-Type-Options "nosniff"
        Header always set X-Frame-Options "SAMEORIGIN"
        Header always set X-XSS-Protection "1; mode=block"

        # 프록시 설정
        ProxyPreserveHost On
        ProxyRequests Off

        # 정방향 프록시 금지(보안)
        <Proxy "*">
            Require all denied
        </Proxy>

        # 리버스 프록시
        ProxyPass        "/" "http://<WAS_PRIVATE_IP_or_DNS>:8080/"
        ProxyPassReverse "/" "http://<WAS_PRIVATE_IP_or_DNS>:8080/"

        ErrorLog  /var/log/httpd/petclinic_error.log
        CustomLog /var/log/httpd/petclinic_access.log combined
    </VirtualHost>

포인트
- Apache SG는 80(또는 443)만 외부로 오픈
- WAS SG는 8080 포트만 **Apache SG로부터** 허용
- `ProxyPreserveHost On`으로 원래 Host 헤더 유지(앱이 절대경로 링크 생성 시 유리)
## 트러블슈팅

- **Maven이 `aws-secretsmanager-jdbc`만 못 받는 경우**
    - Maven Central 미러/프록시/캐시 문제일 확률 높음
    - 조치: *Reload All Maven Projects*, 필요 시 `pom.xml`에 `<repositories>`로 Central 지정, `~/.m2` 캐시 삭제, 오프라인 모드 해제
- **`URI is not registered` 경고**
    - 스키마 헤더를 정석으로 교체
    - `p` 네임스페이스는 **beans 스키마 축약** → **Fetch 대상 아님**(Ignore 처리 또는 beans.xsd로 매핑)
- **`AccessDeniedException` (Secrets Manager)**
    - EC2 IAM Role 정책의 `Resource` ARN 범위/리전 확인
- **프록시 루프/404**
    - Apache `ProxyPass` 경로 슬래시(`/`) 매칭, Tomcat 컨텍스트 경로 확인
- **한/영 문자 깨짐**
    - `?useUnicode=true&characterEncoding=utf8` 옵션 확인
    - Apache `AddDefaultCharset Off` 검토

## 라이선스
본 프로젝트는 학습/실습 목적 샘플입니다.
