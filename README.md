# 서블릿 예외 처리 -시작
### 서블릿은 다음 2가지 방식으로 예외처리
1. Exception(예외)
2. response.sendError(HTTP 상태 코드, 오류 메세지)

## Exception( 예외)
### 자바 직접 실행
자바의 메인 메서드를 직접 실행하는 경우 main 이라는 이름의 쓰레드가 실행
실행 도중에 예외를 잡지 못하고 처음 실행한 main() 메서드를 넘어서 예외가 던져지면, 예외 정보를 남기고 해당 쓰레드는 종료

### 웹 애플리케이션
웹 애플리케이션은 사용자 요청별로 별도의 쓰레드가 할당되고 , 서블릿 컨테이너 안에서 실행
애플리케이션에서 예외가 발생했는데 어디선가 try~catch로 예외를 잡아서 처리하면 아무런 문제가 없다.
그런데 만약에 애플리케이션에서 예외를 잡지 못하고, 서블릿 밖으로 까지 예외가 전달된다면 동작하는 방법을 알아야한다.
WAS(여기까지 전파)<-필터<-서블릿 <-인터셉터<-컨트롤러(예외발생)
톰캣같은 WAS까지 예외가 전달된다. 

톰캣이 기본으로 제공하는 오류화면은 > HTTP Status 500- Internal Server Error이다.

웹 브라우저에서 개발자 모드로 확인해보면 HTTP 상태 코드가 500으로 보임
Exception의 경우 서버 내부에서 처리할 수 없는 오류가 발생한 것으로 생각해서 HTTP 상태 코드를 500을 반환 
아무 사이트를 호출하면 톰캣이 기본으로 제공하는 404오류 화면을 볼 수 있다.

### response.sendError(HTTP 상태코드, 오류메세지)
오류가 발생했을 때 HttpServletResponse 가 제공하는 sendError 라는 메서드를 사용해도 된다. 
이것을 호출한다고 당장 예외가 발생하는 것은 아니지만, 서블릿 컨테이너에게 오류가 발생했다는 점을
전달할 수 있다.
이 메서드를 사용하면 HTTP 상태 코드와 오류 메시지도 추가할 수 있다.

* response.sendError(HTTP 상태 코드)
* response.sendError(HTTP 상태 코드, 오류 메시지)

#### sendError흐름
> WAS(sendError 호출 기록 확인) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(response.sendError())  
response.sendError() 를 호출하면 response 내부에는 오류가 발생했다는 상태를 저장해둔다.
그리고 서블릿 컨테이너는 고객에게 응답 전에 response 에 sendError() 가 호출되었는지 확인한다. 
그리고 호출되었다면 설정한 오류 코드에 맞추어 기본 오류 페이지를 보여준다

## 서블릿 예외 처리 - 오류 화면 제공
서블릿은 Exception (예외)가 발생해서 서블릿 밖으로 전달되거나 또는 response.sendError() 가 호출
되었을 때 각각의 상황에 맞춘 오류 처리 기능을 제공한다.
이 기능을 사용하면 친절한 오류 처리 화면을 준비해서 고객에게 보여줄 수 있다.

* response.sendError(404) : errorPage404 호출
* response.sendError(500) : errorPage500 호출
* RuntimeException 또는 그 자식 타입의 예외: errorPageEx 호출

오류 페이지는 예외를 다룰 때 해당 예외와 그 자식 타입의 오류를 함께 처리한다. 예를 들어서 위의 경우
RuntimeException 은 물론이고 RuntimeException 의 자식도 함께 처리한다.



## 서블릿 예외 처리 -오류 페이지 작동 원리
서블릿은 Exception (예외)가 발생해서 서블릿 밖으로 전달되거나 또는 response.sendError() 가 호출 되었을 때 설정된 오류 페이지를 찾는다.

#### 예외 발생 흐름
> WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)

#### sendError흐름
> WAS(sendError 호출 기록 확인) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(response.sendError())
WAS는 해당 예외를 처리하는 오류 페이지 정보를 확인한다.-> new ErrorPage(RuntimeException.class, "/error-page/500") 등록이 되어있다.
예를 들어서 RuntimeException 예외가 WAS까지 전달되면, WAS는 오류 페이지 정보를 확인한다. 
확인해보니 RuntimeException 의 오류 페이지로 /error-page/500 이 지정되어 있다. WAS는 오류 페이지를 출력하기 위해 /error-page/500 를 다시 요청한다

##### 오류 페이지 요청 흐름
> WAS `/error-page/500` 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> View

##### 예외 발생과 오류 페이지 요청 흐름
> 1. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)  
> 2. WAS `/error-page/500` 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/errorpage/500) -> View
#### 중요한 점은 웹 브라우저(클라이언트)는 서버 내부에서 이런 일이 일어나는지 전혀 모른다는 점이다. 오직 서버 내부에서 오류 페이지를 찾기 위해 추가적인 호출을 한다.
1. 예외가 발생해서 WAS까지 전파
2. WAS는 오류 페이지 경로를 찾아서 내부에서 오류 페이지를 호출, 이떄 오류 페이지 경로로 필터,서블릿,인터셉터,컨트롤러가 모두 다시 호출

### 오류 정보 추가 
WAS는 오류 페이지를 단순히 다시 요청만 하는 것이 아니라, 오류 정보를 request의 attribute에 추가해서 넘겨준다.

#### request.attribute에 서버가 담아준 정보
* javax.servlet.error.exception : 예외
* javax.servlet.error.exception_type : 예외 타입
* javax.servlet.error.message : 오류 메시지
* javax.servlet.error.request_uri : 클라이언트 요청 URI
* javax.servlet.error.servlet_name : 오류가 발생한 서블릿 이름
* javax.servlet.error.status_code : HTTP 상태 코드
