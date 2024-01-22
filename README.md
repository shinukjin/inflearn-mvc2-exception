## 예외처리, 오류페이지

### 예외처리(Exception)
- `Servlet`은 `Exception`, `response.sendError(http상태코드, 메시지)` 방식으로 예외처리 지원
- Exception 시 `was < filter < servlet < interceptor < controller(Exception 발생)`과 같이 전달
- sendError 흐름 : `WAS(sendError 호출 기록 확인) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러 (response.sendError())`
- 서블릿 오류페이지 요청 흐름  
`1. WAS '/error-page/500' 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500)-> View`  
`2.  WAS '/error-page/500' 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> View`
- was까지 전파 후 was는 오류페이지 경로를 찾아 내부에서 오류 페이지 호출, 이때 오류페이지 경로로 필터~컨트롤러를 다시 호출한다.

### 서블릿 예외처리-필터
- 예외발생 시, 오류페이지를 출력하기 위해 was 내부에서 필터 서블릿 인터셉터를 다시 호출되는 것은 비효율적으로 클라이언트 발생인지, 오류페이지 출력 위한 내부요청인지 구분할수있어야함.
- `DispatcherType`으로 해결할수있다.
```
# DispatcherType 종류
javax.servlet.DispatcherType
public enum DispatcherType {
    FORWARD, // 서블릿에서 다른 서블릿이나 jsp 호출할 때
    INCLUDE, //서블릿에서 다른 서블릿이나 jsp의 결과 포함
    REQUEST, // 클라이언트요청
    ASYNC,   // 비동기 호출
    ERROR    // 오류 요청
}
```
```
DispatcherType 필터 적용
@Bean
public FilterRegistrationBean logFilter() {
    FilterRegistrationBean<Filter> filterRegistrationBean = new FilterRegistrationBean<>();
    filterRegistrationBean.setFilter(new LogFilter());
    filterRegistrationBean.setOrder(1);
    filterRegistrationBean.addUrlPatterns("/*");
    filterRegistrationBean.setDispatcherTypes(DispatcherType.REQUEST,DispatcherType.ERROR); <==== DispatcherType 적용부분
    return filterRegistrationBean;
}
```
- 전체 흐름
```
1. WAS(/error-ex, dispatchType=REQUEST) -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러
2. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
3. WAS 오류 페이지 확인
4. WAS(/error-page/500, dispatchType=ERROR) -> 필터(x) -> 서블릿 -> 인터셉터(x) -> 컨트
   롤러(/error-page/500) -> View
```

### 예외처리 - 인터셉터
- 인터셉터는 다음과 같이 요청 경로에 따라서 추가하거나 제외하기 쉽게 되어 있기 때문에, 이러한 설정을 사용해서 오류 페이지 경로를 excludePathPatterns 를 사용해 빼줌.
```
@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(new LogInterceptor())
    .order(1)
    .addPathPatterns("/**")
    .excludePathPatterns(
    "/css/**", "/*.ico"
    , "/error", "/error-page/**" //오류 페이지 경로
    );
}
```
- 전체 흐름
```
1. WAS(/error-ex, dispatchType=REQUEST) -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러
2. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
3. WAS 오류 페이지 확인
4. WAS(/error-page/500, dispatchType=ERROR) -> 필터(x) -> 서블릿 -> 인터셉터(x) -> 컨트
롤러(/error-page/500) -> View
```

### 오류페이지
- `BasicErrorController` 컨트롤러는 다음 정보를 model에 담아서 뷰에 전달
```* timestamp: Fri Feb 05 00:00:00 KST 2021
* status: 400
* error: Bad Request
* exception: org.springframework.validation.BindException
* trace: 예외 trace
* message: Validation failed for object='data'. Error count: 1
* errors: Errors(BindingResult)
* path: 클라이언트 요청 경로 (`/hello`)
```
- application.properties에서 오류정보 전달여부 선택 가능
```
application.properties
server.error.include-exception=true
server.error.include-message=on_param
server.error.include-stacktrace=on_param
server.error.include-binding-errors=on_param

# never : 사용하지 않음
# always :항상 사용
# on_param : 파라미터가 있을 때 사용
```