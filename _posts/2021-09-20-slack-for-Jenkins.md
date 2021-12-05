---
layout: post
title: "Slack을 이용한 Jenkins 알림 설정"
author: "jhkim593"
tags: Jenkins
---

Slack을 이용해서 Jenkins 빌드 후 알림을 받을 수 있도록 설정해보겠습니다.

<!-- **Tale's** -->
<!-- `_config.yml` -->

1. 알림을 받을 채널을 개설합니다.

2. 더보기-> 앱 클릭 후 Jenkins CI를 설치 합니다. <br>    
<img src="https://user-images.githubusercontent.com/53510936/133948094-2d024464-1e9e-42f4-8e14-93c76f6f1530.png"  width="300" height="350" style="margin-left: auto; margin-right: auto; display: block;"/>


3. Jenkins CI 설치 후 에 추가 버튼을 클릭합니다. <br>  
<img src="https://user-images.githubusercontent.com/53510936/133948464-c7b90ab0-f21b-4a1c-82c4-49299921724e.png"  width="600" height="350"/>

4. Jenkins 알림을 받을 채널을 설정합니다. <br>  
<img src="https://user-images.githubusercontent.com/53510936/133948572-cd53291d-68b8-45e5-858f-c99572b455bf.png"  width="600" height="350"/>

5. Jenkins 설정에 팀 하위 도메인과 통합 토큰 자격 증명 ID가 사용되므로 복사하고 , 세부 설정들을 수정한 뒤 저장합니다.  <br>  
<img src="https://user-images.githubusercontent.com/53510936/133948748-b853384a-d80d-4a0c-b849-cc8d0024a7e2.png"  width="600" height="350"/>

6. Jenkins 관리 -> 플러그인 관리 -> ***Slack Notification*** 플러그인을 설치합니다.  <br>  
<img src="https://user-images.githubusercontent.com/53510936/134147242-acec5854-27b2-4fc9-b566-184ac331f5d7.png"  width="600" height="350"/>  

7. Jenkins 관리 -> 시스템 설정으로 이동한 뒤 <br>Slack 설정 WorkSpace에 5번 항목에서 복사해둔 ***팀 하위 도메인*** 을 입력합니다.<br> Default channel / member id에는 Jenkins 알림을 받기로 한 **채널명**을 입력합니다.  <br>  
<img src="https://user-images.githubusercontent.com/53510936/134148448-ac7ccf0a-f2f6-42f4-beb6-db3896307dd0.png"  width="600" height="350"/>

8. Credential 항목의 Add 버튼을 클릭합니다.<br> Kind 에서 Secret text 을 선택 <br>Secret에서 5번항목에서 복사한 **통합 토큰 자격 증명 ID** 입력합니다.<br> ID를 임의로 작성 후 Add 버튼을 누릅니다.  <br>  
<img src="https://user-images.githubusercontent.com/53510936/134149429-52e50d68-9cd8-43ca-839f-13a8aa3f64ad.png"  width="600" height="350"/>

9. 정상적으로 설정되었다면 Test Connection 버튼을 클릭시 Success 표시가 뜨고 Slack에 메세지가 옵니다.<br>  
<img src="https://user-images.githubusercontent.com/53510936/134152195-f95dae09-5490-4c9e-80af-9f6a695d92bc.png"  width="600" height="350"/>

10. Slack메세지를 받을 **project** 를 선택합니다. 해당 **project** 의 구성으로 이동합니다.<br>
빌드 후 조치 에서 Slack Notification 을 선택한 뒤 원하는 항목을 체크하고 저장버튼을 클릭합니다.<br>  
<img src="https://user-images.githubusercontent.com/53510936/134152667-783aff13-c149-4951-b760-c7572c3166b7.png"  width="600" height="350"/>

11. Jenkins를 통해 빌드를 진행하면 아래처럼 Slack으로 설정한 메시지가 전송됩니다.<br>   
<img src="https://user-images.githubusercontent.com/53510936/134153067-3c7038e8-b193-4f12-b910-47ecddd6b6da.png"  width="600" height="220"/>
