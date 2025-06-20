# 프로젝트 설계
이 다이어리 프로젝트는 여러 사용자가 동시에 수정과 편집을 할 수 있는 웹 기반의 페이지 입니다.
사용자는 다른사람들과 편집, 수정 등이 가능하고 개인 다이어리를 만들어 기록 또한 가능합니다.

---

## 기능 구현
- 구글 로그인(Auth 2.0 사용)
- 로그아웃 기능
- 토큰 발행 & 재발행 기능 구현
- 다이어리 내용 저장
- Firebase DB를 통해 다이어리 작성 내용 저장
- 다이어리 내용 실시간 동기화 기능 구현
- 동시 편집 기능 구현
- 다이어리 선택창 UI 구현
- 사용자 권한 설정 기능
- 사용자 DB에 저장된 다이어리 불러오기
- 다이어리 내용 수정, 삭제 기능 구현

---

## 주요 기능

### 다이어리 생성 및 수정(삭제)
- 다이어리를 생성하면 자동으로 저장되고 생성 시간이 저장됩니다.
- 생성 날짜 기준으로 정렬되며 다이어리 알림 기능도 사용 가능합니다.

### 다른 사람들과 편집, 수정
– Firebase Firestore를 사용하여 여러 사용자가 동시에 동일한 변경을 수행하더라도 변경 사항이 바로 저장됩니다.( 실시간 리스너 기능 사용)
- 서버의 오류로 오프라인에서 작업을 한다 해도 서버 재연결 시 자동으로 동기화가 가능합니다.
- 다른사람들과 서로 편집을 할 때 오류가 생겨 충돌하게 되어도 방지가 가능하도록 설계하였습니다.
- 향후 WebRTC에 대한 위치 공유 서비스를 추가 할 예정입니다.

### 사용자 및 관리자 권한
– 각 사용자는 읽기, 쓰기 및 관리자 권한이 있는 계정을 갖습니다.
- 초대 링크를 사용하여 다른 사람을 자신의 다이어리에 초대할 수 있으며, 관리자는 액세스 규칙(접근 권한)을 설정할 수 있습니다.
- 서버는 로그인을 안했거나 초대받지 않은 사용자가 자신의 다이어리 페이지에 들어오는 것을 방지하는 기능 또한 탑재되어 있습니다.

### 편집 기능 내역 및 보안 
– 사용자들의 편집에 대한 정보는 MongoDB에 저장되어 관리합니다.
- JWT 토큰은 만료되면 자동 로그아웃 처리가 됩니다.
- 특정 기준(예: 작성 후 n일이 경과한 경우)에 대한 알림 기능도 포함되어 있습니다.

--- 

## 개발 예정 기능

- 공동 편집자 초대 기능 확장
- 실시간 커서 위치 표시(WebRTC 기반)
- 알림 시스템(초대 요청 등)
- 향상된 편집 기능(Markdown 지원, 텍스트 포맷팅)
- 향상된 모바일 UI 반응성
- 글의 양, 기여도 등을 보여주는 통계 대시보드 제공

---

## TDD 이슈 구분

| 기능 구분         | 이슈 내용                                      | 해결 방법/비고               
-----------------------------------------|------------------------------------
| 다이어리 저장   | 실시간 저장 중 동시 편집 시 충돌 발생  | Firestore 트랜잭션 및 `updatedAt` 기준 동기화적용 |
| 권한 체크         | 공유된 유저가 수정 가능한지 검증 필요  | 권한 필드 (`sharedWith`) 기반 조건 추가 |
| 토큰 인증         | 토큰 만료 시에도 페이지 접근되는 문제 발생  | 토큰 만료 확인 후 자동 로그아웃 처리 |
| 데이터 무결성   | 삭제된 다이어리가 다른 사용자에게 보이는 문제 | Firestore에서 `isDeleted` 플래그 처리 |
| 로그인/로그아웃 | Firebase Auth의 비동기 처리 	| useEffect 내부에서 비동기 처리 분리 |

--- 

## 단위 테스트 코드

### 다이어리 저장 - 실시간 저장 & 충돌 처리 테스트
import { saveDiary } from '@/lib/firebaseFunctions'; /
import { firestore } from '@/lib/firebase';
import { runTransaction } from 'firebase/firestore';

jest.mock('firebase/firestore');

describe('다이어리 저장 테스트', () => {
  it('동시 편집 시 updatedAt 기준으로 병합 처리', async () => {
    const diaryRef = { id: 'testDiaryId' };
    const mockTransaction = {
      get: jest.fn().mockResolvedValue({
        exists: () => true,
        data: () => ({ updatedAt: 1000 }),
      }),
      update: jest.fn(),
    };

    (runTransaction as jest.Mock).mockImplementation(async (_, callback) => {
      return await callback(mockTransaction);
    });

    await saveDiary(diaryRef, {
      updatedAt: 2000,
      content: '최신 내용',
    });

    expect(mockTransaction.update).toHaveBeenCalledWith(
      diaryRef,
      expect.objectContaining({ content: '최신 내용' })
    );
  });
});

### 토큰 만료 시 자동 로그아웃
import { checkTokenValidity, logoutUser } from '@/lib/auth'; 

jest.mock('@/lib/auth');

describe('토큰 만료 확인 로직', () => {
  it('만료된 토큰이면 자동 로그아웃', async () => {
    (checkTokenValidity as jest.Mock).mockReturnValue(false);
    const mockLogout = jest.fn();
    (logoutUser as jest.Mock).mockImplementation(mockLogout);

    if (!checkTokenValidity()) {
      logoutUser();
    }

    expect(mockLogout).toHaveBeenCalled();
  });
});



## 디렉터리 구조

client/              # React 프론트엔드 코드
server/              # Express 서버 및 API 코드
firestore-rules/     # Firestore 보안 규칙 정의
mongo-schemas/       # MongoDB 스키마 정의
public/              # 정적 리소스 (이미지 등)
.env                 # 환경 변수 설정 파일

---



## 활용 예시

- 일기 or 성찰 기록
- 팀원과의 회의 도중 내용 저장
- 프로젝트 활동 중 실시간으로 메모 공유
- 프레젠테이션 초안 작성 및 동료 피드백 기록

---

## 문제 상황 및 해결 내용
프로젝트를 하다가 문제가 생긴다면 어떤 문제점이 있는 지 파악하고 잘 안 풀린다면
여기서 어떻게 하는 게 좋을 것인지 팀원들끼리 해결 후 문제를 해결하는 방식으로 하였다

---

## 사용된 기술 및 설계 목적

### front end

- 웹 개발 화면은 React로 구현하였습니다.
- styled-components를 사용하여 동적 스타일링과 UI 정렬을 구현했고,
- 타입의 안정성과 유지관리를 용이하게 하기 위하여 TypeScript를 사용하였습니다.

### back end
- Express.js 기반의 Node.js 서버를 통해 API를 제공합니다.
- 실시간으로 다어리 내용을 백업, 동기화 하기 위하여 Firebase의 Firestore를 사용하였습니다.
- MongoDB를 사용하여 사용자 데이터와 권한, 변경 내역 등을 저장합니다.
- 향후 위치 공유 기능에 WebRTC를 사용할 예정입니다.

### 인증 및 보안
- Firebase 인증을 통해 이메일과 Google 계정을 연동 하였습니다.
- JWT를 사용하여 세션을 유지하고 인증이 필요한 요청에 대해서는 토큰을 통해 액세스를 제어합니다.

---

## 역할 분담
프로젝트의 UI 작업은 이기재 팀원이, 
Firebase DB를 쓰는 작업은 최우혁 팀원이, 
다이어리 기능 작업은 박성한 팀원과 제갈윤미 팀장이 하였다.

---

## 기여자

- 최우혁
- 이기재
- 제갈윤미
- 박성한

---

## 실행 방법

### 환경 변수 설정(.'env')

PORT=4000
MONGO_URI=mongodb+srv://<YOUR_MONGO_URI>
FIREBASE_API_KEY=<FIREBASE_KEY>
FIREBASE_PROJECT_ID=<PROJECT_ID>
JWT_SECRET=your_jwt_secret

### 실행 명령어

```bash
# 클라이언트 실행
cd client
npm install
npm start 

# 서버 실행
cd server
npm install
npm run dev

---

## 마무리 향후 계획

본 프로젝트는 실시간 데이터 집계, 사용자 권한 관리, 이력 추적 기록 등
다양한 기능을 반영하여 설계되었습니다.
향후 다중 사용자 기록, 기능 확장, 모바일 환경 지원을 통하여
더 많은 사용자에게 편리한 환경을 제공할 계획입니다.
