# Qurtz Helm Chart

Xpenology NAS에 있는 Obsidian vault를 NFS 퍼시스턴트 볼륨으로 모니터링하여 자동으로 "🧠 Taliesin's Second Brain" Quartz 정적 사이트를 빌드하는 Kubernetes 차트입니다.

## Overview

이 Helm 차트는 Xpenology NAS의 Obsidian vault 파일이 변경될 때마다 자동으로 Quartz 정적 사이트를 재빌드하는 시스템을 배포합니다. 체크섬 기반 폴링 방식을 사용하여 안정적인 파일 변경 감지와 디바운싱 기능을 제공합니다.

## Architecture

이 차트는 **통합 아키텍처**를 사용하여 단일 파드에서 3개의 컨테이너가 협력합니다:

- **File Watcher**: 체크섬 기반 폴링으로 NFS vault 변경을 감지하는 Alpine 컨테이너
- **Quartz Builder**: Node.js 22 기반으로 Quartz 정적 사이트를 빌드하는 컨테이너
- **Web Server**: Nginx 1.16으로 생성된 정적 파일을 서빙하는 컨테이너
- **NFS Persistent Volume**: Xpenology NAS의 Obsidian vault를 위한 10Gi PV/PVC
- **Shared Volumes**: 컨테이너 간 데이터 공유를 위한 EmptyDir 볼륨들

## Prerequisites

- NFS를 지원하는 Kubernetes 클러스터
- NFS 익스포트가 설정된 Xpenology NAS
- Helm 3.x

## Installation

1. values.yaml 또는 test-values.yaml에 NFS 서버 정보를 설정하세요:

```yaml
nfs:
  server: "192.168.1.100"  # Xpenology NAS IP 주소
  path: "/volume1/obsidian"  # NAS의 Obsidian vault 경로
  readOnly: false

quartz:
  gitRepo: "https://github.com/jackyzha0/quartz.git"  # 기본 Quartz 설정
  gitBranch: "v4"

fileWatcher:
  pollInterval: 5  # 폴링 간격 (초)
```

2. 차트를 설치합니다:

```bash
# 기본 설정으로 설치
helm install qurtz . --namespace qurtz --create-namespace

# 또는 테스트 설정으로 설치
helm install qurtz . -f test-values.yaml
```

## Configuration

### NFS Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `nfs.server` | Xpenology NAS IP 주소 | `""` |
| `nfs.path` | NAS의 Obsidian vault 경로 | `""` |
| `nfs.readOnly` | 읽기 전용으로 마운트 | `false` |

### File Watcher Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `fileWatcher.pollInterval` | 체크섬 기반 폴링 간격 (초) | `5` |
| `fileWatcher.image.repository` | File Watcher 이미지 | `alpine` |
| `fileWatcher.image.tag` | 이미지 태그 | `latest` |

### Quartz Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `quartz.gitRepo` | Quartz 설정이 있는 Git 저장소 | `"https://github.com/jackyzha0/quartz.git"` |
| `quartz.gitBranch` | 사용할 Git 브랜치 | `"v4"` |
| `quartz.image.repository` | Quartz Builder 이미지 | `node` |
| `quartz.image.tag` | 이미지 태그 | `"22-alpine"` |

## How It Works

1. **File Monitoring**: File Watcher 컨테이너가 체크섬 기반 폴링으로 NFS 퍼시스턴트 볼륨의 Obsidian vault를 모니터링합니다
2. **Change Detection**: MD5 체크섬 비교를 통해 vault 콘텐츠 변경을 감지하며, 10초 디바운싱을 적용합니다
3. **Rebuild Trigger**: 변경 감지 시 Watcher가 같은 파드의 Quartz Builder 컨테이너에 HTTP POST 요청을 전송합니다
4. **Content Sync**: Builder가 rsync를 사용해 vault 콘텐츠를 Quartz 콘텐츠 디렉토리로 동기화합니다
5. **Site Generation**: Quartz가 "🧠 Taliesin's Second Brain" 제목으로 정적 사이트를 빌드합니다
6. **Content Serving**: 생성된 파일이 같은 파드의 Nginx 컨테이너로 복사되어 즉시 서빙됩니다

## Networking

차트는 **통합 아키텍처**로 단일 디플로이먼트를 생성합니다:
- `qurtz-watcher`: 3개 컨테이너 (File Watcher, Quartz Builder, Web Server)
- 내부 통신: Builder는 포트 3001에서 리빌드 요청을 수신
- 외부 접근: Service가 Web Server의 포트 80을 클러스터에 노출

## Troubleshooting

### Common Issues

1. **NFS Mount Fails**
   - NFS 서버 IP와 경로 확인
   - Xpenology에서 NFS 익스포트 설정 확인
   - Kubernetes 노드에서 NAS 접근 가능한지 확인

2. **File Changes Not Detected**
   - 폴링 간격 설정 확인 (fileWatcher.pollInterval)
   - NFS 퍼시스턴트 볼륨 마운트 상태 확인
   - 체크섬 계산에 포함되는 파일 타입 확인 (*.md, *.png, *.jpg 등)
   - 컨테이너 로그 확인: `kubectl logs -f deployment/qurtz-watcher -c file-watcher`

3. **Build Failures**
   - Quartz 설정 저장소 확인 (기본값: https://github.com/jackyzha0/quartz.git)
   - Node.js 22 의존성 및 메모리 설정 확인
   - favicon 생성 오류 방지를 위한 스크립트 확인
   - Builder 로그 확인: `kubectl logs -f deployment/qurtz-watcher -c quartz-builder`

### Useful Commands

```bash
# 디플로이먼트 상태 확인
kubectl get pods

# PVC 상태 확인
kubectl get pvc

# File Watcher 로그 보기
kubectl logs -f deployment/qurtz-watcher -c file-watcher

# Builder 로그 보기
kubectl logs -f deployment/qurtz-watcher -c quartz-builder

# Web Server 로그 보기
kubectl logs -f deployment/qurtz-watcher -c web-server

# 사이트 접속하기 (통합 서비스)
kubectl port-forward service/qurtz 8080:80

# 전체 파드 상태 상세 보기
kubectl describe pod -l app.kubernetes.io/name=qurtz
```

## Features

- ✅ **안정적인 파일 모니터링**: 체크섬 기반 폴링으로 NFS 환경에서도 신뢰성 확보
- ✅ **디바운싱**: 연속된 파일 변경을 10초 간격으로 그룹화하여 불필요한 빌드 방지
- ✅ **통합 아키텍처**: 단일 파드에서 모든 컴포넌트 실행으로 리소스 효율성 극대화
- ✅ **퍼시스턴트 스토리지**: NFS PV/PVC로 안정적인 데이터 저장
- ✅ **자동 사이트 커스터마이징**: "🧠 Taliesin's Second Brain" 브랜딩 자동 적용
- ✅ **빌드 안정성**: favicon 생성 오류 방지 및 메모리 최적화