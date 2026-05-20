# 인증/회원 도메인 구조 및 로직

## 1. 패키지 구조

```
com.aidoctor.member
├── controller
│   └── MemberController.java
├── service
│   ├── MemberService.java          (interface)
│   └── MemberServiceImpl.java
├── repository
│   ├── UserRepository.java         (interface, JPA)
│   └── RefreshTokenRepository.java (interface, JPA)
├── entity
│   ├── User.java
│   ├── RefreshToken.java
│   └── UserStatus.java             (enum: ACTIVE, WITHDRAWN)
├── dto
│   ├── request
│   │   ├── SignupRequest.java
│   │   └── LoginRequest.java
│   └── response
│       ├── SignupResponse.java
│       ├── LoginResponse.java
│       └── TokenRefreshResponse.java
└── security
    └── JwtTokenProvider.java
```

## 2. API 명세

| 기능 | Method | URL | 인증 |
|------|--------|-----|------|
| 회원가입 | POST | /api/member/signup | X |
| 로그인 | POST | /api/member/login | X |
| 로그아웃 | POST | /api/member/logout | O (Bearer Token) |
| 토큰 재발급 | POST | /api/member/token/refresh | O (Cookie) |

## 3. 핵심 로직

### 3-1. 회원가입 (signup)

1. 클라이언트가 이메일, 비밀번호, 이름, 아이디, 전화번호, 생년월일, 성별을 전송한다.
2. 이메일 중복 여부를 확인한다. 중복이면 409 응답을 반환한다.
3. 아이디(loginId) 중복 여부를 확인한다. 중복이면 409 응답을 반환한다.
4. 비밀번호를 BCrypt로 해싱하여 암호화한다.
5. User 엔티티를 생성하고 DB에 저장한다. 초기 상태는 ACTIVE이다.
6. 201 응답과 함께 생성된 사용자 ID를 반환한다.

### 3-2. 로그인 (login)

1. 클라이언트가 이메일과 비밀번호를 전송한다.
2. 이메일로 사용자를 조회한다. 존재하지 않으면 401 응답을 반환한다.
3. BCrypt로 비밀번호 일치 여부를 검증한다. 불일치하면 401 응답을 반환한다.
4. JwtTokenProvider를 통해 AccessToken과 RefreshToken을 생성한다.
5. RefreshToken의 해시값을 DB에 저장한다.
6. AccessToken은 응답 Body에, RefreshToken은 HttpOnly Cookie에 담아 반환한다.
7. 클라이언트는 AccessToken과 이메일을 localStorage에 저장하고 MainPage로 이동한다.

### 3-3. 로그아웃 (logout)

1. 클라이언트가 Authorization 헤더에 AccessToken을 담아 요청한다.
2. AccessToken에서 userId를 추출한다.
3. 해당 userId의 RefreshToken을 DB에서 삭제한다.
4. 응답에서 RefreshToken 쿠키를 만료 처리하여 삭제한다.
5. 클라이언트는 localStorage의 AccessToken과 이메일을 제거하고 메인 화면으로 이동한다.

### 3-4. 토큰 재발급 (refresh)

1. 클라이언트가 Cookie에 담긴 RefreshToken을 전송한다.
2. RefreshToken의 해시값으로 DB를 조회한다. 존재하지 않으면 401 응답을 반환한다.
3. 토큰의 만료 시간(expiresAt)을 확인한다. 만료되었으면 401 응답을 반환한다.
4. JwtTokenProvider를 통해 새로운 AccessToken을 생성한다.
5. 새 AccessToken을 응답 Body에 담아 반환한다.

## 4. 엔티티 설계

### User

| 필드 | 타입 | 설명 |
|------|------|------|
| userId | Long (PK) | 사용자 고유 ID |
| loginId | String (UNIQUE) | 로그인 아이디 |
| password | String | BCrypt 해싱된 비밀번호 |
| name | String | 이름 |
| email | String (UNIQUE) | 이메일 (로그인에 사용) |
| phone | String | 전화번호 (선택) |
| birthDate | LocalDate | 생년월일 |
| gender | Character | 성별 |
| status | UserStatus | 계정 상태 (ACTIVE / WITHDRAWN) |
| createdAt | LocalDateTime | 생성일시 |
| updatedAt | LocalDateTime | 수정일시 |

### RefreshToken

| 필드 | 타입 | 설명 |
|------|------|------|
| tokenId | Long (PK) | 토큰 고유 ID |
| userId | Long (FK) | 사용자 ID |
| tokenHash | String (UNIQUE) | RefreshToken 해시값 |
| expiresAt | LocalDateTime | 만료 시간 |
| revoked | boolean | 폐기 여부 |
| createdAt | LocalDateTime | 생성일시 |

## 5. 인증 흐름 요약

```
[회원가입]
Client → POST /api/member/signup → 중복 검사 → BCrypt 암호화 → DB 저장 → 201

[로그인]
Client → POST /api/member/login → 이메일 조회 → 비밀번호 검증
→ AccessToken + RefreshToken 생성 → Body + HttpOnly Cookie 응답

[인증된 요청]
Client → Authorization: Bearer {AccessToken} → JwtTokenProvider 검증 → API 처리

[토큰 만료 시]
Client → POST /api/member/token/refresh (Cookie) → DB 조회 → 만료 확인
→ 새 AccessToken 발급

[로그아웃]
Client → POST /api/member/logout → DB RefreshToken 삭제 → Cookie 만료 처리
```
