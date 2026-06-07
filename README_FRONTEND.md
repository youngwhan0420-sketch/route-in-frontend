# 🖥 Route-In Frontend

Route-In 서비스의 프론트엔드입니다.

React 기반으로 사용자 화면을 구성하고, Axios와 React Query를 통해 백엔드 REST API와 연동합니다.  
게시글, 러닝코스, 루틴, 팔로우, 알림, 채팅, 인바디, 출석 기능을 사용자 화면에서 사용할 수 있도록 구현했습니다.

[🔗 Backend 레포](https://github.com/Koreait-Triple-Stack/route_in_backend.git)　|　[🔗 배포 주소](https://routein.store)

---

## 🛠 Tech Stack

| 구분 | 기술 |
|---|---|
| Library | React |
| Build Tool | Vite |
| Routing | React Router |
| Server State | TanStack Query / React Query |
| Client State | Zustand |
| UI | MUI |
| HTTP Client | Axios |
| 지도 | Kakao Map API |
| 실시간 통신 | WebSocket · STOMP |

---

## 📌 본인 담당 기능

- 출석 시스템 프론트엔드 연동
- REST API 연동 구조 설계 및 구현
- Axios Instance 기반 공통 API 요청 처리
- React Query 기반 서버 상태 조회 및 갱신
- Zustand 기반 로그인 사용자 상태 관리
- 출석 달력 UI 및 팝업 상태 처리
- 게시글, 팔로우, 추천, 코스, 알림 등 API 연동

---

# 1. Frontend 전체 구조

Route-In 프론트엔드는 Page, Component, API Service, 전역 상태, React Query를 중심으로 구성됩니다.

```text
사용자 액션
→ Page / Component 이벤트 발생
→ API Service 함수 호출
→ Axios Instance로 백엔드 요청
→ Spring Boot API 응답 수신
→ React Query 또는 Zustand 상태 갱신
→ 화면 재렌더링
```

---

## 1-1. 계층별 역할

| 구분 | 역할 |
|---|---|
| Page | 하나의 화면 단위 |
| Component | 재사용 가능한 UI 조각 |
| API | 실제 Axios 요청 함수 |
| Service | API 응답 처리 및 에러 처리 |
| React Query | 서버 데이터 조회, 캐싱, 재조회 |
| Zustand | 로그인 사용자 등 클라이언트 전역 상태 관리 |

---

# 2. REST API 연동 구조

## 2-1. Axios Instance 로직

프론트엔드에서는 Axios Instance를 사용해 모든 API 요청에 공통 설정을 적용했습니다.

```javascript
export const instance = axios.create({
    baseURL: import.meta.env.VITE_API_BASE_URL ?? "",
    timeout: 15000,
    withCredentials: false,
});
```

AccessToken이 있으면 요청 헤더에 자동으로 추가합니다.

```javascript
config.headers.Authorization = `Bearer ${accessToken}`;
```

401 응답이 오면 AccessToken을 제거해 인증 상태를 정리합니다.

```javascript
if (status === 401) {
    localStorage.removeItem("AccessToken");
}
```

### 로직 설명

```text
API 요청 발생
→ localStorage에서 AccessToken 조회
→ AccessToken이 있으면 Authorization Header에 Bearer Token 추가
→ 백엔드 JWT 인증 필터에서 토큰 검증
→ 인증 성공 시 Controller에서 사용자 정보 사용 가능
```

---

## 2-2. API / Service 분리 구조

출석 API를 예로 들면 프론트엔드는 실제 HTTP 요청 함수와 응답 처리 함수를 분리했습니다.

```text
attendanceApi.js
→ 실제 Axios 요청 담당

attendanceService.js
→ 응답 status 검사
→ 실패 시 Error 발생
→ 성공 시 result.data 반환
```

### 예시

```javascript
export const getAttendanceMonthDatesRequest = (ym) => {
  return instance.get("/attendance/month", { params: { ym } });
};

export const updateAttendancePopupShownTodayRequest = () => {
  return instance.patch("/attendance/popup/shown");
};
```

```javascript
export const getAttendanceMonthDates = async ({ ym }) => {
  const result = await getAttendanceMonthDatesRequest(ym);
  if (result.data.status !== "success") throw new Error(result.data.message);
  return result.data;
};
```

### 설계 이유

API 요청과 응답 검증 로직을 분리하면 Page나 Component에서는 세부 HTTP 처리 방식을 몰라도 됩니다.

```text
Component
→ service 함수만 호출

Service
→ API 호출 + 응답 검증

API
→ 실제 Axios 요청
```

---

# 3. 출석 시스템 프론트엔드 구현

## 3-1. 관련 Frontend 파일 구조

```text
src/apis/attendance/attendanceApi.js
src/apis/attendance/attendanceService.js
src/components/Calendar.jsx
src/pages/MainPage/MainPage.jsx
src/store/usePrincipalState.js
```

---

## 3-2. 출석 시스템 전체 프론트 로직

```text
1. 앱 실행 후 로그인 사용자 정보 조회
2. /user/account/principal API 응답으로 principal 데이터 저장
3. 백엔드에서 principal.checked 값 전달
4. MainPage에서 principal.checked가 true인지 확인
5. true이면 출석 달력 팝업 표시
6. Calendar 컴포넌트가 열리면 /attendance/month?ym=YYYY-MM 요청
7. 월별 출석 날짜 목록을 받아 달력에 표시
8. 사용자가 팝업을 닫으면 Zustand의 checked 값을 false로 변경
9. PATCH /attendance/popup/shown 요청으로 서버에 팝업 확인 상태 저장
10. 같은 날짜에 다시 접속해도 서버 기준으로 팝업 재노출 방지
```

---

## 3-3. MainPage 출석 팝업 표시 로직

`MainPage.jsx`에서는 로그인 사용자 상태의 `checked` 값을 기준으로 출석 팝업을 표시합니다.

```text
principal.checked === true
→ 출석 팝업 open

principal.checked === false
→ 출석 팝업 close
```

### 처리 흐름

```text
MainPage 렌더링
→ principal 상태 확인
→ principal.checked 값 확인
→ true이면 Calendar Dialog 표시
→ 사용자가 닫기 버튼 클릭
→ handleClose 실행
```

---

## 3-4. 출석 팝업 닫기 로직

사용자가 출석 팝업을 닫으면 프론트 상태를 먼저 닫고, 서버에 팝업 표시 완료 요청을 보냅니다.

```javascript
const handleClose = () => {
    AttendanceChecked();
    updateAttendancePopupShownToday().catch(() => {});
};
```

Zustand store에서는 현재 사용자 상태의 `checked` 값을 false로 변경합니다.

```javascript
AttendanceChecked: () =>
    set((state) =>
        state.principal
            ? { principal: { ...state.principal, checked: false } }
            : {},
    ),
```

### 내부 처리 흐름

```text
출석 팝업 닫기
→ AttendanceChecked() 실행
→ principal.checked = false
→ 팝업 즉시 닫힘
→ updateAttendancePopupShownToday() 실행
→ PATCH /attendance/popup/shown 요청
→ 서버 DB popup_shown = 1 변경
```

### 설계 이유

팝업 닫기는 사용자가 즉시 체감하는 UI 동작입니다.

그래서 프론트 상태를 먼저 변경해 팝업을 닫고, 서버에는 별도 API 요청으로 팝업 확인 상태를 저장했습니다.

---

## 3-5. 월별 출석 조회 로직

`Calendar.jsx`에서는 React Query를 사용해 월별 출석 날짜를 조회합니다.

```javascript
const { data } = useQuery({
    queryKey: ["AttendanceMonthDates", principal?.userId, ym],
    queryFn: () => getAttendanceMonthDates({ ym }),
    enabled: open && !!principal?.userId,
    staleTime: 0,
});
```

### 내부 처리 흐름

```text
Calendar open = true
→ 로그인 사용자 principal.userId 존재 확인
→ 현재 월을 YYYY-MM 형태로 변환
→ queryKey = ["AttendanceMonthDates", userId, ym]
→ getAttendanceMonthDates({ ym }) 실행
→ GET /attendance/month?ym=YYYY-MM 요청
→ 출석 날짜 배열 응답
→ 달력에 출석 날짜 표시
```

---

## 3-6. 출석 날짜 표시 로직

백엔드에서 받은 출석 날짜 배열은 `Set`으로 변환합니다.

```javascript
const markedSet = useMemo(() => new Set(data?.data ?? []), [data?.data]);
```

### 처리 흐름

```text
백엔드 응답
→ ["2026-06-01", "2026-06-02"]

프론트 변환
→ Set(["2026-06-01", "2026-06-02"])

달력 날짜 렌더링
→ day.format("YYYY-MM-DD") 값 확인
→ markedSet에 포함되어 있으면 출석 표시
```

### 설계 이유

날짜를 매번 배열에서 찾는 방식보다 `Set`으로 변환하면 포함 여부를 간단하게 확인할 수 있습니다.

---

## 3-7. React Query queryKey 설계

출석 조회의 queryKey에는 사용자 ID와 조회 월을 함께 넣었습니다.

```javascript
queryKey: ["AttendanceMonthDates", principal?.userId, ym]
```

### 설계 이유

```text
사용자가 바뀜
→ 다른 사용자의 출석 데이터를 다시 조회

조회 월이 바뀜
→ 해당 월의 출석 데이터를 다시 조회
```

이렇게 해서 이전 사용자나 이전 월의 출석 데이터가 잘못 재사용되는 문제를 방지했습니다.

---

# 4. 주요 Frontend 기능 로직

## 4-1. 게시글 목록 조회 로직

```text
BoardListPage 진입
→ React Query로 게시글 목록 요청
→ boardService에서 API 호출
→ 백엔드 응답 수신
→ PostCard 컴포넌트로 게시글 목록 렌더링
```

### 핵심 설명

게시글 목록 화면은 백엔드에서 받은 게시글 데이터를 반복해서 화면에 표시합니다.

---

## 4-2. 게시글 상세 조회 로직

```text
게시글 클릭
→ boardId를 가지고 BoardDetailPage 이동
→ useQuery로 상세 데이터 요청
→ 응답 데이터 화면 표시
→ 게시글 타입에 따라 루틴 또는 코스 영역 표시
```

### 핵심 설명

상세 페이지에서는 `boardId`를 기준으로 데이터를 가져오고, 게시글 타입에 따라 보여주는 화면이 달라집니다.

---

## 4-3. 게시글 작성 로직

```text
BoardWritePage 진입
→ 제목, 내용, 게시글 타입 입력
→ 타입에 따라 루틴 / 코스 입력 영역 표시
→ 작성 버튼 클릭
→ 요청 데이터 구성
→ addBoard API 호출
→ 성공 시 게시글 목록 또는 상세 페이지 이동
```

### 핵심 설명

프론트엔드에서는 사용자가 선택한 게시글 타입에 따라 서버로 보내는 데이터 구조를 다르게 만듭니다.

```text
일반 게시글
→ title, content, type

루틴 게시글
→ title, content, type, routines

코스 게시글
→ title, content, type, course, points
```

---

## 4-4. 추천 기능 로직

```text
추천 버튼 클릭
→ 현재 추천 여부 확인
→ 추천 추가 또는 취소 mutation 실행
→ 성공 시 게시글 상세 query 무효화
→ 추천 목록 query 무효화
→ 최신 데이터 다시 조회
```

### 핵심 설명

추천 수와 버튼 상태가 다르게 보이지 않도록, 추천 요청 성공 후 관련 데이터를 다시 조회했습니다.

```text
프론트 상태만 변경
→ 서버 데이터와 화면이 다를 수 있음

서버 요청 후 재조회
→ 추천 수와 버튼 상태가 서버 기준으로 일치
```

---

## 4-5. 팔로우 기능 로직

```text
UserDetailPage 진입
→ 프로필 사용자 정보 조회
→ FollowButton에 targetUserId 전달
→ FollowButton에서 팔로우 상태 조회
→ 버튼 클릭 시 팔로우 / 언팔로우 요청
→ 성공 후 팔로워 수와 버튼 상태 재조회
```

### 핵심 설명

팔로우 버튼은 여러 페이지에서 사용할 수 있도록 공통 컴포넌트로 분리했습니다.

```text
FollowButton
→ targetUserId 전달받음
→ 현재 팔로우 상태 조회
→ 클릭 시 팔로우 / 언팔로우 요청
→ 성공 후 관련 데이터 재조회
```

---

## 4-6. 러닝코스 작성 로직

```text
코스 작성 페이지 진입
→ 카카오맵 로드
→ 지도 클릭 시 좌표 저장
→ 좌표들을 선으로 연결
→ 총 거리 계산
→ 저장 버튼 클릭
→ 코스 정보와 좌표 배열을 서버로 전송
```

### 핵심 설명

프론트엔드에서는 사용자가 지도에서 클릭한 좌표를 배열로 관리합니다.

서버로 보낼 때는 코스 기본 정보와 좌표 목록을 함께 전달합니다.

```text
course
→ 코스 이름, 거리, 지역

points
→ 위도, 경도, 순서
```

---

# 5. 서버 상태 관리 로직

## 5-1. React Query 사용 방식

React Query는 서버 데이터를 조회하고, 등록 / 수정 / 삭제 이후 화면 데이터를 최신 상태로 유지하기 위해 사용했습니다.

```text
useQuery
→ 서버 데이터 조회

useMutation
→ 등록, 수정, 삭제 요청

invalidateQueries
→ 변경된 데이터 다시 조회
```

---

## 5-2. mutation 이후 데이터 갱신 로직

```text
사용자 액션
→ mutation 실행
→ 서버 데이터 변경
→ invalidateQueries
→ 최신 데이터 재조회
→ 화면 갱신
```

### 적용 예시

```text
추천 버튼 클릭
→ 추천 mutation 실행
→ 서버에서 추천 데이터 변경
→ 게시글 상세 query invalidate
→ 추천 목록 query invalidate
→ 최신 추천 수와 추천 여부 다시 조회
```

---

## 5-3. Zustand 전역 상태 관리

Zustand는 로그인 사용자 정보처럼 여러 화면에서 공통으로 사용하는 상태를 관리하는 데 사용했습니다.

```text
principal
→ 로그인 사용자 정보

principal.checked
→ 오늘 출석 팝업 표시 여부
```

### 출석 상태 변경 예시

```text
출석 팝업 닫기
→ AttendanceChecked() 실행
→ principal.checked = false
→ MainPage에서 팝업 닫힘
```

---

# 6. Frontend 트러블슈팅

## 6-1. 출석 팝업 멀티 디바이스 동기화 문제

### 문제

브라우저 localStorage 기준으로 팝업 여부를 관리하면 A기기에서 닫은 팝업이 B기기에서는 다시 표시될 수 있습니다.

### 원인

localStorage는 브라우저 또는 기기별 저장소이기 때문에 계정 기준 동기화가 되지 않습니다.

### 해결

프론트에서는 팝업을 닫을 때 `PATCH /attendance/popup/shown` 요청을 보내고, 백엔드는 DB의 `popup_shown` 값을 변경하도록 처리했습니다.

### 결과

팝업 표시 여부를 서버 기준으로 관리할 수 있게 되어 멀티 디바이스 환경에서도 일관성을 유지했습니다.

---

## 6-2. 출석 달력 월 변경 시 데이터 갱신 문제

### 문제

달력에서 조회 월이 바뀌었을 때 이전 월 출석 데이터가 그대로 보일 수 있습니다.

### 원인

React Query의 queryKey가 사용자와 월을 명확히 구분하지 않으면 잘못된 캐시 데이터가 재사용될 수 있습니다.

### 해결

queryKey에 `principal.userId`와 `ym`을 함께 포함했습니다.

```javascript
queryKey: ["AttendanceMonthDates", principal?.userId, ym]
```

### 결과

사용자 또는 조회 월이 바뀔 때마다 해당 월의 출석 데이터를 다시 조회할 수 있게 되었습니다.

---

## 6-3. 추천 / 팔로우 상태 불일치 문제

### 문제

추천 수나 팔로워 수가 버튼 상태와 다르게 보일 수 있었습니다.

### 원인

서버 데이터가 변경된 후 관련 데이터를 다시 조회하지 않으면 화면 상태와 DB 상태가 달라질 수 있습니다.

### 해결

mutation 성공 후 관련 query를 invalidate하여 최신 데이터를 다시 조회했습니다.

### 결과

추천 수, 추천 여부, 팔로워 수, 팔로우 버튼 상태가 서버 기준으로 일치했습니다.

---

# 7. Frontend 핵심 포인트

Route-In 프론트엔드에서 중요한 부분은 사용자의 행동을 API 요청으로 연결하고, 서버 응답을 기준으로 화면 상태를 갱신하는 것이었습니다.

특히 출석, 추천, 팔로우처럼 데이터가 자주 바뀌는 기능은 프론트 상태만 임시로 변경하면 서버 데이터와 화면 상태가 달라질 수 있습니다.

그래서 React Query를 사용해 서버 데이터를 조회하고, mutation 이후 관련 데이터를 다시 조회하도록 처리했습니다.

출석 시스템에서는 Zustand로 팝업 표시 상태를 즉시 닫고, 서버에는 `PATCH /attendance/popup/shown` 요청을 보내 DB 기준 상태를 저장했습니다.

이 구조를 통해 사용자 경험과 데이터 일관성을 함께 유지할 수 있었습니다.
