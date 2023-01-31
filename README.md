<img src="https://companieslogo.com/img/orig/MDB_BIG-ad812c6c.png?t=1648915248" width="50%" title="Github_Logo"/> <br>

# Single Sign On in MongoDB Atlas
Atlas의 사용자 접근 제어를 위한 Single Sign On 설정 과정을 보여 줍니다.
SSO는 SAML 방식을 사용 하며 이를 지원하는 SSO 솔루션과는 연계 가능합니다. 이번 내용에서는 Open Source 인 keycloak를 이용하여 SSO를 진행 합니다.

Atlas 는 Email을 이용한 로그인 방식을 사용합니다. 따라서 계정은 이메일이 됩니다. 
지원 되는 Atlas의 인증 방법은 다음과 같습니다.
- 이메일 및 패스워드
- Social (Gmail, Github)을 이용한 인증
- SAML 방식의 SSO

SAML 방식의 로그인은 기업등에서 많이 사용하는 인증 방법으로 Atlas는 계정(이메일) 정보만을 가지고 있고 인증은 SSO 솔루션이 진행 합니다. 
Atlas에 SAML Federation 설정을 하기 위해서는 Email 주소의 도메인이 필요 합니다. 즉, user@email.com 인 경우 email.com 도메인에 대한 소유를 하고 있어야 합니다. Atlas에 내장된 IAM 시스템이 user@email.com 사용자가 로그인을 시도 할 때 email.com 도메인과 연계된 SAML Federation 서버를 검색 하고 해당 SSO 서비스로 인증을 요청 하는 구조 입니다.

테스트를 위해 필요한 준비는 다음과 같습니다.
- Email Domain 소유 (테스트에서는 nosql.site를 사용)
- Identity Access Management System (IAM - SSO 제공) : Keycloak 사용
- IAM 의 계정 저장소 (LDAP) : Active Directory 사용 (Option)
- Atlas 관리자 권한

전체 구성은 다음과 같습니다.
<img src="/images/image01.png" width="80%" height="80%">    

### KeyCloak 과 Active Directory
KeyCloak은 계정 및 접근 관리를 제공하는 서비스로 Redhat 주도의 Open Source 입니다. 
프로그램을 다운 받아 설치가 가능하며 이번 테스트에서는 간단히 Docker를 이용하여 구동 합니다. 운영 환경에서는 구성시 보안을 위한 Domain 구성 및 TLS 적용을 권고 합니다. 
Docker 가 설치된 인스턴스에서 다음 명령어로 컨테이너를 구동 합니다. 사용하는 포트는 8080을 사용 합니다.

````bash
$ docker run -d -p 8080:8080 -e KEYCLOAK_USER=admin -e KEYCLOAK_PASSWORD=<<password>> --name atlas_sso jboss/keycloak:latest
Unable to find image 'jboss/keycloak:latest' locally
latest: Pulling from jboss/keycloak
ac10f00499d5: Pull complete 
96d53117c12e: Pull complete 
1d929376eb7f: Pull complete 
93e1e1b6d192: Pull complete 
f353ba0db29e: Pull complete 
Digest: sha256:abdb1aea6c671f61a594af599f63fbe78c9631767886d9030bc774d908422d0a
Status: Downloaded newer image for jboss/keycloak:latest
d0c236f392013e1b470cd5b5e0b0404e2a61aa110aee353fd5d93fe8e3fb2c76
````
컨테이너가 구동되면 http://localhost:8080 으로 접근이 가능 합니다. 접근 계정은 설정한 admin/<<password>> 로 접근이 가능 합니다. 외부 IP를 이용한 로그인을 하는 경우 https가 적용되지 않아 불가능 합니다. 이를 해제 하기 위해서는 우선 localhost에 관리자로 로그인 후 Realm Settings 에서 Login 탭에서 Require SSQL을 none으로 수정 하여 줍니다.
<img src="/images/image03.png" width="80%" height="80%"> 

접근 메뉴중 Administration Console을 클릭 하면 관리자 로그인이 됩니다.   
<img src="/images/image02.png" width="80%" height="80%"> 

Keycloak은 인증, 권한의 부여 적용 범위를 Realm으로 관리 합니다. 1개 이상을 생성하여 realm 단위로 인증, 권한을 설정 할 수 있습니다. 기본적으로 생성된 realm은 master 이며 테스트에서는 master realm을 이용 합니다.

Active Directory는 Mircorsoft에서 제공하는 계정 저장소로 다양한 서비스를 제공 합니다. 테스트에서는 AD에 계정을 생성 하여 방식으로 진행 합니다. (AC가 없는 경우 직접 Keycloak에서 User 생성합니다)

Windows Server 에 Active Directory 서비스를 구성 합니다. 
Windows 2012 R2이상에서 구성이 가능합니다.
https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/deploy/install-active-directory-domain-services--level-100-

구성 과정에서 도메인네임을 입력 하여야 하며 테스트에서는 nosql.site를 이용합니다.
서비스 구성은 다음과 같으며 Docker가 구동되는 인스턴스와 Active Directory 서비스는 동일한 네트워크에 구성 하도록 합니다. (방화벽 등으로 인한 트래픽 차단을 확인 합니다)
<img src="/images/image04.png" width="80%" height="80%"> 

Keycloak에 계정 저장소 정보를 입력 하여 줍니다. 
관리자로 로그인 후 User Federation에서 ldap을 추가 하여 줍니다.
설정 중 Edit mode 는 READ_ONLY로 선택 하고 Vendor를 Active Directory로 선택 합니다.
Connection URL 부분에 ldap://Active Directory IP 로 입력 하여 주고 연결 테스트를 합니다.
<img src="/images/image05.png" width="80%" height="80%"> 

Users DN 은 CN=Users,DC=nosql,DC=site 로 입력 하여 주고 Bind DN은 CN=Administrator,CN=Users,DC=nosql,DC=site 를 입력 하고 Bind Credential 에 Administrator의 패스워드를 입력 해 줍니다. (별도 Active Directory를 읽을 수 있는 계정이 있는 경우 해당 계정 정보를 입력 하여 줍니다.)
인증 테스트가 성공 하면 저장 합니다.

사용자 정보를 보기 위해 Manage/Users 메뉴를 클릭 하고 View all users 버튼을 클릭 합니다.
<img src="/images/image06.png" width="80%" height="80%"> 

Active Directory에 저장 된 사용자 정보를 조회 할 수 있습니다. Active Directory를 사용하지 않는 경우 Add Users 버튼을 클릭 하여 사용자를 추가 합니다.

SAML Client 를 생성하기 위해 Realm Settings의 General 탭을 선택 합니다.
내용 중 enpoints 정보에서 SAML 2.0 identity Provider Metadata를 클릭 합니다.
<img src="/images/image07.png" width="80%" height="80%"> 

SAML Metadata XML 파일을 저장 합니다.

### Atlas 준비
Atlas 계정은 이메일 형태로 구성되며 로그인시 이메일의 도메인을 이용하여 로그인 방법이 지정 됩니다. 따라서 이메일의 도메인에 대해 관리 권한을 가지고 있어야 합니다. 도메인은 nosql.site를 이용할 것이며 사용자 계정은 ***@nosql.site 가 됩니다.

Atlas 에 관리자 권한으로 로그인 합니다.    
로그인 후 관리자 페이지로 이동 합니다. (상단의 톱니 아이콘 클릭)    
<img src="/images/image11.png" width="90%" height="90%">      

관리자 페이지에서 Settings 에 Setup Federation Settings 의 Visit Federation Management App 를 클릭 합니다.
<img src="/images/image12.png" width="90%" height="90%">      

#### Domain 설정
Domain 탭에서 Add Domains 를 클릭 합니다.   
<img src="/images/image13.png" width="85%" height="85%">    

이름과 도메인 명을 입력 하여 줍니다.
<img src="/images/image14.png" width="85%" height="85%">    

도메인 소유 방법을 증명 하기 위한 방법을 선택 합니다. 별도 웹서버 없이 도메인관리 툴에서 지정된 텍스트를 도메인에 등록 하는 방법을 선택 합니다. (도메인 웹서버가 있는 경우 지정된 html 을 올려서 할 수도 있습니다)
<img src="/images/image15.png" width="90%" height="90%">    

도메인 소유를 증명하기 위해 임의의 텍스트 스트링이 생성 되며 이를 도메인에 Txt 레코드로 등록 하여야 합니다.

<img src="/images/image16.png" width="90%" height="90%">    

GoDaddy 에서 구매한 도메인으로 해당 사이트에서 DNS 레코드를 등록 할 수 있습니다.  
godaddy.com 에 로그인 후 소유 도메인(nosql.site)에서 DNS 관리를 선택 합니다.
<img src="/images/image17.png" width="80%" height="80%">    

추가 버튼을 클릭 후 record type 을 txt 로 선택 합니다.    
값 필드에 Atlas 에서 생성된 임의 스트링 값을 입력 하여 줍니다.    
<img src="/images/image18.png" width="80%" height="80%">    


TTL 은 최소 시간인 30분을 선택 하고 저장 합니다. 도메인이 적용 되는 시간은 최대 1시간 까지 걸립니다.
Atlas 페이지에서 Finish 버튼을 클릭 합니다.    
<img src="/images/image19.png" width="90%" height="90%">      

등록된 도메인이 확인 되지 않은 상태로 있으며 몇 분 후 verfiy 버튼을 클릭 합니다.
<img src="/images/image20.png" width="90%" height="90%">    

확인이 완료 되면 Verified 상태가 되며 등록이 완료 됩니다.
<img src="/images/image21.png" width="90%" height="90%">    

(Domain 에 등록한 Txt record 는 삭제 하여 줍니다.)

#### Identity Provider 등록
Keycloak 정보(Identity Provider)를 등록 하여 줍니다.
Identity Providers 탭을 클릭 후 Setup Identity Provider 를 클릭 합니다.
<img src="/images/image40.png" width="90%" height="90%">    

IDP 이름을 입력하고 Issue URI 을 입력 하여 줍니다.
(이전 작업에서 기록한 EntityID를 입력 합니다.)
Single Sign-On URL은 Keycloak의 Meta 데이터의 SingleSignOneServie 의 Location 주소를 입력 하여 줍니다.
<img src="/images/image42.png" width="90%" height="90%">    

인증서는 metadata 의 X.509Certificate의 내용을 텍스트 파일로 복사한 후 다음과 같은 형태로 생성하여 주고 파일을 cert.pem으로 저장 합니다.
-----BEGIN CERTIFICATE-----
<<Certification>>
-----END CERTIFICATE-----
<img src="/images/image43.png" width="90%" height="90%">    

Identity Provider Signature Certificate 에서 파일 선택을 클릭 하고 저장한 인증서 파일을 선택 하여 줍니다.

Request Binding 은 Http Post를 선택 하고 Response Signature Algorithm 은 SHA-256을 선택 하여 줍니다.
<img src="/images/image44.png" width="50%" height="50%">    

완료 버튼을 클릭하면 Atlas가 제공하는 SAML 의 Meta 정보를 받을 수 있는 페이지가 오픈 됩니다. Atlas의 Federation 정보를 저장하고 있는 metadata 파일을 다운로드 받습니다.
<img src="/images/image45.png" width="80%" height="80%">    

관련 도메인과 조직에 대한 정보를 입력 하여 줍니다.
<img src="/images/image46.png" width="75%" height="75%">    
버튼을 클릭 하면 기존에 입력 해준 도메인 및 Organization을 볼 수 있습니다. 알맞은 내용을 선택 하여 줍니다.    

Group 정보와 Organization 간의 Mapping 을 등록 하기 위해 Organizations 를 선택 합니다.
<img src="/images/image47.png" width="75%" height="75%">    
SSO 로 접근 한 사용자의 기본 권한을 선택 합니다. 테스트에서는 Read-only 형태로 Organization Read Only 를 선택 합니다.  이후 Create Role Mapping 을 클릭합니다.
<img src="/images/image48.png" width="75%" height="75%">    

Role 이름으로 /atlas_owner 와 /atlas_member,/atlas_limited를 생성 하여 줍니다. (테스트에서는 3개의 권한을 생성 할 것이며 필요에 따라 추가로 생성 할 수 있습니다.)
<img src="/images/image49.png" width="75%" height="75%">    

Atlas_owner는 Organization 의 Owner Role로 다음과 같이 입력 하여 줍니다.
<img src="/images/image50.png" width="75%" height="75%">    

Project에 대한 권한을 선택 하여 줍니다. Owner Role 임으로 관리 권한을 선택 하여 줍니다.
<img src="/images/image51.png" width="75%" height="75%">    


Atlas_member 는 Member 성격으로 다음과 같이 등록 하여 줍니다. (프로젝트에 대한 데이터 읽기/쓰기 권한)
<img src="/images/image52.png" width="75%" height="75%">   

Project에 대한 권한을 선택 하여 줍니다. member Role 임으로 데이터 읽기 쓰기 권한을 부여 합니다.
<img src="/images/image53.png" width="75%" height="75%">    


Atlas_limited 는 최소 권한을 주는 것으로 가정 하여 Organization에 읽기 권한을 가지도록 합니다.
<img src="/images/image54.png" width="75%" height="75%">   

Project에 대한 권한은 권한을 쵀소화하여 데이터에 대한 앍가/쓰기 모두 제한 합니다.
<img src="/images/image55.png" width="75%" height="75%">    


### KeyCloak SAML Client 등록

Keycloak에 SAML Federation Client 로 Atlas 를 등록 하여 줍니다. 우선 Keycloak에 관리자로 로그인 후 Clients 메뉴를 클릭 후 Create를 클릭 합니다.
<img src="/images/image60.png" width="75%" height="75%">     

Client 정보를 등록 하는 화면에서 import 의 Select file버튼을 클릭 한 후 Atlas 에서 다운로드한 metadata.xml을 선택 하여 줍니다.
<img src="/images/image61.png" width="75%" height="75%">    

Metadata.xml파일을 읽어서 설정 정보가 기본적으로 등록 되어 지게 됩니다. 저장을 하면 상세 정보를 편집 할 수 있습니다.
기본 정보 중 이름과 상세 설명을 적절히 작성 하여 줍니다.
<img src="/images/image62.png" width="75%" height="75%">    
 

상세 정보 중 Encrypt Assertion 을 Off 하여 줍니다. (메시지를 암호화하지 않고 Signature 만 전달 합니다.) 또한 Email로 ID가 전달 되게 됨으로 NameID formation 을 email 로 하고 Force Name ID Formation 을 On 하여 줍니다.
<img src="/images/image63.png" width="75%" height="75%">    
 

Mappers 탭에서 SAML을 통해 전달 할 데이터를 지정 하여 줍니다. 기본 email, firstName, lastName, mobilePhone, memberOf가 생성 됩니다. 우선 SAML에 전달되는 기본 Attribute (NameID)를 email로 지정하기 위해 Create 버튼을 클릭 합니다.
<img src="/images/image64.png" width="75%" height="75%"> 

사용자 정보 중 email 정보를 NameID와 매핑하여 주는 것으로 다음과 같이 입력 하여 줍니다.
<img src="/images/image65.png" width="75%" height="75%"> 


Name 은 적절한 것으로 입력 하여 주고 Mapper Type을 User Attribute Mapper For NameID를 선택 하여 줍니다. 이후 Name ID format을 emailAddress를 선택 하여 주고 사용자 정보의 email 과 매핑하여 주기 위해 User Attribute 에 email을 입력 하고 저장 합니다.
firstName을 클릭 하고 User Attribute 에 firstName을 입력 합니다. (firstName 항목에 이름 – firstName을 전달 하는 것입니다.) 
<img src="/images/image66.png" width="75%" height="75%"> 
 

동일한 방법으로 Last Name을 수정 하여 줍니다. User Attribute에 lastName을 입력 하여 줍니다.   
<img src="/images/image67.png" width="75%" height="75%"> 


사용자의 소속 그룹 정보는 memberOf로 전달 됩니다. 해당 그룹 정보를 전달 하기 위해 기존 생성된 memberOf를 삭제하고 다시 생성 하여 줍니다.

Mapper Type 을 Group list로 하며 Group attribute name을 memberOf 로 하여 줍니다.
<img src="/images/image68.png" width="75%" height="75%"> 

### KeyCloak Group 등록
SAML Federation 과정에서 사용자의 인증 정보를 전달시 권한 정보 또한 전달 됩니다.
권한 정보는 KeyCloak에서 Group으로 설정이 가능합니다. Atlas와 연계한 권한 테스트를 위해 권한으로 atlas_owner, atlas_member, atlas_limited를 구성하였습니다. 이에 대한 권한 설정을 위해 Keycloak에 연결되는 Group을 생성하여 줍니다.
KeyCloak의 Group은 동일한 이름으로 생성하여 줍니다.

그룹 생성을 위해 Keycloak에 관리자로 로그인 하고 manage에 groups 메뉴를 선택 합니다.
New 버튼을 클릭하여 그룹을 생성 하여 줍니다.
<img src="/images/image70.png" width="75%" height="75%"> 

첫번째로 group 이름을 atlas_owner로 지정 합니다.
<img src="/images/image71.png" width="75%" height="75%">

동일한 방법으로 atlas_member, atlas_limited를 생성하여 줍니다.
<img src="/images/image72.png" width="75%" height="75%">

SAML 방식으로 SSO가 될 때 memberOf라는 이름으로 소속된 Group의 이름을 전달 하게 됩니다. Atlas는 memberOf 컬럼을 읽어서 권한과 매핑하여 권한을 할당하는 방식으로 작동 합니다.

### SSO 테스트

#### AD User 생성
Keycloak에 사용자를 추가 하여 테스트 합니다. 사용자 정보는 Active Directory에 있음으로 사용자를 추가 하여 줍니다. (AD가 없는 경우 keycloak의 manage/User 메뉴에서 사용자를 추가하여 줍니다)
Full name이 Keycloak에 로그인하는 사용자 이름임으로 atlas로 하여 줍니다.
<img src="/images/image90.png" width="75%" height="75%">

사용자 정보 중 email을 반드시 입력 하여 줍니다.
 
이메일의 도메인은 반드시 설정한 도메인 (nosql.site)와 동일하여야 합니다.
생성된 정보는 Keycloak의 관리자 페이지에 User 메뉴에서 확인 할 수 있습니다.  (데이터가 동기화 되어 있지 않은 경우 User Federation 메뉴에 등록된 active directory 정보에서 Synchronize all users 혹은 Synchronize changed users 를 클릭합니다.)
<img src="/images/image91.png" width="75%" height="75%">

Keycloak의 Manage/Users에서 사용자를 조회 합니다.
<img src="/images/image92.png" width="75%" height="75%">

테스트 사용자 atlas의 ID를 클릭 선택 하여 group 정보에서 소속 그룹을 추가 하여 줍니다.

Join 을 클릭하여 주면 atlas 사용자는 atlas_owner 권한을 가지게 됩니다.
<img src="/images/image93.png" width="75%" height="75%">
 
#### MongoDB Atlas Login
새롭게 브라우저를 오픈하고 MongoDB Atlas 에 로그인을 위해 접속 합니다. 사용자는 입력한 사용자 atlas@nosql.site 를 입력합니다.

<img src="/images/image94.png" width="75%" height="75%">

Single Sign On 설정에 따라 keycloak 로그인 페이지로 이동 합니다. 로그인을 위해 사용자 ID를 입력 합니다.

<img src="/images/image95.png" width="75%" height="75%"> 
패스워드 인증이 완료 되면 Single Sign On 을 통해 추가 인증 없이 Atlas Console 로 이동 됩니다.
로그인 된 사용자 정보를 보면 Single Sign On을 통해 사용자가 로그인 된 것을 확인 할 수 있습니다.
 
사용자 권한을 확인 하기 위해 사용자 Preference 를 클릭 합니다.
<img src="/images/image96.png" width="75%" height="75%"> 
권한이 owner 로 설정 된 것을 확인 할 수 있습니다. (atlas_owner에 소속됨으로 해당 권한을 가지게 됩니다.)
<img src="/images/image97.png" width="75%" height="75%"> 


#### 권한 테스트
Keycloak를 관리자로 로그인 한 후 atlas 사용자의 소속 그룹을 atlas_limited로 변경 하여 줍니다.
Users에서 사용자를 선택 후 Groups 정보에서 기존 소속 그룹을 삭제하고 atlas_limited를 추가 하여 줍니다.

<img src="/images/image98.png" width="75%" height="75%"> 

다시 로그인을 진행 하고 User Preferences를 클릭 하고 권한을 확인 합니다.
atlas_limited에 설정된 것과 같이 권한이 제한된 Organization Read only 인것을 볼 수 있습니다.
<img src="/images/image99.png" width="75%" height="75%"> 

데이터 읽기/쓰기 권한을 확인 하기 위해 데이터 베이스 클러스터의 데이터 탐색을 클릭 합니다.
읽기 권한이 없음으로 읽을 수 없는 것을 확인 합니다.
<img src="/images/image100.png" width="75%" height="75%"> 
