# Jupyter Lab Dockerfile

이 Dockerfile은 Python 3.11-slim 이미지를 기반으로 JupyterLab 환경을 빠르게 구축할 수 있도록 작성되었습니다.

## 주요 내용
- Python 3.11-slim 이미지를 사용합니다.
- 빌드에 필요한 필수 패키지(build-essential, curl, git)를 설치합니다.
- pip을 최신 버전으로 업그레이드하고, JupyterLab을 설치합니다.
- 기본 작업 디렉토리는 `/workspace`입니다.
- 8888 포트를 외부에 개방합니다.
- 컨테이너 실행 시 JupyterLab이 자동으로 실행됩니다.

---

## docker-compose.yml 설명
- `docker-compose.yml` 파일을 이용해 JupyterLab 컨테이너를 쉽게 실행할 수 있습니다.
- 8888 포트를 호스트와 컨테이너에 매핑합니다.
- `./workspace` 폴더를 컨테이너의 `/workspace`에 마운트하여, 로컬 파일을 바로 사용할 수 있습니다.
- 컨테이너 이름은 `jupyterlab-container`로 지정되어 있습니다.

### 실행 방법
```bash
docker-compose up --build
```
