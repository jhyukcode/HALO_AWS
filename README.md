# HALO 프로젝트 기본 설정 가이드

> GitHub 원본: [https://github.com/IN-P/HALO.git](https://github.com/IN-P/HALO.git)

---

## ✅ 1. AWS EC2 접속

```bash
ssh -i "[보안키.pem]" ubuntu@ec2-[IPv4-퍼블릭-주소].ap-northeast-2.compute.amazonaws.com
```

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

#### 📌 환경변수 추가 (.env 예시)

```env
NEXT_PUBLIC_API_URL=http://[IPv4-퍼블릭-주소]:3000
```

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
axios.defaults.baseURL = process.env.NEXT_PUBLIC_API_URL;

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

## 🎉 설정 완료

이제 서버 실행을 통해 HALO 프로젝트를 구동할 수 있습니다.
