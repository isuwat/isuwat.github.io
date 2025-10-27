---
layout: default
title: 애플 인증서 갱신
parent: Cheat Sheet
nav_order: 5
---


애플 쪽은 **인증서(Certificate)** 와 **프로비저닝 프로필(Provisioning Profile)** 이 서로 연결돼 있기 때문에 **항상 인증서를 먼저 갱신하고 → 그 인증서를 포함한 새로운 프로비저닝 프로필을 갱신**해야 합니다.

---

## 📌 권장 순서

1. **개발자/배포 인증서 갱신 (Certificate Renewal)**
    
    - Keychain Access에서 CSR 생성 → Apple Developer 계정에서 새 인증서 발급 → .cer 설치        
    - 필요하다면 `.p12`로 추출하여 CI/CD 환경에도 적용
    -
1. **프로비저닝 프로필 갱신 (Profile Renewal)**
    
    - 기존 프로필은 만료된 인증서를 포함하고 있으므로 새 인증서로 바꿔줘야 합니다.        
    - Developer Portal → _Profiles_ → 해당 프로필 **Edit → 새 인증서 선택 → Save → Download**        
    - 새 프로필을 Xcode나 CI/CD 환경에 반영
        
2. **빌드/테스트**
    
    - Xcode 프로젝트에서 새 프로필 반영 확인        
    - TestFlight 빌드 배포 테스트로 정상 동작 여부 확인