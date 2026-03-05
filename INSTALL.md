# Fukubukuro 통합 설치 가이드

이 문서는 **OOTD 패션 이커머스(Fukubukuro)** 프로젝트를 로컬 환경에 구성하고 실행하기 위한 통합 설치 가이드입니다. 
전체 시스템은 **Backend(Spring Boot)**, **Crawler(FastAPI)**, **AI(FastAPI)** 3개의 주요 파트로 구성되어 있습니다.

## 📌 시스템 구성 및 포트 안내

시스템 내의 분리된 서비스들이 원활하게 통신하기 위해 각 서비스는 아래의 포트를 기본으로 사용합니다.

- **Backend (Spring Boot)**: `9999`
- **Crawler API (FastAPI)**: `8888`
- **AI FastFit (피팅)**: `8000`
- **AI Veo 3.1 (비디오)**: `8001`
- **AI Gemini (라벨링/사주)**: `8002`

---

## 🛠 0. 사전 요구사항 (Prerequisites)

### 0.1 공통 필수 도구
- **Git**
- **Docker & Docker Compose** (DB 및 크롤러 구동 권장)

### 0.2 로컬 직접 구동 시 필수
- **Java 21+** (프로젝트는 JDK 21을 사용)
- **Python 3.10+** (AI 및 크롤러 서버 구동용)
- **MySQL 8.x** (Docker 미사용 시 별도 설치)

### 0.3 AI GPU 구동 시
- **CUDA 지원 GPU** (VRAM 8GB 이상 권장)
- **NVIDIA Container Toolkit** (AI를 Docker로 구동할 경우 필요)

### 0.4 🔑 필수 API Keys
각종 코어 기능 연동을 위해 다음 API 키들의 발급이 필수적입니다.
- **[Google Gemini API Key](https://aistudio.google.com/apikey)**: AI 라벨링, 사주, 비디오 생성 서비스에 필수
- **[Toss Payments Test Keys](https://developers.tosspayments.com/)**: 결제 연동
- **[Naver Developers OAuth](https://developers.naver.com/)**: 소셜 로그인 연동

---

## 🚀 1. 소스코드 다운로드 및 환경 변수 설정

### 1.1 Repository 클론
```bash
git clone <repository-url>
cd fukubukuro
```

### 1.2 환경 변수(.env) 세팅
프로젝트 내 총 4곳에 `.env` 파일을 생성하고 키를 기입해야 합니다. 각 폴더에 존재하는 `.env.example` 파일을 복사하여 올바른 값을 기입하세요.

```bash
# 1. Backend 환경변수 세팅
cp backend/.env.example backend/.env

# 2. AI FastFit 환경변수 세팅
cp AI/server/fastfit/.env.example AI/server/fastfit/.env

# 3. AI Gemini 환경변수 세팅
cp AI/server/gemini/.env.example AI/server/gemini/.env

# 4. AI Veo 3.1 환경변수 세팅
cp AI/server/veo3.1/.env.example AI/server/veo3.1/.env
```

> **주의**: 
> - `backend/.env`에는 DB 계정 정보, Toss API Key, Naver API Key 등이 포함되어야 합니다.
> - `AI/.env`에는 발급받은 Gemini API Key가 반드시 기입되어야 정상 동작합니다.

---

## 🗄️ 2. 데이터베이스 및 크롤러 서버 구동 (추천: Docker Compose)

가장 간단한 구축 방법은 크롤러 폴더에서 제공하는 `docker-compose.yml`을 활용하여 **MySQL**과 **Crawler API**를 함께 실행하는 것입니다.

### 2.1 도커로 크롤러 & DB 실행
```bash
cd crawler
docker compose up --build -d
cd ..
```
- 실행 후 FastAPI 크롤러는 `http://localhost:8888`에서 대기합니다.
- MySQL 컨테이너는 호스트의 `3307` 포트로 매핑됩니다. (백엔드 `application.properties`나 `.env`의 접속 포트가 3307로 올바르게 설정되어 있는지 체크하세요.)

### 2.2 (선택) DB 초기화
Spring Boot 실행 시 JPA의 `ddl-auto=update` 속성 덕분에 테이블은 자동 생성됩니다. 하지만 필요시 초기 데이터를 수동으로 밀어넣을 수 있습니다.
```bash
# 프로젝트 루트에서 실행 (초기 스키마 수동 적용 시):
mysql -u root -p -h 127.0.0.1 -P 3307 ootd < backend/ootd.sql
```

---

## 🤖 3. AI 서버 구성 및 구동

AI 서버는 딥러닝 모델 용량이 크므로 실행 전에 모델 저장소를 먼저 클론해야 합니다. (실제 weights는 최초 실행 시 Hugging Face에서 다운받게 됩니다.)

### 3.1 핵심 모델 클론
```bash
# 루트 디렉토리(fukubukuro)에서 실행
cd AI/models
git clone https://github.com/chenxwh/cog-RMBG.git
git clone https://github.com/Zheng-Chong/FastFit.git
cd ../../
```

### 3.2 서버 실행 방식 선택

개발 시 편의를 위해 로컬에서 직접 구동하거나, 배포 환경에서 Docker 컨테이너를 구동하는 방식을 택할 수 있습니다.

#### [방법 A] 로컬 구동 (Conda 환경 추천)
```bash
# 가상환경 세팅 및 PyTorch with CUDA 설치
conda create -n ootd-ai python=3.10
conda activate ootd-ai
# 본인 환경에 맞는 CUDA 버전을 선택 (아래는 cu128 예시)
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu128

# 1. FastFit 서버 (API Key 불필요)
cd AI/server/fastfit
pip install -r requirements.txt
pip install easy-dwpose --no-dependencies
uvicorn app.main:app --host 0.0.0.0 --port 8000

# 2. Gemini AI 서버 (새 터미널 열기 권장)
conda activate ootd-ai
cd AI/server/gemini
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8002

# 3. Veo 3.1 서버 (새 터미널 열기 권장)
conda activate ootd-ai
cd AI/server/veo3.1
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8001
```

#### [방법 B] Docker 기반 GPU 구동 (Vast.ai, RunPod 등)
클라우드 GPU 인스턴스에서 쉽게 띄우려면 다음과 같이 도커를 사용하세요.
```bash
cd AI/server/fastfit
docker build -t fastfit-server .
docker run --gpus all -p 8000:8000 -v $(pwd)/../../models:/models fastfit-server
```

---

## ☕ 4. Backend (Spring Boot) 서버 구동

위의 **DB, Crawler, AI가 모두 정상적으로 떠있는 상태**에서 뷰페이지와 비즈니스 로직을 서빙하는 백엔드 서버를 구동합니다.

```bash
# 루트 디렉토리(fukubukuro)에서 실행
cd backend

# Windows 환경
.\gradlew.bat bootRun

# macOS / Linux 환경
./gradlew bootRun
```
> **운영(Production) 환경 배포 실행방법**:  
> `./gradlew bootJar` 명령으로 빌드 후 `java -jar build/libs/ootd-0.0.1-SNAPSHOT.jar --spring.profiles.active=prod`로 실행할 수 있습니다.

---

## ✅ 5. 최종 실행 상태 확인 가이드

모든 서버가 구동되었다면, 다음 URL에 접속하여 정상적으로 띄워지고 상호 통신이 되는지 확인하세요!

1. **메인 서비스 (UI)**: [http://localhost:9999](http://localhost:9999)
2. **AI FastFit Swagger**: [http://localhost:8000/docs](http://localhost:8000/docs)
3. **AI Veo 3.1 Swagger**: [http://localhost:8001/docs](http://localhost:8001/docs) 
4. **AI Gemini Swagger**: [http://localhost:8002/docs](http://localhost:8002/docs)
5. **크롤러 API Swagger**: [http://localhost:8888/docs](http://localhost:8888/docs)

수고하셨습니다! 이제 OOTD 패션 이커머스 파이프라인 전체를 활용하실 수 준비가 완료되었습니다.
