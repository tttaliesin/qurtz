# Qurtz Helm Chart

Xpenology NAS에 있는 Obsidian vault를 NFS로 모니터링하여 자동으로 Quartz 정적 사이트를 빌드하는 Kubernetes 차트입니다.

## Overview

이 Helm 차트는 Xpenology NAS의 Obsidian vault 파일이 변경될 때마다 자동으로 Quartz 정적 사이트를 재빌드하는 시스템을 배포합니다. inotify를 사용하여 파일 변경을 실시간으로 감지하고 즉시 리빌드를 실행합니다.

## Architecture

- **File Watcher**: inotify-tools를 사용해 NFS 마운트된 Obsidian vault를 모니터링하는 Alpine 컨테이너
- **Quartz Builder**: Quartz를 사용해 정적 사이트를 빌드하는 Node.js 컨테이너  
- **Web Server**: 생성된 정적 파일을 서빙하는 Nginx 컨테이너
- **NFS Volume**: Xpenology NAS의 Obsidian vault를 마운트
- **Shared Storage**: 빌드 결과물을 위한 임시 볼륨

## Prerequisites

- NFS를 지원하는 Kubernetes 클러스터
- NFS 익스포트가 설정된 Xpenology NAS
- Helm 3.x

## Installation

1. values.yaml에 NFS 서버 정보를 설정하세요:

```yaml
nfs:
  server: "192.168.1.100"  # Xpenology NAS IP 주소
  path: "/volume1/obsidian"  # NAS의 Obsidian vault 경로
  readOnly: false

quartz:
  gitRepo: "https://github.com/yourusername/your-quartz-config.git"
  gitBranch: "main"
```

2. 차트를 설치합니다:

```bash
helm install qurtz . --namespace qurtz --create-namespace
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
| `fileWatcher.events` | 감시할 inotify 이벤트 | `"modify,create,delete,move"` |
| `fileWatcher.pollInterval` | 폴링 간격 (초) | `5` |

### Quartz Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `quartz.gitRepo` | Quartz 설정이 있는 Git 저장소 | `""` |
| `quartz.gitBranch` | 사용할 Git 브랜치 | `"main"` |

## How It Works

1. **File Monitoring**: File Watcher 컨테이너가 inotify를 사용해 NFS 마운트된 Obsidian vault를 모니터링합니다
2. **Change Detection**: 파일이 수정, 생성, 삭제, 이동될 때 이벤트가 트리거됩니다
3. **Rebuild Trigger**: Watcher가 Quartz Builder 컨테이너에 리빌드 요청을 전송합니다
4. **Content Sync**: Builder가 vault 콘텐츠를 Quartz 콘텐츠 디렉토리로 동기화합니다
5. **Site Generation**: Quartz가 정적 사이트를 빌드합니다
6. **Content Serving**: 생성된 파일이 Web Server의 문서 루트로 복사됩니다

## Networking

차트는 두 개의 디플로이먼트를 생성합니다:
- `qurtz-watcher`: File Watcher와 Quartz Builder (포트 3001에서 내부 통신)
- `qurtz-web`: Nginx Web Server (포트 80으로 서비스 노출)

## Troubleshooting

### Common Issues

1. **NFS Mount Fails**
   - NFS 서버 IP와 경로 확인
   - Xpenology에서 NFS 익스포트 설정 확인
   - Kubernetes 노드에서 NAS 접근 가능한지 확인

2. **File Changes Not Detected**
   - inotify 이벤트 설정 확인
   - NFS 마운트가 읽기 전용이 아닌지 확인
   - 컨테이너 로그 확인: `kubectl logs -f deployment/qurtz-watcher -c file-watcher`

3. **Build Failures**
   - Quartz 설정 저장소 확인
   - Node.js 의존성 확인
   - Builder 로그 확인: `kubectl logs -f deployment/qurtz-watcher -c quartz-builder`

### Useful Commands

```bash
# 디플로이먼트 상태 확인
kubectl get pods -n qurtz

# File Watcher 로그 보기
kubectl logs -f deployment/qurtz-watcher -c file-watcher -n qurtz

# Builder 로그 보기
kubectl logs -f deployment/qurtz-watcher -c quartz-builder -n qurtz

# Web Server 로그 보기
kubectl logs -f deployment/qurtz-web -n qurtz

# 사이트 접속하기
kubectl port-forward service/qurtz-web 8080:80 -n qurtz
```