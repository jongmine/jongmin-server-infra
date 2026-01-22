# 🏠 Homepage Dashboard Guide

Homepage는 서버의 상태를 모니터링하고 서비스로 이동할 수 있는 대시보드입니다. 모든 설정은 `YAML` 파일로 관리됩니다.

## 📂 Configuration Files

위치는 `roles/homepage/templates/` 입니다.

| 파일명              | 역할                  | 주요 설정 항목                               |
| :------------------ | :-------------------- | :------------------------------------------- |
| `settings.yaml.j2`  | 전체 테마, 레이아웃   | `layout`, `background`, `title`              |
| `services.yaml.j2`  | 메인 화면 위젯 & 링크 | `System Monitor`, `Infrastructure` 그룹 정의 |
| `widgets.yaml.j2`   | 상단 바 (Header)      | `greeting`, `datetime`, `logo`               |
| `bookmarks.yaml.j2` | 하단 링크 (선택)      | 현재는 비워둠                                |

## 🎨 Customization

### 1. 위젯 추가 (Glances)

시스템 정보를 보여주는 Glances 위젯을 추가하려면 `services.yaml.j2`를 수정합니다.

```yaml
- CPU Usage:
    icon: cpu
    widget:
      type: glances
      url: "http://glances:61208" # 내부 Docker Network 통신
      metric: cpu
      chart: true
```

### 2. 아이콘 변경

Homepage는 `dashboard-icons` 팩을 내장하고 있습니다.

- [아이콘 검색하기](https://github.com/walkxcode/dashboard-icons)
- 사용법: `icon: brand-name` (예: `plex`, `jellyfin`, `linux-mint`)

### 3. 레이아웃 변경 (`settings.yaml.j2`)

위젯 개수에 맞춰 `columns` 값을 조정하면 예쁘게 정렬됩니다.

```yaml
layout:
  System Monitor:
    style: row
    columns: 3 # 6개 위젯 -> 3x2 배열
```

## 🌐 Networking

- **접속 주소**: `https://jongmine.cloud`
- **내부 포트**: 3000 (Traefik을 통해서만 접근 권장)
