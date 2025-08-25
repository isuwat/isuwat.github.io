---
layout: default
title: 배포/로컬 분리
parent: Jekyll
nav_order: 1
---

# Jekyll + Just the Docs 배포 환경 정리

Here are some Jekyll production and local development tips.

## 🚀 배포 환경 (Production: GitHub Pages)

GitHub Pages에서는 remote_theme를 사용하여  
GitHub 서버가 Just the Docs 테마를 원격으로 받아옵니다.  


### 1. _config.yml
```yaml
remote_theme: just-the-docs/just-the-docs
plugins:
  - jekyll-feed
  - jekyll-remote-theme

title: My Docs
description: Project Documentation
url: "https://USERNAME.github.io"
```

### 2. GitHub Pages 빌드
* 별도 명령어 필요 없음
* main 브랜치에 푸시 → GitHub Actions(GitHub Pages)에서 자동 빌드
* _config.yml만 사용됨 (로컬용 _config.dev.yml은 무시됨)

## 📦 로컬 개발 환경 (Development)

로컬에서는 `just-the-docs` 테마를 **gem**으로 설치해서 사용합니다.    
빠르고 오프라인에서도 개발 가능합니다.

### 1. Gemfile
```ruby
gem "jekyll", "~> 4.4.1"
gem "just-the-docs"        # 로컬 개발용 테마

group :jekyll_plugins do
  gem "jekyll-feed"
end
```

### 2. 로컬 전용 설정 파일 (_config.dev.yml)
```yaml
theme: "just-the-docs"
url: "http://localhost:4000"
```

### 3. 실행 방법
```bash
bundle install
bundle exec jekyll serve --config _config.yml,_config.dev.yml
```

## ⚡ 요약

로컬 개발 → gem 기반 테마 (theme: just-the-docs)  
배포(GitHub Pages) → 원격 테마 (remote_theme: just-the-docs/just-the-docs)