# jwt 만료 토큰 초기화 이슈

## 문제 상황

로그인 과정에서 설정한 Jwt 토큰이 만료된 경우 Kerberos 혹은 Ldap 로그인 요청을 하도록 설정하였고 이에 따라서 웹페이지에 로그인 창이 생성 되었습니다.
그런데 ID/PW를 통해 로그인을 시도해도 계속해서 로그인 창이 뜨는 이슈가 발생해 로그인이 되지 않고 계속 반복되는 이슈가 발생하였습니다.

## 문제 원인

`Header`에 `Authorization`으로 설정된 토큰이 만료가 되더라도 전송이 되도록 설정이 되어 있는 상태였습니다.

기존 코드

```java
@Override
protected void doFilterInternal(HttpServletRequest request,
                              HttpServletResponse response,
                              FilterChain filterChain) throws ServletException, IOException {

  String authHeader = request.getHeader("Authorization");

  if (authHeader != null && authHeader.startsWith("Bearer ")) {
    String jwt = authHeader.substring(7);
    if (jwtUtils.validateToken(jwt)) {
        String email = jwtUtils.extractEmail(jwt);
        UserType authType = jwtUtils.extractUserType(jwt);
        CustomUserDetails userDetails = new CustomUserDetails(email, jwtUtils.extractSubject(jwt), authType.authName());
        UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
        SecurityContextHolder.getContext().setAuthentication(authentication);
    } 
    filterChain.doFilter(request, response);
  } 
}
```

이로 인해, Basic form으로 로그인 유도한 정보가 `Authorization`에 등록되어 전달 되었지만,
해당 헤더 값을 먼저 확인을 하게 되면서, Spring의 `BasicAuthenticationFilter.class` 가 아예 무시하는 상황이 된 것이었습니다.

## 해결 방안

기존에 jwt 토큰이 들어왔지만, 해당 토큰에 대해서 유효성 검사한 것에 대해서 처리하지 않던 부분을 `401 - UNAUTHORIZED` 를 전송하도록 조치하였습니다.

수정 코드

```java
@Override
protected void doFilterInternal(HttpServletRequest request,
                                HttpServletResponse response,
                                FilterChain filterChain) throws ServletException, IOException {

    String authHeader = request.getHeader("Authorization");

    if (authHeader != null && authHeader.startsWith("Bearer ")) {
        String jwt = authHeader.substring(7);
        try {
            if (jwtUtils.validateToken(jwt)) {
                String email = jwtUtils.extractEmail(jwt);
                UserType authType = jwtUtils.extractUserType(jwt);
                CustomUserDetails userDetails = new CustomUserDetails(email, jwtUtils.extractSubject(jwt), authType.authName());
                UsernamePasswordAuthenticationToken authentication =
                        new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                SecurityContextHolder.getContext().setAuthentication(authentication);
            } else {
                throw new SecurityException("Invalid JWT Token");
            }
        } catch (Exception e) {
            log.warn("JWT 인증 처리 중 오류 발생: {}", e.getMessage());

            SecurityContextHolder.clearContext();

            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.setCharacterEncoding("UTF-8");
            response.getWriter().write("{\"error\": \"Unauthorized\", \"message\": \"" + e.getMessage() + "\"}");

            return;
        }

    }

    filterChain.doFilter(request, response);
}
```

에러를 전달 받은 클라이언트는 저장한 토큰을 삭제하도록 처리하며, 다시 한번 페이지를 reload 하도록 유도하였습니다.
이를 통해 저장된 토큰이 삭제되고, 신규 로그인으로 인식이 되며 Kerberos, Ldap 순으로 로그인 유도하도록 처리하였습니다.

## 느낀점

> 만료된 토큰으로 로그인이 되는 것을 확인을 안하고 배포한 서비스에서 로그인 에러가 나타난 것은 굉장히 위험한 상황이었습니다.
> 항상 로그인 시에는 안되는 상황에 대해서 더 검토해보고 로그인 처리가 가능하도록 해야겠습니다.
> 또한, http 요청에서 헤더 값이 겹치는 경우에 대해서 서버에서는 하나만 인식하여 처리가 되고 이를 인지하여 처리해야 합니다.
