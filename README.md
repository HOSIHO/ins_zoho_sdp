Zoho ServiceDesk Plus API 연동 명세서
2026년 11월 10일
1. 개요

첫 단계는 코드가 아니라 우리 회사 ServiceDesk Plus가 Cloud인지 On-Premise(사내 설치형)인지 확인하는 것입니다. 둘은 인증 방식과 주소가 완전히 달라서, 이 답에 따라 이후 설계가 갈립니다.
• ServiceDesk Plus(SDP): Zoho Corporation의 IT 관리 브랜드인 ManageEngine이 만든 ITSM(서비스데스크/티켓) 솔루션입니다. 요청(티켓), 문제, 변경, 자산 등을 관리합니다.
• REST API v3: 웹 화면에서 하는 거의 모든 작업(티켓 생성·조회·수정·노트 추가 등)을 HTTP 호출로 할 수 있게 열어둔 창구입니다. (공식 소개)
• 이 문서의 목표: 사내 PC(로컬 미들웨어)에서 SDP API를 호출해 티켓 등록·조회·처리결과 기록을 자동화하는 것.
구분
이번 범위
이후 확장
기능
연결 테스트, 요청 조회, 요청 생성, 노트 추가
상태 변경, 워크로그, 엑셀 일괄 등록
실행 위치
개인 PC의 Python 스크립트
사내 서버 상주(스케줄러)
권한
테스트용 기술자(Technician) 계정 1개
전용 서비스 계정 + 최소 권한 역할



2. 0단계: 사전 확인
브라우저로 티켓 시스템에 접속했을 때의 주소창 URL만 보면 대부분 판별됩니다.
판별 기준
Cloud (SaaS)
On-Premise (사내 설치형)
접속 주소 예
sdpondemand.manageengine.com, servicedeskplus.jp 등, 또는 회사 커스텀 도메인
http(s)://사내서버명:8080 처럼 사내 IP/호스트 + 포트
로그인 화면
Zoho 계정 로그인(accounts.zoho...)
SDP 자체 로그인 또는 AD 로그인
API 인증
Zoho OAuth 2.0 (Client ID/Secret + Refresh Token)
기술자 API 키(authtoken)
API 문서 위치
공식 웹 문서
SDP 관리자 화면 → Admin → API → Documentation
인증 방식 차이는 여러 연동 제품 문서에서도 동일하게 설명됩니다: Cloud는 Zoho API Console의 Self Client + Refresh Token, On-Prem은 기술자 API 키 (Rapid7 가이드, AutomationEdge).
담당자에게 확인할 것
[ ] SDP 배포 형태: Cloud / On-Prem, 그리고 버전(On-Prem이면 Admin 화면 하단 빌드 번호)
[ ] Cloud라면 데이터센터(US/JP/IN 등)와 포털 이름(URL의 /app/<portal> 부분)
[ ] API용 계정 발급 가능 여부: 내 계정 키 사용 vs 전용 서비스 계정
[ ] 그 계정의 역할(Role)과 요청 모듈 권한(조회/생성/수정)
[ ] 사내 PC에서 해당 주소로 나가는 443(또는 8080/8443) 통신 허용 여부, 프록시/SSL 검사 장비 유무
[ ] 보안팀 승인 절차: API 키 발급·외부 통신 신청서가 필요한지
[ ] 테스트용 티켓을 만들어도 되는 요청 템플릿/카테고리(운영 티켓 오염 방지)
[ ] 동료가 운영 중인 로컬 미들웨어의 ITSM 연동 방식(이미 API 키나 코드가 있을 수 있음)
3. 아키텍처와 방화벽 요건
통신은 항상 내 PC에서 서버로 나가는 방향 하나만 필요합니다(아웃바운드). 응답은 같은 연결로 돌아오므로 PC 쪽에 포트를 열 필요는 없습니다. Cloud는 ①에서 토큰을 받고 ②를 호출하는 두 단계입니다.
경로
목적지
포트
방향
확인 포인트
A안 On-Prem
SDP 서버 IP/호스트명
8080 또는 8443(설치 설정에 따름)
PC → 사내 서버
사용자망↔서버망 사이 내부 방화벽/ACL
B안 ① 토큰
accounts.zoho.com(데이터센터별)
443
PC → 인터넷
프록시 사용 시 Python에 proxies 설정
B안 ② API
sdpondemand.manageengine.com 등 또는 커스텀 도메인
443
PC → 인터넷
SSL 검사 장비가 있으면 사내 CA 인증서 등록
4. 인증 명세
인증은 “이 호출을 누가 보냈는가”를 SDP에 증명하는 절차입니다. 키/토큰을 가진 계정의 권한을 그대로 물려받으므로, 비밀번호와 동급으로 다뤘야 합니다.
4-A. On-Premise: 기술자 API 키(authtoken)
1. SDP에 API용 계정으로 로그인 → 우측 상단 프로필 → Generate API Key / Generate Authtoken (또는 관리자가 Admin → Technicians → 해당 기술자 편집에서 발급)
2. 만료일을 지정해 발급하고 키를 안전한 곳에 보관
3. 모든 요청 헤더에 authtoken: <발급받은 키> 추가
• 키는 로그인 권한이 있는 사용자가 만료일 있음/없음으로 발급하며, 해당 계정 로그인이 비활성화되면 키도 삭제됩니다. (공식 API 문서)
• 관리자가 아닌 역할은 만료일 없는 토큰을 만들 수 없습니다. (1Password 가이드)
• API 키는 해당 기술자 역할의 권한을 그대로 상속합니다. (Rapid7 가이드)
• 버전에 따라 PORTALID 헤더가 추가로 필요하다는 커뮤니티 보고가 있습니다. 401이 나면 사내 SDP의 Admin → API 문서에서 필수 헤더를 확인하세요. (GitHub 예제)
4-B. Cloud: Zoho OAuth 2.0 (Self Client)
사람이 매번 로그인하지 않는 개인 스크립트는 Self Client 방식이 가장 간단합니다.
1. Zoho API Console에서 Add Client → Self Client 생성 → Client ID / Client Secret 확보
2. Self Client의 Generate Code 탭에서 스코프 입력(예: SDPOnDemand.requests.READ,SDPOnDemand.requests.CREATE) → Grant Token(code) 발급(유효시간 짧음)
3. Grant Token을 Access Token + Refresh Token으로 교환(아래 ①)
4. 이후에는 Refresh Token으로 Access Token만 재발급(아래 ②)
5. API 호출 헤더: Authorization: Zoho-oauthtoken <access_token>
# ① Grant Token -> Access/Refresh Token (최초 1회)
curl -X POST https://accounts.zoho.com/oauth/v2/token \
  -d grant_type=authorization_code \
  -d client_id=<CLIENT_ID> -d client_secret=<CLIENT_SECRET> \
  -d code=<GRANT_TOKEN>

# ② Refresh Token -> 새 Access Token (만료될 때마다)
curl -X POST https://accounts.zoho.com/oauth/v2/token \
  -d grant_type=refresh_token \
  -d client_id=<CLIENT_ID> -d client_secret=<CLIENT_SECRET> \
  -d refresh_token=<REFRESH_TOKEN>
• accounts.zoho.com은 US 기준입니다. 데이터센터가 다르면 해당 지역의 accounts 주소를 써야 합니다. (Zoho Self Client 문서)
• Access Token은 주기적으로 만료되며, 유효시간은 토큰 응답의 expires_in(초)으로 내려옵니다. 응답의 api_domain을 API 기본 주소로 쓰고 특정 지역 URL을 하드코딩하지 말라고 안내합니다. (SDP Cloud OAuth 문서)
• 스코프는 토큰이 할 수 있는 일을 제한하고, 승인한 사용자의 기술자 역할이 한 번 더 제한합니다. (Rapid7 가이드)
스코프
용도
SDPOnDemand.requests.READ
요청(티켓) 조회
SDPOnDemand.requests.CREATE
요청 생성
SDPOnDemand.requests.UPDATE
요청 수정·상태 변경
SDPOnDemand.requests.ALL
요청 모듈 전체(테스트 후 최소 권한으로 축소 권장)
5. 공통 API 규격
SDP v3의 가장 큰 특징은 JSON을 본문에 그대로 넣지 않고 input_data라는 폼 파라미터에 담아 보낸다는 점입니다. 초보자가 가장 많이 막히는 지점입니다.
5-1. Base URL
환경
Base URL 형식
On-Premise
http(s)://<사내서버>:<포트>/api/v3 (기본 포트 예: 8080)
Cloud (US)
https://sdpondemand.manageengine.com/app/<portal>/api/v3
Cloud (JP)
https://servicedeskplus.jp/app/<portal>/api/v3
Cloud (기타)
EU .eu, IN .in, AU .net.au, UK .uk 등 — 토큰 응답의 api_domain 사용
Cloud 데이터센터별 주소는 SDP Cloud OAuth 문서, 포털 경로(/app/<portal>)는 요청 API 문서 기준입니다.
5-2. 공통 헤더
헤더
값
비고
Accept
application/vnd.manageengine.sdp.v3+json
v3 응답 형식 지정
Content-Type
application/x-www-form-urlencoded
POST/PUT 시
authtoken
<API 키>
On-Prem만
Authorization
Zoho-oauthtoken <access_token>
Cloud만
5-3. input_data 규칙
• POST/PUT: 본문을 input_data=<JSON 문자열> 형태의 폼 데이터로 전송
• GET(목록 조회): ?input_data=<JSON> 쿼리스트링으로 검색·페이징 조건 전달(URL 인코딩 필수)
• JSON의 최상위 키는 모듈 이름(request, note, list_info 등)
• 참조 필드(요청자·우선순위·템플릿 등)는 문자열이 아니라 {"name": "High"}처럼 객체로 넣음
5-4. 응답 형식
모든 응답에는 response_status가 들어 있고, 성공 시 status_code는 2000, status는 "success"입니다. (요청 API 문서)
{
  "request": { "id": "2290537481381500", "display_id": "39", "subject": "..." },
  "response_status": { "status_code": 2000, "status": "success" }
}
• id는 API에서 쓰는 내부 번호, display_id는 화면에 보이는 티켓 번호입니다. 후속 호출(수정·노트)에는 **id**를 씁니다.
• 실패 시 status가 "failed"이고 messages 배열에 오류 내용이 담깁니다. 상세 코드표는 공식 문서의 Status Codes 항목을 따릅니다.
6. 엔드포인트 명세
첫 버전은 아래 5개만 구현하면 티켓 자동화의 핵심 흐름(등록 → 조회 → 처리내용 기록)이 끝납니다. 경로는 5-1의 Base URL 뒤에 붙입니다.
ID
기능
메서드
경로
필수 입력
우선순위
API-01
요청 목록 조회(연결 테스트 겸용)
GET
/requests
없음(list_info 선택)
1
API-02
요청 단건 조회
GET
/requests/{id}
id
1
API-03
요청 생성
POST
/requests
subject
1
API-04
요청 수정(상태·담당자 등)
PUT
/requests/{id}
id + 변경 필드
2
API-05
노트(처리 메모) 추가
POST
/requests/{id}/notes
description
2
API-06
워크로그(작업시간) 추가
POST
/requests/{id}/worklogs
사내 버전 문서 확인
3
요청 생성의 필수 필드는 subject 하나뿐이며, 필드는 최상위 request 객체 아래에 넣습니다. (요청 API 문서) API-04~06의 세부 필드는 버전별 차이가 있을 수 있어, 구현 전 사내 SDP의 API 문서로 최종 확인합니다.
API-03 요청 생성 — 요청 예시
# On-Prem 예시 (Cloud는 authtoken 대신 Authorization: Zoho-oauthtoken ... )
curl -X POST "https://sdp.사내도메인:8080/api/v3/requests" \
  -H "Accept: application/vnd.manageengine.sdp.v3+json" \
  -H "authtoken: <API_KEY>" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode 'input_data={
    "request": {
      "subject": "[자동등록] 모니터 지급 요청",
      "description": "엑셀 3행 데이터 기반 자동 생성",
      "requester": { "email_id": "user@company.com" },
      "request_type": { "name": "Service Request" },
      "priority": { "name": "High" },
      "template": { "name": "Raise a New Monitor Request" }
    }
  }'
선택 필드로 impact, urgency, category, subcategory, item, group, technician, site, status, udf_fields(사용자 정의 필드) 등을 쓸 수 있습니다. 값(이름)은 우리 SDP에 실제 등록된 이름과 글자 하나까지 같아야 합니다.
API-01 목록 조회 — input_data 예시
{
  "list_info": {
    "row_count": 50,
    "start_index": 1,
    "sort_field": "created_time",
    "sort_order": "desc",
    "get_total_count": true
  }
}
• 페이징: start_index를 row_count만큼 늘려가며 반복 호출
• 검색 조건은 list_info.search_criteria에 넣습니다(형식은 공식 문서의 Search Criteria 항목 참고).
API-05 노트 추가 — input_data 예시
{
  "note": {
    "description": "AD 계정 생성 완료 (자동화 스크립트)",
    "show_to_requester": false
  }
}
7. 로컬 미들웨어 설계
Python + requests 라이브러리로 시작하고, 설정·인증·API 호출·업무 로직을 파일로 분리하면 나중에 Cloud/On-Prem이 바뀌어도 인증 파일만 고치면 됩니다.
7-1. 폴더 구조
sdp-middleware/
├─ .env                 # 비밀값 (Git/공유폴더 업로드 금지)
├─ config.py            # .env 읽기, Base URL·모드 결정
├─ auth.py              # On-Prem: authtoken 헤더 / Cloud: 토큰 갱신·캐시
├─ sdp_client.py        # 공통 호출 함수(헤더, input_data, 재시도, 로깅)
├─ jobs/
│   ├─ ping.py          # API-01 연결 테스트
│   └─ excel_to_ticket.py  # 엑셀 → 티켓 일괄 등록
└─ logs/
7-2. 설정값(.env)
키
예시
비고
SDP_MODE
onprem / cloud
0단계 확인 결과
SDP_BASE_URL
https://sdp.사내도메인:8080/api/v3
끝에 / 없이
SDP_API_KEY
****
On-Prem만
ZOHO_ACCOUNTS_URL
https://accounts.zoho.com
Cloud만, 데이터센터별
ZOHO_CLIENT_ID / ZOHO_CLIENT_SECRET / ZOHO_REFRESH_TOKEN
****
Cloud만
SDP_VERIFY_SSL
true
사내 인증서면 CA 파일 경로 지정
DRY_RUN
true
true면 실제 전송 없이 로그만
7-3. 핵심 코드 골격 (sdp_client.py)
import json, requests
from config import BASE_URL, MODE, VERIFY
from auth import auth_headers   # onprem: {"authtoken": KEY} / cloud: {"Authorization": "Zoho-oauthtoken ..."}

ACCEPT = {"Accept": "application/vnd.manageengine.sdp.v3+json"}

def call(method, path, payload=None):
    headers = {**ACCEPT, **auth_headers()}
    url = f"{BASE_URL}{path}"
    data = {"input_data": json.dumps(payload, ensure_ascii=False)} if payload else None
    if method == "GET":
        r = requests.get(url, headers=headers, params=data, verify=VERIFY, timeout=15)
    else:
        r = requests.request(method, url, headers=headers, data=data, verify=VERIFY, timeout=15)
    body = r.json()
    status = body.get("response_status", {})
    if isinstance(status, list):
        status = status[0]
    if r.status_code >= 400 or status.get("status") != "success":
        raise RuntimeError(f"{r.status_code} {status}")
    return body

# 사용 예
# call("GET", "/requests", {"list_info": {"row_count": 5}})
# call("POST", "/requests", {"request": {"subject": "테스트"}})
7-4. 운영 규칙
• 토큰 관리(Cloud): Access Token과 만료시각을 메모리/파일에 캐시하고, 만료 5분 전이나 401 응답 시에만 갱신. 매 호출마다 갱신하면 Zoho 토큰 발급 한도에 걸릴 수 있음(Zoho Token limits)
• 재시도: 네트워크 오류·5xx는 최대 3회(2·4·8초 간격), 4xx는 재시도하지 않고 입력값 확인
• 중복 방지: 엑셀 행마다 생성된 id/display_id를 결과 열에 기록, 값이 있는 행은 건너뜀
• 로깅: 시각, 메서드, 경로, 결과 코드, 티켓 번호를 logs/에 저장. API 키·토큰은 절대 로그에 출력하지 않음
• DRY_RUN 기본값 true: 처음 돌릴 때는 보낼 JSON만 출력해 확인
8. 단계별 진행 로드맵
각 단계는 통과 조건을 만족해야 다음으로 넘어갑니다. 문제가 생기면 어느 단계(통신/인증/데이터)인지 바로 좌혁집니다.
1. 사전 확인 — 2장 체크리스트 완료. 통과: Cloud/On-Prem, URL, API 계정 확정
2. 통신 확인(L3/L4) — PowerShell Test-NetConnection <호스트> -Port 443(또는 8080). 통과: TcpTestSucceeded : True
3. 키/토큰 발급 — 4장 절차. 통과: 키 또는 Refresh Token 확보, .env 저장
4. Postman/curl 수동 호출 — API-01 row_count: 1. 통과: status_code 2000
5. Python 연결 테스트 — jobs/ping.py. 통과: 스크립트로 동일 결과
6. 테스트 티켓 1건 생성 — 제목에 [TEST] 표기. 통과: 화면에서 티켓 확인 후 수동 종료
7. 엑셀 일괄 등록(DRY_RUN → 실전송) — 3행으로 시작. 통과: 결과 열에 티켓 번호 기록, 중복 없음
8. 기존 업무 연결 — AD 계정 생성 스크립트 결과를 API-05 노트로 자동 기록
테스트 시나리오
TC
상황
기대 결과
확인 포인트
TC-01
정상 키로 목록 조회
2000 success
연결·인증 정상
TC-02
키 한 글자 틀림
401 또는 인증 실패 메시지
에러 처리·로그에 키 미노출
TC-03
subject 빠진 생성
failed + 필수값 오류
입력 검증
TC-04
없는 우선순위 이름(예: 긴급)
failed
이름 일치 여부
TC-05
한글 제목·설명 생성
화면에서 깨짐 없이 표시
UTF-8 / URL 인코딩
TC-06
Cloud Access Token 만료 후 호출
자동 갱신 후 성공
토큰 갱신 로직
TC-07
같은 엑셀 파일 두 번 실행
2번째는 전부 건너뜀
중복 방지
TC-08
네트워크 차단(랜선 분리)
3회 재시도 후 실패 기록
재시도·타임아웃
9. 보안 고려사항
• 전용 계정 + 최소 권한: 개인 관리자 계정 키 대신, 요청 모듈만 다루는 역할의 서비스 계정 사용. 키는 계정 권한을 그대로 갖습니다.
• 만료일 지정: On-Prem 키는 만료 없음으로 발급하지 말고 주기(예: 6개월) 교체
• 비밀값 보관: .env는 개인 PC에만, 공유폴더·메일·메신저 전송 금지. 가능하면 Windows 자격 증명 관리자(keyring 라이브러리) 사용
• SSL 검증 유지: 사내 사설 인증서라서 오류가 나면 verify=False 대신 사내 CA 인증서 파일 경로를 지정
• 유출 시 대응: 키 재발급(Regenerate)으로 기존 키 무효화, Cloud는 Refresh Token revoke
• 사전 승인: 개인 도구라도 운영 ITSM에 쓰기 작업을 하므로, 팀장/보안 담당에게 이 문서를 공유하고 허가를 받는 것을 권장
참고자료
• ServiceDesk Plus Cloud API v3 소개
• SDP Cloud OAuth 2.0 (데이터센터별 URL, 스코프)
• SDP Cloud 요청(Request) API
• SDP API 키(authtoken) 안내
• Zoho Self Client 인증 흐름
• Rapid7 SDP 연동 가이드 (Cloud/On-Prem 비교)
• 1Password SDP On-Prem 가이드
• On-Prem 사내 문서: SDP 로그인 → Admin → API → Documentation (AutomationEdge)
