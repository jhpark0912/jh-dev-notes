# Mac OS에서 kerberos 로그인 이슈

## 문제 상황

로그인 서비스의 단계를 jwt > kerberos > ldap 순서로 적용하여 kerberos 인증 실패하는 경우 ldap 로그인이 가능하도록 처리했습니다.
그런데 배포 후에 `Mac OS`를 사용하는 유저는 단순 401 에러가 나타나고 로그인 창이 더이상 나타나지 않는 현상이 발생하였습니다.

## 문제 원인

`Mac OS`에서는 Windows AD 로그인 방식인 Kerberos를 지원하지 않아 발생한 문제였습니다.

기존 Windows OS에서는 Kerberos 로그인 시도를 위해 헤더에 `Negotiate` 전달하면, 
Kerberos 티켓을 발급하여 로그인 시도를 하거나 티켓이 없는 경우 ldap 로그인으로 전환되었습니다.

하지만, `Mac OS` 에는 Kerberos 라는 기능 자체가 없다보니 티켓 요청에 대한 판단을 브라우저에서 처리하지 못해 다음 단계로 넘어가지 못했습니다.

기존 코드는

```java

// EntryPoint: 인증 흐름 분기
@Bean
public AuthenticationEntryPoint delegatingEntryPoint() {
    return (request, response, authException) -> {
        String authHeader = request.getHeader("Authorization");
        if (StringUtils.isBlank(authHeader)) {
            log.debug("[ENTRYPOINT] Authorization 없음 → Negotiate 헤더 응답");
            response.addHeader("WWW-Authenticate", "Negotiate");
        } else if (authHeader.startsWith("Negotiate")) {
            log.debug("[ENTRYPOINT] Negotiate 인증 실패 → Basic 헤더 응답 (fallback)");
            response.addHeader("WWW-Authenticate", "Basic realm=\"FasooRealm\"");
        } else {
            log.debug("[AUTH_HEADER] {}", authHeader);
            log.debug("[ENTRYPOINT] 기타 인증 흐름 → Basic 헤더 응답");
            response.addHeader("WWW-Authenticate", "Basic realm=\"FasooRealm\"");
        }

        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
    };
}
```

위와 같은 형태로 `Request`의 헤더 내의 `Authorization`을 기준으로 판단해 로그인 요청을 3단계로 처리하였습니다.
이는 로그인 실패 시 처리되는 요소로 로그인 과정은 앞서 서술한 것처럼

1. `jwt` 인증
2. `kerberos` 인증
3. `ldap` 인증

순서로 이루어지도록 유도했습니다.

## 개선 사항

해당 문제는 `Mac OS`라는 특수한 환경에서는 kerberos 인증이 되지 않는 이슈이므로 요청이 `Mac OS`에서 왔는지를 판단하도록 수정하였습니다.

기존 코드에서 헤더의 `User-Agent` 값을 추가로 판단하여, Windows 에서 오는 요청이 아닌 경우 바로 `Basic form`을 통한 로그인을 유도하도록 수정하였습니다.

```java
// EntryPoint: 인증 흐름 분기
@Bean
public AuthenticationEntryPoint delegatingEntryPoint() {
    return (request, response, authException) -> {
        String authHeader = request.getHeader("Authorization");
        String userAgent = request.getHeader("User-Agent");
        if (StringUtils.isBlank(authHeader)) {
            if (userAgent != null && userAgent.toLowerCase().contains("windows nt")) {
                log.debug("[ENTRYPOINT] Windows OS User-Agent 감지, Authorization 없음 → Negotiate 헤더 응답 (Kerberos 유도)");
                response.addHeader("WWW-Authenticate", "Negotiate");
            } else {
                // windows가 아니거나 User-Agent가 없는 경우 → 기존 Kerberos 유도
                log.debug("[ENTRYPOINT] Non-Windows OS User-Agent 감지, Authorization 없음 → Basic 헤더 응답 (Kerberos 스킵)");
                response.addHeader("WWW-Authenticate", "Basic realm=\"FasooRealm\"");
            }
        } else if (authHeader.startsWith("Negotiate")) {
            log.debug("[ENTRYPOINT] Negotiate 인증 실패 → Basic 헤더 응답 (fallback)");
            response.addHeader("WWW-Authenticate", "Basic realm=\"FasooRealm\"");
        } else {
            log.debug("[AUTH_HEADER] {}", authHeader);
            log.debug("[ENTRYPOINT] 기타 인증 흐름 → Basic 헤더 응답");
            response.addHeader("WWW-Authenticate", "Basic realm=\"FasooRealm\"");
        }

        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
    };
}
```

## 느낀점

> 모든 유저가 로그인 가능하도록 하려면 다양한 환경을 고려해야 한다는 것을 놓친 부분 이었습니다.
> kerberos가 windows 로그인 기능인 것을 알았다면, 자연스럽게 `Mac OS`에서는 어떻게 진행될지를 판단해야했고 이를 대처해야했습니다.
> 그렇기 때문에 앞으로는 특정 OS 혹은 특정 환경에서 구현되는 것에 대해서는 다른 환경에서의 처리방안을 마련하고 개발을 진행하려 합니다.
