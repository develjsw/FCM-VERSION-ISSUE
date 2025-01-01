### FCM Legacy API Deprecated

---
- 기존 FCM ( Firebase Cloud Messaging ) API 지원종료 / 중단과 관련된 이슈 해결과정 기록
---

- 이슈 내용
  - 통화 분석 요청 후 APP에서 `통화 분석 결과`를 확인하려고 할 때, 상태가 '`분석중`'에서 `멈춰 있는 이슈` 발생
  - 통화 분석은 외부 업체의 STT ( Speech-to-Text ) API를 활용해 진행됨
    - STT란? 음성을 분석해 텍스트 데이터로 변환하는 기술로, 통화 중 대화 내용을 텍스트 형태로 저장하거나 활용할 수 있도록 지원하는 기능


- 가설 및 검증 과정
  - DB 값 확인
    - 통화 분석 결과와 상태값이 `정상적으로` '`분석완료`'로 `저장`되어 있는 것을 확인
    - 저장된 값에 이상이 없으며, 로그에도 별도의 `에러가 기록되어 있지 않음`을 확인
  - APP Push와 관련된 가설 수립
    - APP의 통화 분석 결과 확인 화면은 Native APP으로 구현되어 있으며, APP Push 발송 후 전달받은 data 값을 활용하여 새로 데이터를 받아와 표시하는 구조
    - 이 점을 기반으로, APP Push 발송에 문제가 있을 가능성을 가설로 설정하고 Push 발송 소스코드를 확인함
  - APP Push 발송 소스코드 검증
    - Push 발송 부분은 별도의 try ~ catch로 처리되어 있었으며, statusCode가 200이면 정상 처리로 간주하는 로직 확인
    - 그러나 내부 응답값에서 아래와 같은 에러 발견
      ~~~
      { 
          success: 0,
          failure: 1,
          results: { error: 'DeprecatedApi' } 
      }
      ~~~
  - 문제 원인 파악
    - FCM API 관련 공식문서 확인 결과
      - `2023년 6월 20일` `지원 중단`
      - `2024년 7월 22일` `서비스 종료 시작`
      ![FCM API에서 HTTP v1로 이전](https://github.com/user-attachments/assets/e308ea57-0fdd-4e25-9397-70ff1abbae81)
    - 따라서, `HTTP v1 버전`으로 `마이그레이션이 필요`하다는 결론에 도달함


- 해결 과정
  - Firebase 서비스 계정 키(JSON 파일) 생성 후, 이를 통해 client_email, private_key, token_uri 값을 사용하여 Google OAuth 2.0 Access Token을 발급받아 Push 발송에 사용 
  - Legacy API → HTTP v1 API로 마이그레이션
    - Firebase Admin SDK 대신 HTTP v1 API를 선택한 이유
      - Firebase Admin SDK는 Node.js 18 이상에서 사용 권장되지만, 내부 서비스가 Node.js 16 버전을 사용하고 있어 SDK 도입 전 버전 업그레이드가 필요한 상황이였음 ![firebase admin sdk 기본요건](https://github.com/user-attachments/assets/6e0c48b4-1ae9-4de7-ac56-6eb0db02527c)
      - HTTP v1 API는 Legacy API와 구조적으로 유사하여 기존 로직을 최소한으로 수정하며 전환할 수 있었음
      - 25년도 PUSH 발송 API 통합 계획에 따라, 현재 문제를 신속하고 안정적으로 해결하기 위해 HTTP v1 API를 사용


- 재발 방지대책 + 확장성 고려
  - 재발 방지
    - 에러 처리 및 모니터링 강화
      - HTTP v1 API는 Legacy API와 달리, HTTP 상태 코드와 함께 상세한 오류 메시지를 제공
        - `2024년 12월 17일부터` `Legacy API`는 results 배열의 error 필드로 Deprecated API를 알리던 방식에서, `HTTP status 코드로 에러를 반환`하는 방식으로 변경됨  
      - 이를 기반으로 에러 발생 시 예외 처리 로직을 명확히 구현하여 안정성을 강화
      - 예외 발생 시 Slack 알림을 자동 발송하도록 설정하여, 실시간으로 문제를 감지하고 빠르게 대응할 수 있도록 함
  - 확장성
    - 멀티 PUSH 발송 기능 구현
      - 멀티 발송 기능이 사용되고 있는 곳은 없었지만, 확장성 확보를 위해 구현 
        - HTTP v1 API는 registration_ids 배열 방식 대신 반복문으로 개별 발송하는 구조 
    - iOS 지원 추가
      - 기존 API에는 Android만 대상으로 설정되어 있었으나, HTTP v1 API로 마이그레이션하면서 iOS 지원 추가
  - 코드 모듈화 작업
    - 산발적으로 작성된 PUSH 관련 소스코드를 하나의 모듈로 통합
    - 25년도 PUSH 발송 통합 작업에 대비해 통합 비용을 줄이고 개발 생산성을 높임


- 참고사항
  - Legacy API vs HTTP v1 API
    - Legacy API
      - 요청 형식 예시
         ~~~
         {
             registration_ids: [FCM TOKEN_1, FCM TOKEN_2 ... 값들],
             priority: 'high',
             data: {
                 pushType: 값,
                 id: 값,
                 title: 값,
                 body: 값
             }
         }
         ~~~ 
      - 응답 형식 예시
         ~~~
         {
             "multicast_id": 216,
             "success": 1, // 성공 개수
             "failure": 2, // 실패 개수
             "canonical_ids": 0,
             "results": [
                 {"message_id": "1:234234234234"}, // 성공
                 {"error": "Unavailable"}, // 실패
                 {"error": "InvalidRegistration"} // 실패
             ]
         }
         ~~~
    - HTTP v1 API
      - 요청 형식 예시
         ~~~
         {
             message: {
             token: FCM TOKEN값,
             notification: {
                 ...(body && { body }),
                 title,
             },
             ...(android && { android }),
             ...(apns && { apns }),
             ...(data && { data }),
             },
         }
         ~~~
      - 응답 형식 예시
         ~~~
         // 성공
         {
             name: 'projects/프로젝트ID/messages/0:1734511116280273%071979c5071979c5'
         }
        
         // 실패
         {
             "error": {
                 "code": 400,
                 "message": "Invalid JSON payload received. Unexpected token.",
                 "status": "INVALID_ARGUMENT",
                 "details": [
                     {
                         "@type": "type.googleapis.com/google.rpc.BadRequest",
                         "fieldViolations": [
                             {
                                 "field": "message.notification.title",
                                 "description": "The title field is required."
                             }
                         ]
                     }
                 ]
             }
         }
         ~~~
      
| **특징**          | **Legacy API**                        | **HTTP v1 API**                                                                                             |
|-----------------|---------------------------------------|-------------------------------------------------------------------------------------------------------------|
| **멀티 PUSH 발송 방식** | `registration_ids` 배열로 한 번에 발송        | 반복문을 통해 개별 메시지 발송                                                                               |
| **인증 방식**       | 서버 키 ( Server Key )                   | OAuth 2.0 Bearer Token                                                                                      |
| **엔드포인트**       | `https://fcm.googleapis.com/fcm/send` | `https://fcm.googleapis.com/v1/projects/{project-id}/messages:send`                                         |
| **오류 보고 및 응답**  | 간단한 성공/실패 상태만 반환                      | 상세한 오류 코드와 메시지 제공                                                                               |
| **메시지 구조**      | 단순 JSON 형식, 플랫폼별 세부 설정이 제한적           | JSON 구조화, 플랫폼별 키 블록 제공으로 세부 설정 가능                                                       |

