# HALO 프로젝트 기본 설정 가이드

> GitHub 원본: [https://github.com/IN-P/HALO.git](https://github.com/IN-P/HALO.git)

---

## ✅ 1. AWS EC2 접속

```bash
ssh -i "[보안키.pem]" ubuntu@ec2-[IPv4-퍼블릭-주소].ap-northeast-2.compute.amazonaws.com
```

> 🔸 **주의:** EC2 도메인에 사용하는 `[IPv4-퍼블릭-주소]`는 일반적인 `0.00.000.00` 형식이 아닌, **`.` 대신 `-`로 변환한 형식**이어야 합니다.  
> 예: `13.125.70.164` → `ec2-13-125-70-164.ap-northeast-2.compute.amazonaws.com`

---

## ✅ 2. 스왑 메모리 추가 (※ 서버 재시작 시 초기화됨)

```bash
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo swapon --show
sudo free -h
```

---

## ✅ 3. Git 클론

```bash
git clone https://github.com/IN-P/HALO.git
cd HALO
ls
```

> `HALO` 디렉토리 생성 여부로 정상 클론 확인

---

## ✅ 4. MySQL 설치 및 설정

```bash
sudo apt-get update
sudo apt-get install mysql-server-8.0
sudo /etc/init.d/mysql status
sudo mysql -uroot
```

MySQL 접속 후 아래 쿼리 실행:

```sql
SELECT user, host, plugin FROM mysql.user;
USE mysql;
UPDATE user SET plugin='caching_sha2_password' WHERE user='root';
FLUSH PRIVILEGES;
SET PASSWORD FOR 'root'@'localhost' = '1234';
EXIT;
```

비밀번호로 접속:

```bash
mysql -uroot -p
# 비밀번호: 1234
```

데이터베이스 생성:

```sql
CREATE DATABASE halo;
```

---

## ✅ 5. 백엔드 패키지 설치

```bash
cd HALO/backend
sudo npm install
```

---

## ⚙️ 백엔드 설정

### 📄 1. `.env` 파일 설정

#### [옵션 1] 로컬에서 AWS로 전송

```bash
scp -i "[보안키.pem]" ".env" ubuntu@[IPv4-퍼블릭-주소]:/home/ubuntu/HALO/backend
```

#### [옵션 2] AWS에서 직접 생성

```bash
cd HALO/backend
touch .env
vi .env
# i (입력 모드) → 내용 붙여넣기 → ESC → :wq!
```

#### 📌 환경변수 예시 (.env)

```env
NEXT_PUBLIC_API_URL=http://[IPv4-퍼블릭-주소]:3000
```

> 🔸 이때 `[IPv4-퍼블릭-주소]`는 `0.00.000.00` 형식 그대로 사용합니다.  
> 도메인용 변형(`-` 변환)은 EC2 도메인에서만 적용됩니다.

---

### 🛠 2. `app.js` 수정

```bash
vi app.js
```

아래 코드 추가 및 수정:

```js
const apiURL = process.env.NEXT_PUBLIC_API_URL;

app.use(morgan('dev'));
app.use(cors({
  origin: apiURL,  // ← 기존 하드코딩된 http:// 주소를 apiURL로 변경
  credentials: true,
}));
```

---

## ⚙️ 프론트엔드 설정

```bash
cd /home/ubuntu/HALO/frontend
```

### 📄 1. `.env` 환경변수 설정

```bash
touch .env
vi .env
```

내용 추가:

```env
REACT_APP_API_URL=http://[IPv4-퍼블릭-주소]:3065
REACT_APP_REDIRECT_URL=http://[IPv4-퍼블릭-주소]:3000
```

---

### 🛠 2. Axios 설정 파일 수정

```bash
vi utils/axiosConfig.js
```

```js
import axios from 'axios';

axios.defaults.withCredentials = true;
axios.defaults.baseURL = process.env.REACT_APP_API_URL;

export default axios;
```

---

### 🔧 3. Redux Saga 내부 axios 경로 수정

```bash
cd sagas
```

#### ✅ 모든 saga 파일에서 다음 수정 적용:

1. **axios import 경로 수정**

```js
// 변경 전
import axios from 'axios';

// 변경 후
import axios from '../utils/axiosConfig';
```

2. **API 주소 하드코딩 제거**

```js
// 변경 전
axios.get("http://localhost:3065/achievements");

// 변경 후
axios.get("/achievements");
```

> `.env`의 `REACT_APP_API_URL`에 맞춰 자동으로 처리되도록 수정

---

## 🚀 서버 실행

> ✅ 터미널 2개를 실행하여 **백엔드**와 **프론트엔드**를 각각 구동하는 것을 추천합니다.

### 🧩 1. 백엔드 실행

```bash
cd /home/ubuntu/HALO/backend
vi models/user.js
```

`models/user.js` 내부에서 다음 항목 수정:

```js
  }, {
    tableName: 'users',  // ← 추가
    charset: 'utf8mb4',
    collate: 'utf8mb4_general_ci',
  });
```

> 🔸 **주의:** 개발 환경은 Windows였지만, 배포 환경은 Linux이므로 **대소문자 구분 문제**로 인해 FK 제약 오류가 발생할 수 있습니다.  
> 이를 방지하기 위해 `tableName`을 명시적으로 설정합니다.

```bash
npm run start
```

---

### 🌐 2. 프론트엔드 실행

```bash
npm run build
npm run start
```

> ⚠️ **build가 반드시 선행되어야 합니다.**  
> 변경사항 반영이 되지 않거나 예기치 않은 오류가 발생할 수 있습니다.  
> **한 글자라도 수정했다면 `npm run build`를 다시 실행해야 합니다.**

---

## 🎉 설정 완료

이제 HALO 프로젝트 서버가 정상적으로 실행됩니다.  
문제 발생 시 `.env`, 포트 설정, 콘솔 로그 등을 먼저 확인해 보세요.

---

## ⚠️ 현재 발생 중인 주요 이슈

> 아래 이슈들은 설정 누락, 환경변수 미반영, 경로 오류 또는 API 서버 상태로 인해 발생할 수 있습니다.

- [ ] 포스트에 댓글 등록 안됨  
- [ ] 포스트 공유 인터랙션 안됨  
- [ ] 포스트 이미지 불러오지 못함  
- [ ] 게시글 수정하기 안됨 (게시글을 불러오지 못함)  
- [ ] 외부 API 로그인 연결 안됨 (localhost로 연결됨)  
- [ ] 채팅 유저 검색 안됨  
- [ ] 뽑기 동작 안됨  
- [ ] 문의 전송 안됨  
- [ ] 멤버십 정보 불러오기 안됨  
- [ ] 카카오페이 결제하기 안됨  
- [ ] 미니 프로필로 리다이렉트 안됨  
- [ ] 프로필 공유하기 복사 안됨  
- [ ] 프로필 이미지 변경 안됨  
- [ ] 프로필 본인 정보 수정 안됨  
- [ ] 비밀번호 변경 안됨  
- [ ] 뱃지 이미지 불러오기 안됨  
- [ ] 날씨 API 연결 안됨

> ✅ 위 문제들은 `.env` 설정, 서버 도메인/포트 불일치, 인증 토큰 문제, 프론트 빌드 누락, 서버/DB 상태 등 전반적인 배포 환경 확인이 필요합니다.

---