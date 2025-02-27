# Router-Hacking-Guideline

광운대학교 blackcat 동아리 프로젝트 : ‘라우터 취약점 찾기’

> 프로젝트 기간: 2025년 1월 ~ 2025년 2월(약 2개월)
> 

2개월동안 분석한 결과를 공유하고, 공유기 해킹을 입문하는 사람에게 가이드라인을 제공한다.

두 종류의 공유기(netis mex605 ax3000, tew-831dr)를 대상으로 취약점을 분석하고 탐색한 취약점을 제보했다.

## 저자

> 광운대학교 kwang_researchers 팀
> 

[장형범](https://github.com/hbeooooooom)(PM), [강주영](https://github.com/jju00), [김영균](https://github.com/GHYoungKyun), [남승욱](https://github.com/Jujack0x0), [유영준](https://github.com/merryberry01), [조민우](https://github.com/cmw9957)

## 카테고리

---

### 사전 지식

1. [연구배경](https://github.com/hbeooooooom/router_vuln_analyze/blob/main/Background%20of%20study/README.md#%EC%97%B0%EA%B5%AC-%EB%B0%B0%EA%B2%BD)
   - [사전지식](https://github.com/hbeooooooom/router_vuln_analyze/blob/main/Background%20of%20study/README.md#1-%EC%82%AC%EC%A0%84-%EC%A7%80%EC%8B%9D)

### 분석 방법

1. [분해 과정](https://github.com/hbeooooooom/router_vuln_analyze/tree/main/Teardown%20process#2-%EB%B6%84%ED%95%B4-%EA%B3%BC%EC%A0%95)
    - [tew-831dr](https://github.com/hbeooooooom/router_vuln_analyze/tree/main/Teardown%20process#21-tew-831dr)
    - [mex605](https://github.com/hbeooooooom/router_vuln_analyze/tree/main/Teardown%20process#12-mex605)
2. [부팅 프로세스](https://github.com/hbeooooooom/router_vuln_analyze/tree/main/Booting%20process#3-%EB%B6%80%ED%8C%85-%ED%94%84%EB%A1%9C%EC%84%B8%EC%8A%A4-%EB%B6%84%EC%84%9D)
    - [tew-831dr](https://github.com/hbeooooooom/router_vuln_analyze/blob/main/Booting%20process/README.md#21-tew-831dr-%EB%B6%80%ED%8C%85-%ED%94%84%EB%A1%9C%EC%84%B8%EC%8A%A4-%EB%B6%84%EC%84%9D)
    - [mex605](https://github.com/hbeooooooom/router_vuln_analyze/tree/main/Booting%20process#22-mex605-%EB%B6%80%ED%8C%85-%ED%94%84%EB%A1%9C%EC%84%B8%EC%8A%A4-%EB%B6%84%EC%84%9D)
3. [통신 방법](https://github.com/hbeooooooom/router_vuln_analyze/blob/main/Communication%20method/README.md#4-%ED%86%B5%EC%8B%A0-%EB%B0%A9%EB%B2%95)
    - [tew-831dr](https://github.com/hbeooooooom/router_vuln_analyze/blob/main/Communication%20method/README.md#41-tew-831dr)
4. [분석 환경](https://github.com/hbeooooooom/router_vuln_analyze/blob/main/Analysis%20environment/README.md#4-%EB%B6%84%EC%84%9D-%ED%99%98%EA%B2%BD)
    - [mex605 Firmware Emulation & Debugging](https://github.com/hbeooooooom/router_vuln_analyze/blob/main/Analysis%20environment/README.md#41-mex605-firmware-emulation--debugging)
    - [포트 포워딩](https://github.com/hbeooooooom/router_vuln_analyze/blob/main/Analysis%20environment/README.md#42-%ED%8F%AC%ED%8A%B8-%ED%8F%AC%EC%9B%8C%EB%94%A9)

### 결론

1. 성과
   - [CVE-2025-25881](https://diagnostic-tower-322.notion.site/tew831dr-command-injection-9616a693a3584f8f88973cb9330eebde?pvs=4)
   - [CVE-2025-25882](https://diagnostic-tower-322.notion.site/MEX605-Stored-XSS-Vulnerability-Port-Mirroring-page-fc417b0026e14177b5de6c92db8d6107?pvs=4)
   - [CVE-2025-25883](https://diagnostic-tower-322.notion.site/MEX605-Stored-XSS-Vulnerability-Port-Mirroring-page-0e4c360c3f5e4993ac34b41022635d2e?pvs=4)
   - [CVE-2025-25884](https://diagnostic-tower-322.notion.site/MEX605-Stored-XSS-Vulnerability-remote-access-page-ACL-configuration-0f0da49ae94c4691a330d75481d2514a?pvs=4)
   - [CVE-2025-25886](https://diagnostic-tower-322.notion.site/MEX605-Stored-XSS-Vulnerability-wol-html-page-185c2ef102e8805aa47ae4db64e8df4a)
   - [CVE-2025-25887](https://diagnostic-tower-322.notion.site/MEX605-Stored-XSS-Vulnerability-advertisement-html-185c2ef102e8800cacc0c901dfde0223)
   - [CVE-2025-25888](https://diagnostic-tower-322.notion.site/MEX605-Stored-XSS-Vulnerability-vpn-html-pw-185c2ef102e880da93d4d190e9b76bde)
   - [CVE-2025-25889](https://diagnostic-tower-322.notion.site/MEX605-Stored-XSS-Vulnerability-vpn-html-page-id-185c2ef102e880dabf2ddf7dbbf121eb)
   - [CVE-2025-25890](https://diagnostic-tower-322.notion.site/MEX605-Stored-XSS-Vulnerability-vpn-html-page-name-185c2ef102e880e89cdada520d696b78?pvs=73)
