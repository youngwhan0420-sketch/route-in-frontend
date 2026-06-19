# 🖥 Route-In-AI-Coach Frontend

Route-In-AI-Coach 서비스의 프론트엔드입니다.

React 기반으로 사용자 화면을 구성하고, Axios와 React Query를 통해 백엔드 REST API와 연동합니다.
게시글, 러닝코스, 루틴, 팔로우, 알림, 채팅, 인바디, 출석, AI 추천, AI 코치 질문 / 답변 기능을 사용자 화면에서 사용할 수 있도록 구현했습니다.

---

## 🛠 Tech Stack

| 구분           | 기술                           |
| ------------ | ---------------------------- |
| Library      | React                        |
| Build Tool   | Vite                         |
| Routing      | React Router                 |
| Server State | TanStack Query / React Query |
| Client State | Zustand                      |
| UI           | MUI                          |
| HTTP Client  | Axios                        |
| 지도           | Kakao Map API                |
| 실시간 통신       | WebSocket · STOMP            |

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

| 구분          | 역할                       |
| ----------- | ------------------------ |
| Page        | 하나의 화면 단위                |
| Component   | 재사용 가능한 UI 조각            |
| API         | 실제 Axios 요청 함수           |
| Service     | API 응답 처리 및 에러 처리        |
| React Query | 서버 데이터 조회, 캐싱, 재조회       |
| Zustand     | 로그인 사용자 등 클라이언트 전역 상태 관리 |

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

## 2-3. AI API / Service 분리 구조

AI 기능도 동일하게 API 요청 함수와 Service 함수를 분리했습니다.

```text
aiRecommendApi.js
→ AI 관련 실제 Axios 요청 담당

aiRecommendService.js
→ 응답 status 검사
→ 실패 시 Error 발생
→ 성공 시 result.data 반환
```

### AI API 예시

```javascript
export const getTodayRecommendationRequest = async (userId) => {
    return await instance.get(`/ai/recommend/${userId}`);
};

export const getAIRespRequest = async (data) => {
    return await instance.post("/ai/question", data);
};

export const getAIChatListByUserIdRequest = async (userId) => {
    return await instance.get(`/ai/chatList/${userId}`);
};

export const removeAiChatRequest = async (userId) => {
    return await instance.delete(`/ai/chatList/${userId}`);
};
```

### AI Service 예시

```javascript
export const getTodayRecommendation = async (userId) => {
    const result = await getTodayRecommendationRequest(userId);
    if (result.data.status !== "success") throw new Error(result.data.message);
    return result.data;
};

export const getAIResp = async (data) => {
    const result = await getAIRespRequest(data);
    if (result.data.status !== "success") throw new Error(result.data.message);
    return result.data;
};
```

### 설계 이유

AI 기능은 API 호출 실패 가능성이 상대적으로 높습니다.

외부 LLM API 응답 지연, 백엔드 처리 오류, 네트워크 문제 등이 발생할 수 있기 때문에 Service 계층에서 공통 응답 검증을 처리하고, Component에서는 성공 / 실패 상태에 따른 UI 처리에 집중하도록 구성했습니다.

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

## 4-7. AI 추천 카드 화면 로직

Route-In-AI-Coach의 메인 페이지에서는 사용자의 운동 데이터를 기반으로 생성된 오늘의 AI 추천을 카드 형태로 보여줍니다.

관련 파일 구조는 다음과 같습니다.

```text
src/apis/aiRecommend/aiRecommendApi.js
src/apis/aiRecommend/aiRecommendService.js
src/pages/MainPage/AIRecommend.jsx
src/pages/MainPage/AIChat.jsx
src/components/Recommendation.jsx
```

### 전체 처리 흐름

```text
MainPage 렌더링
→ AIRecommend 컴포넌트 호출
→ React Query로 /ai/recommend/{userId} 요청
→ 백엔드에서 오늘의 AI 추천 데이터 조회 또는 생성
→ AI 추천 응답 수신
→ 루틴 추천 카드와 러닝 추천 카드로 분리 렌더링
```

### 핵심 코드

```javascript
const {
    data: recommendation,
    isLoading,
    error,
} = useQuery({
    queryKey: ["getTodayRecommendation", userId],
    queryFn: () => getTodayRecommendation(userId),
    enabled: !!userId,
});
```

### 로직 설명

```text
queryKey: ["getTodayRecommendation", userId]
→ 사용자별 오늘의 추천 데이터를 구분하기 위한 캐시 키

queryFn: () => getTodayRecommendation(userId)
→ 백엔드 /ai/recommend/{userId} API 호출

enabled: !!userId
→ userId가 있을 때만 API 요청 실행
```

### 설계 이유

AI 추천은 로그인한 사용자별로 달라지는 데이터입니다.

따라서 React Query의 queryKey에 `userId`를 포함해 사용자가 바뀌었을 때 이전 사용자의 AI 추천 데이터가 잘못 재사용되지 않도록 처리했습니다.

---

## 4-8. AI 추천 카드 렌더링 로직

백엔드에서 받은 AI 추천 데이터는 `AIRecommend.jsx`에서 필요한 값만 분리해 카드 컴포넌트로 전달합니다.

```javascript
const {
    routineTitle,
    routineReason,
    routineTags,
    runningTitle,
    runningReason,
    runningTags,
} = recommendation.data.aiResp;
```

### 화면 출력 구조

```text
AIRecommend
→ 오늘의 AI 추천 가이드 영역
→ 루틴 추천 카드
→ 러닝 추천 카드
→ AI 코치 질문하기 버튼
```

### 렌더링 예시

```javascript
<Stack spacing={3}>
    <Recommendation
        title={routineTitle}
        reason={routineReason}
        tags={routineTags}
    />

    <Divider />

    <Recommendation
        title={runningTitle}
        reason={runningReason}
        tags={runningTags}
    />
</Stack>
```

### 설계 이유

AI 추천 결과에는 러닝 추천과 근력 루틴 추천이 함께 포함됩니다.

프론트엔드에서는 하나의 응답 데이터를 받아 화면에서는 두 개의 추천 카드로 분리해 보여주도록 구성했습니다.

```text
aiResp
→ routineTitle / routineReason / routineTags
→ 루틴 추천 카드

aiResp
→ runningTitle / runningReason / runningTags
→ 러닝 추천 카드
```

이렇게 분리하면 사용자는 오늘 해야 할 러닝 운동과 근력 루틴을 한 화면에서 구분해서 확인할 수 있습니다.

---

## 4-9. AI 코치 질문 / 답변 채팅 로직

사용자는 메인 페이지의 `AI 코치 질문하기` 버튼을 눌러 AI에게 운동 관련 질문을 할 수 있습니다.

### 전체 처리 흐름

```text
AI 코치 질문하기 버튼 클릭
→ AIChat 컴포넌트 표시
→ 사용자가 질문 입력
→ 전송 버튼 클릭 또는 Enter 입력
→ POST /ai/question 요청
→ 백엔드에서 사용자 데이터 기반 프롬프트 생성
→ Gemini API 호출
→ AI 답변 생성
→ 질문 / 답변 DB 저장
→ 프론트엔드에서 채팅 목록 재조회
→ 화면에 사용자 질문과 AI 답변 표시
```

### 질문 전송 로직

```javascript
const aiMutation = useMutation({
    mutationFn: (question) => getAIResp({ userId, question }),
    onMutate: async (question) => {
        setNewChats((prev) => [...prev, { type: "user", text: question }]);
        setQuestion("");
    },
    onSuccess: () => {
        setNewChats([]);
        queryClient.invalidateQueries(["getAIChatListByUserId", userId]);
    },
    onError: () => {
        setNewChats((prev) => [
            ...prev,
            { type: "ai", text: "오류가 발생했습니다." },
        ]);
    },
});
```

### 로직 설명

```text
mutationFn
→ 사용자의 질문을 백엔드로 전송

onMutate
→ 서버 응답을 기다리기 전에 사용자 질문을 화면에 먼저 표시
→ 입력창 초기화

onSuccess
→ AI 답변 저장이 완료되면 기존 채팅 목록 query 무효화
→ 서버에서 최신 질문 / 답변 목록 재조회

onError
→ AI 호출 또는 서버 처리 실패 시 오류 메시지 표시
```

### 설계 이유

AI 응답은 생성 시간이 걸릴 수 있습니다.

사용자가 질문을 보냈는데 화면에 아무 반응이 없으면 요청이 처리되고 있는지 알기 어렵습니다.

그래서 `onMutate`에서 사용자의 질문을 먼저 화면에 보여주고, AI 답변 생성 중에는 로딩 UI를 표시하도록 구성했습니다.

---

## 4-10. AI 채팅 기록 조회 로직

AI 채팅 기록은 사용자가 이전에 질문한 내용과 AI 답변을 다시 볼 수 있도록 구성했습니다.

```javascript
const { data: chatList } = useQuery({
    queryKey: ["getAIChatListByUserId", userId],
    queryFn: () => getAIChatListByUserId(userId),
    enabled: !!userId,
});
```

### 처리 흐름

```text
AIChat 컴포넌트 렌더링
→ userId 존재 여부 확인
→ /ai/chatList/{userId} 요청
→ 사용자의 AI 질문 / 답변 목록 수신
→ 질문과 답변을 채팅 메시지 형태로 변환
→ 화면에 말풍선 UI로 표시
```

### 채팅 데이터 변환 로직

```javascript
const historyChat = useMemo(() => {
    if (!chatList?.data) return [];

    return chatList.data.flatMap((chat) => [
        { type: "user", text: chat.question },
        { type: "ai", text: chat.resp },
    ]);
}, [chatList]);
```

### 로직 설명

백엔드에서 받은 데이터는 질문과 답변이 하나의 객체로 들어 있습니다.

```json
{
  "question": "초보자 하체 운동 추천해줘",
  "resp": "스쿼트, 런지, 브릿지를 추천합니다."
}
```

프론트엔드에서는 이를 채팅 화면에 맞게 두 개의 메시지로 변환합니다.

```text
사용자 메시지
→ question

AI 메시지
→ resp
```

### 설계 이유

DB에는 질문과 답변이 하나의 기록으로 저장되지만, 채팅 UI에서는 사용자 말풍선과 AI 말풍선이 분리되어야 합니다.

그래서 `flatMap`을 사용해 하나의 채팅 기록을 두 개의 화면 메시지로 변환했습니다.

---

## 4-11. AI 채팅 기록 초기화 로직

사용자는 AI 채팅 기록을 초기화할 수 있습니다.

### 처리 흐름

```text
채팅 기록 초기화 버튼 클릭
→ confirm 창 표시
→ 사용자가 확인 선택
→ DELETE /ai/chatList/{userId} 요청
→ 백엔드에서 해당 사용자의 AI 질문 기록 삭제
→ React Query query 무효화
→ 화면에서 채팅 기록 제거
```

### 코드 흐름

```javascript
const removeAiChatMutation = useMutation({
    mutationFn: () => removeAiChat(userId),
    onSuccess: () => {
        setNewChats([]);
        queryClient.invalidateQueries(["getAIChatListByUserId", userId]);
        alert("채팅 기록이 초기화되었습니다.");
    },
    onError: () => {
        alert("채팅 기록이 초기화중 오류가 발생했습니다.");
    },
});
```

### 설계 이유

AI 대화 기록은 사용자별 데이터입니다.

기록을 삭제한 뒤에도 화면에 이전 대화가 남아 있으면 서버 상태와 화면 상태가 맞지 않게 됩니다.

그래서 삭제 성공 후 `invalidateQueries`를 실행해 서버 기준 최신 상태를 다시 조회하도록 처리했습니다.

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

## 5-4. AI 기능의 React Query 상태 관리

AI 기능에서는 오늘의 추천 조회, AI 채팅 목록 조회, AI 질문 전송, 채팅 기록 초기화에 React Query를 사용했습니다.

| 기능           | React Query 사용 방식                              |
| ------------ | ---------------------------------------------- |
| 오늘의 AI 추천 조회 | `useQuery(["getTodayRecommendation", userId])` |
| AI 채팅 기록 조회  | `useQuery(["getAIChatListByUserId", userId])`  |
| AI 질문 전송     | `useMutation(getAIResp)`                       |
| AI 채팅 기록 초기화 | `useMutation(removeAiChat)`                    |

### queryKey 설계

```text
["getTodayRecommendation", userId]
→ 사용자별 오늘의 AI 추천 캐시 분리

["getAIChatListByUserId", userId]
→ 사용자별 AI 채팅 기록 캐시 분리
```

### mutation 이후 데이터 갱신

AI 질문 전송에 성공하면 채팅 기록을 다시 조회합니다.

```javascript
queryClient.invalidateQueries(["getAIChatListByUserId", userId]);
```

### 설계 이유

AI 질문에 대한 답변은 백엔드에서 생성되고 DB에 저장됩니다.

따라서 프론트엔드에서 임시로 답변을 만드는 것이 아니라, 서버 저장 완료 후 채팅 기록을 다시 조회해 화면과 DB 상태를 일치시켰습니다.

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

## 6-4. AI 응답 로딩 처리

### 문제

AI 답변 생성에는 시간이 걸릴 수 있습니다.

사용자가 질문을 보낸 뒤 화면에 변화가 없으면 요청이 실패했는지, 처리 중인지 알기 어렵습니다.

### 해결

`aiMutation.isPending` 값을 사용해 답변 생성 중 로딩 UI를 표시했습니다.

```javascript
{aiMutation.isPending && (
    <Paper>
        <CircularProgress size={14} />
        <Typography>
            답변 생성 중...
        </Typography>
    </Paper>
)}
```

### 결과

사용자가 AI 질문을 전송한 뒤 답변 생성 중이라는 상태를 명확히 확인할 수 있어 사용자 경험이 개선되었습니다.

---

## 6-5. AI 질문 전송 실패 처리

### 문제

Gemini API 오류, 백엔드 오류, 네트워크 오류가 발생하면 AI 답변을 받을 수 없습니다.

### 해결

`useMutation`의 `onError`에서 AI 메시지 형태로 오류 문구를 화면에 표시했습니다.

```javascript
onError: () => {
    setNewChats((prev) => [
        ...prev,
        { type: "ai", text: "오류가 발생했습니다." },
    ]);
}
```

### 결과

AI 응답 생성이 실패해도 화면이 멈추지 않고, 사용자에게 오류 상태를 안내할 수 있게 되었습니다.

---

## 6-6. AI 채팅 스크롤 위치 문제

### 문제

질문과 답변이 계속 추가되면 사용자가 직접 아래로 스크롤해야 하는 불편함이 생길 수 있습니다.

### 해결

채팅 메시지 개수나 로딩 상태가 바뀔 때마다 스크롤을 아래로 이동시켰습니다.

```javascript
useEffect(() => {
    scrollToBottom();
}, [displayChatList.length, aiMutation.isPending]);
```

### 결과

사용자가 질문을 보내거나 AI 답변이 생성될 때 최신 메시지가 자동으로 보이도록 개선했습니다.

---

# 7. Frontend 핵심 포인트

Route-In 프론트엔드에서 중요한 부분은 사용자의 행동을 API 요청으로 연결하고, 서버 응답을 기준으로 화면 상태를 갱신하는 것이었습니다.

특히 출석, 추천, 팔로우처럼 데이터가 자주 바뀌는 기능은 프론트 상태만 임시로 변경하면 서버 데이터와 화면 상태가 달라질 수 있습니다.

그래서 React Query를 사용해 서버 데이터를 조회하고, mutation 이후 관련 데이터를 다시 조회하도록 처리했습니다.

출석 시스템에서는 Zustand로 팝업 표시 상태를 즉시 닫고, 서버에는 `PATCH /attendance/popup/shown` 요청을 보내 DB 기준 상태를 저장했습니다.

AI 기능에서는 사용자의 질문을 단순히 화면에 표시하는 것이 아니라, 질문 전송부터 답변 저장, 채팅 기록 재조회까지 서버 상태를 기준으로 일관되게 처리했습니다.

```text
사용자 질문 입력
→ 프론트엔드에서 POST /ai/question 요청
→ 백엔드에서 사용자 데이터 기반 AI 답변 생성
→ 질문 / 답변 DB 저장
→ 프론트엔드에서 채팅 기록 query 무효화
→ 최신 채팅 기록 재조회
→ 화면 갱신
```

또한 오늘의 AI 추천 기능은 사용자별로 다른 데이터를 보여주기 때문에 React Query의 queryKey에 `userId`를 포함했습니다.

이를 통해 사용자가 바뀌었을 때 다른 사용자의 AI 추천이나 채팅 기록이 잘못 표시되는 문제를 방지했습니다.

AI 응답 생성 중에는 로딩 UI를 표시하고, 오류 발생 시 채팅 메시지 형태로 오류를 안내하여 사용자 경험을 유지했습니다.

이 구조를 통해 사용자 경험과 데이터 일관성을 함께 유지할 수 있었습니다.
