# 웹서버와 서블릿 컨테이너

## 웹서버와 스프링부트 소개


### **외장 서버 VS 내장 서버**

<img width="696" alt="Screenshot 2024-11-12 at 22 26 04" src="https://github.com/user-attachments/assets/bb2f97e5-3806-4a92-be22-48d4f890bafa">

**전통적인 방식**

과거에 자바로 웹 애플리케이션을 개발할 때는 먼저 서버에 톰캣 같은 WAS(웹 애플리케이션 서버)를 설치했다. 

그리고 WAS에서 동작하도록 서블릿 스펙에 맞추어 코드를 작성하고 WAR 형식으로 빌드해서 war 파일을 만들었다. 

이렇게 만들어진 war 파일을 WAS에 전달해서 배포하는 방식으로 전체 개발 주기가 동작했다.

이런 방식은 WAS 기반 위에서 개발하고 실행해야 한다. 

IDE 같은 개발 환경에서도 WAS와 연동해서 실행되도록 복잡한 추가 설정이 필요하다.

**최근 방식**

최근에는 스프링 부트가 내장 톰캣을 포함하고 있다. 

애플리케이션 코드 안에 톰캣 같은 WAS가 라이브러리로 내장되어 있다는 뜻이다. 

개발자는 코드를 작성하고 JAR로 빌드한 다음에 해당 JAR를 원하는 위치에서 실행하기만 하면 WAS도 함께 실행된다.

쉽게 이야기해서 개발자는 `main()` 메서드만 실행하면 되고, WAS 설치나 IDE 같은 개발 환경에서 WAS와 연동하는 복잡한 일은 수행하지 않아도 된다.

그런데 스프링 부트는 어떤 원리로 내장 톰캣을 사용해서 실행할 수 있는 것일까?

지금부터 과거로 돌아가서 톰캣도 직접 설치하고, WAR도 빌드하는 전통적인 방식으로 개발을 진행해보자. 

그리고 어떤 불편한 문제들이 있어서 최근 방식으로 변화했는지 그 과정을 함께 알아보자.

이미 잘 아는 옛날 개발자들은 WAS를 설치하고 war를 만들어서 배포하는 것이 익숙하겠지만, 최근 개발자들은 이런 것을 경험할 일이 없다. 

그래도 한번은 알아둘 가치가 있다. 과거에 어떻게 했는지 알아야 현재의 방식이 왜 이렇게 사용 되고 있는지 더 깊이있는 이해가 가능하다. 

서블릿 컨테이너도 설정하고, 스프링 컨테이너도 만들어서 등록하고, 디스패처 서블릿을 만들어서 스프링 MVC와 연결하는 작업을 스프링 부트 없이 직접 경험해보자. 

그러면 스프링 부트가 웹 서 버와 어떻게 연동되는지 자연스럽게 이해할 수 있을 것이다.

참고로 여기서는 `web.xml` 대신에 자바 코드로 서블릿을 초기화 한다. 

옛날 개발자라도 대부분 `web.xml` 을 사용했지 자바 코드로 서블릿 초기화를 해본 적은 없을 것이므로 꼭 한번 코드로 함께 따라해보자.

```shell

sudo lsof -i :8080 #프로세스 ID(PID) 조회 
sudo kill -9 PID  #프로세스 종료
```

## WAR 빌드와 배포

WAS에 우리가 만든 코드를 빌드하고 배포해보자.

**프로젝트 빌드**

프로젝트 폴더로 이동 

프로젝트 빌드

```shell
`./gradlew build`
```

WAR 파일 생성 확인 `build/libs/server-0.0.1-SNAPSHOT.war`

<img width="393" alt="Screenshot 2024-11-16 at 11 42 37" src="https://github.com/user-attachments/assets/147f1153-b242-4f92-972c-17b1df00864c">

**참고**

`build.gradle` 에 보면 `war` 플러그인이 사용된 것을 확인할 수 있다. 이 플러그인이 `war` 파일을 만들어준다.

```groovy
plugins {
    id 'java'
    id 'war' 
}
```

**WAR 압축 풀기**

우리가 빌드한 war 파일의 압축을 풀어서 내용물을 확인해보자.

`build/libs` 폴더로 이동하자.

다음 명령어를 사용해서 압축을 풀자

```shell
`jar -xvf server-0.0.1-SNAPSHOT.war`
```

**WAR를 푼 결과** 

<img width="417" alt="Screenshot 2024-11-16 at 11 47 43" src="https://github.com/user-attachments/assets/13ec3ab3-480f-4ec9-87e5-ddf6cd18af14">

* `WEB-INF`
  * `classes` 
    * `hello/servlet/TestServlet.class` 
  * `lib`
    * `jakarta.servlet-api-6.0.0.jar`
* `index.html`

WAR를 푼 결과를 보면 `WEB-INF` , `classes` , `lib` 같은 특별한 폴더들이 보인다. 이 부분을 알아보자.

### JAR, WAR 간단 소개 

**JAR 소개**

자바는 여러 클래스와 리소스를 묶어서 `JAR` (Java Archive)라고 하는 압축 파일을 만들 수 있다.

이 파일은 JVM 위에서 직접 실행되거나 또는 다른 곳에서 사용하는 라이브러리로 제공된다.

직접 실행하는 경우 `main()` 메서드가 필요하고, `MANIFEST.MF` 파일에 실행할 메인 메서드가 있는 클래스를 지정해 두어야 한다.

실행 예) `java -jar abc.jar`

Jar는 쉽게 이야기해서 클래스와 관련 리소스를 압축한 단순한 파일이다. 필요한 경우 이 파일을 직접 실행할 수도 있고, 다른 곳에서 라이브러리로 사용할 수도 있다.

**WAR 소개**

WAR(Web Application Archive)라는 이름에서 알 수 있듯 WAR 파일은 웹 애플리케이션 서버(WAS)에 배포할 때 사용하는 파일이다.

JAR 파일이 JVM 위에서 실행된다면, WAR는 웹 애플리케이션 서버 위에서 실행된다.

웹 애플리케이션 서버 위에서 실행되고, HTML 같은 정적 리소스와 클래스 파일을 모두 함께 포함하기 때문에 JAR와 비교해서 구조가 더 복잡하다.

그리고 WAR 구조를 지켜야 한다.


**WAR 구조**

* `WEB-INF`
  * `classes` : 실행 클래스 모음
  * `lib` : 라이브러리 모음
  * `web.xml` : 웹 서버 배치 설정 파일(생략 가능)
* `index.html` : 정적 리소스
* `WEB-INF` 폴더 하위는 자바 클래스와 라이브러리, 그리고 설정 정보가 들어가는 곳이다.
* `WEB-INF` 를 제외한 나머지 영역은 HTML, CSS 같은 정적 리소스가 사용되는 영역이다.


### WAR 배포

이렇게 생성된 WAR 파일을 톰캣 서버에 실제 배포해보자.

1. 톰캣 서버를 종료한다.  `./shutdown.sh`
2. `톰캣폴더/webapps` 하위를 모두 삭제한다. 
3. 빌드된 `server-0.0.1-SNAPSHOT.war` 를 복사한다. 
4. `톰캣폴더/webapps` 하위에 붙여넣는다.
   * `톰캣폴더/webapps/server-0.0.1-SNAPSHOT.war`
5. 이름을 변경한다. 
   * `톰캣폴더/webapps/ROOT.war`
6. 톰캣 서버를 실행한다. `./startup.sh`

**주의!**

`ROOT.war` 에서 `ROOT` 는 대문자를 사용해야 한다.

실행해보면 `index.html` 정적 파일과 `/test` 로 만들어둔 `TestServlet` 모두 잘 동작하는 것을 확인할 수 있다. 

**참고**

진행이 잘 되지 않으면 `톰캣폴더/logs/catalina.out` 로그를 꼭 확인해보자.

실제 서버에서는 이렇게 사용하면 되지만, 개발 단계에서는 `war` 파일을 만들고, 이것을 서버에 복사해서 배포하는 과정이 너무 번잡하다.

인텔리J나 이클립스 같은 IDE는 이 부분을 편리하게 자동화해준다.

## 인텔리제이 톰켓 설정

## 서블릿 컨테이너 초기화 1

WAS를 실행하는 시점에 필요한 초기화 작업들이 있다. 

서비스에 필요한 필터와 서블릿을 등록하고, 여기에 스프링을 사용한다면 스프링 컨테이너를 만들고, 서블릿과 스프링을 연결하는 디스페처 서블릿도 등록해야 한다.

WAS가 제공하는 초기화 기능을 사용하면, WAS 실행 시점에 이러한 초기화 과정을 진행할 수 있다.

과거에는 `web.xml` 을 사용해서 초기화했지만, 지금은 서블릿 스펙에서 자바 코드를 사용한 초기화도 지원 한다.

### **서블릿 컨테이너와 스프링 컨테이너**

지금부터 서블릿 컨테이너의 초기화 기능을 알아보고 이어서 이 초기화 기능을 활용해 스프링 만들고 연결해보자.

 
### 서블릿 컨테이너 초기화 개발

서블릿은 `ServletContainerInitializer` 라는 초기화 인터페이스를 제공한다. 이름 그대로 서블릿 컨테이너를 초기화 하는 기능을 제공한다.

서블릿 컨테이너는 실행 시점에 초기화 메서드인 `onStartup()` 을 호출해준다. 

여기서 애플리케이션에 필요한 기능 들을 초기화 하거나 등록할 수 있다.


**ServletContainerInitializer** 

```java
public interface ServletContainerInitializer {
    public void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException;
}
```

* `Set<Class<?>> c` : 조금 더 유연한 초기화를 기능을 제공한다. `@HandlesTypes` 애노테이션과 함께 사용한다. 이후에 코드로 설명한다.

* `ServletContext ctx` : 서블릿 컨테이너 자체의 기능을 제공한다. 이 객체를 통해 필터나 서블릿을 등록할 수 있다.

방금 본 서블릿 컨테이너 초기화 인터페이스를 간단히 구현해서 실제 동작하는지 확인해보자.


```java
public class MyContainerInitV1 implements ServletContainerInitializer {

    @Override
    public void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException {
        System.out.println("MyContainerInitV1.onStartup");
        System.out.println("MyContainerInitV1 c= " + c);
        System.out.println("MyContainerInitV1 ctx= " + ctx);
    }
}
```

이것이 끝이 아니다. 추가로 WAS에게 실행할 초기화 클래스를 알려줘야 한다. 다음 경로에 파일을 생성하자

`resources/META-INF/services/jakarta.servlet.ServletContainerInitializer`

```
hello.container.MyContainerInitV1
```

이 파일에 방금 만든 `MyContainerInitV1` 클래스를 패키지 경로를 포함해서 지정해주었다. 

이렇게 하면 WAS를 실행할 때 해당 클래스를 초기화 클래스로 인식하고 로딩 시점에 실행한다.

**주의!**

경로와 파일 이름을 주의해서 작성해야한다.

`META-INF` 는 대문자이다.

`services` 는 마지막에 `s` 가 들어간다.

파일 이름은 `jakarta.servlet.ServletContainerInitializer` 이다.

WAS를 실행해보자. 

**실행 결과 로그**

```shell
MyContainerInitV1.onStartup
MyContainerInitV1 c= null
MyContainerInitV1 ctx= org.apache.catalina.core.ApplicationContextFacade@1baffe33
```

WAS를 실행할 때 해당 초기화 클래스가 실행된 것을 확인할 수 있다.



