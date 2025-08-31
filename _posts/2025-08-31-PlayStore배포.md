---
title: "Play Store 배포를 위한 CI/CD 구축하기"
categories: [Git, DevOps]
tags: [Flutter, Git, DevOps]
---
# 들어가며
현재 barlow 어플리케이션은 Google Play Store에 입점해있습니다. Play Store에 어플리케이션을 업데이트하는 과정은 다음과 같습니다.
```
[수정사항 빌드]
    ↓
[Play Console에 번들 추가]
    ↓
[특정 트랙 버전 승급]
    ↓
[Google에 의한 심사]
    ↓
[심사통과/거절]
```

Play Store의 정책이 바뀌어 신규 개발계정에 대해 까다로운 심사를 요구하므로 이에 대응하기 위해 Play Console을 통해 빌드 산출물은 수동으로 업데이트하고 트랙을 관리했었습니다.

현재는 이를 `Github Actions`를 사용하여 workflow단위로 자동화 하였습니다.

## 왜 Github Actions인가?
`Jenkins`, `GitLab` 등 강력한 기능을 제공하는 CI/CD 도구들이 존재하나 barlow app 프로젝트에서는 `Github Actions`를 사용합니다. 그 이유는,,,,

**편하다 ★★★☆☆**
- Github 이벤트를 통해 쉽게 통합이 가능
- 저장소와 워크플로우가 한 곳에 통합되어 관리하기 쉬움

**싸다 ★★★★★**
- 현재 추가적 비용 없이 workflow 사용 가능
- CI/CD를 위한 별도의 호스팅 불필요


**쉽다 ★★★★☆** 
- Marketplace를 통해 이미 만들어진 Actions 재사용 가능
- 빠르게 구축 가능

<br>

![예시](/assets/posts/2025-08-31/2.png){: width="300" height = "300"}
_가난합니다_

현재 Barlow 프로젝트는 소규모로 진행되고 있기 때문에 GitHub Actions를 CI/CD 도구로 사용하고 있습니다. 

Google Cloud와 Play Console 정책은 수시로 변경되고 있습니다. 특히 Play Console의 배포 절차와 권한 관리 방식은 과거와 달라진 부분이 꽤나 많습니다.

_2025-08-31기준으로 정리_

<br>

# 1. 사전 준비 사항 - _Google Oauth 2.0 AccessToken 발급 환경 구성_

<br>

Play Developer API의 [Edits](https://developers.google.com/android-publisher/edits?hl=ko)를 통해 업로드 하기 위해서는 `androidpublisher`의 scope를 획득해야 합니다.

`androidpublisher` scope는 Google Play Google Play Developer Publishing API에 접근할 때 사용하는 `OAuth 2.0 권한`입니다.

**이걸로 할수 있는것**

- 앱 번들 / APK 업로드 
- 스토어 등록정보 수정
- 트랙 승급 및 버전 관리
- 구독 관리
- 기타 거의 모든 Play Console 기능들
 
>[Google API Oauth 2.0 scope](https://developers.google.com/identity/protocols/oauth2/scopes?utm_source=chatgpt.com&hl=ko)

### 1.1 프로젝트 및 서비스 계정 생성

먼저 Google Cloud를 통해 프로젝트를 생성하거나 기존 프로젝트를 선택합니다.

`왼쪽 상단 탭` > `IAM 및 관리자` > `서비스 계정` 탭을 통해 서비스 계정을 생성합니다.


### 1.2 Workload Identity Federation 활성화

`워크로드 아이덴티티 제휴` 탭에 들어가 `워크로드 아이덴티티 풀`을 생성합니다.

`풀에 공급업체 추가`에  OIDC를 선택하고 발급자(iss) url을 `https://token.actions.githubusercontent.com`로 설정해줍시다.

`대상` 항목은 기본을 설정하고 아래의 주소를 기억해 놉니다.

```
projects/<id>/locations/global/workloadIdentityPools/pool/providers/<providerName>
 ```
이후 `google-github-actions/auth@v2`에서 `workload_identity_provider`항목에서 사용합니다.

`공급 업체 속성 구성`에서 `google.subject`에 `assertion.sub` 항목을 매핑합니다.(필수)
이후 필요한 속성을 매핑합니다.

![예시](/assets/posts/2025-08-31/1.png){: width="350" height = "350"}
_속성 맵핑 예시_

repository에 따라 검증하기 위해서는 `attribute.repository`에 `assertion.repository`를 매핑하고 속성 조건에 `attribute.repostiory=="<owner>/<repository>"`하면 됩니다.

> _Github Actions [OIDC 문서](https://docs.github.com/ko/actions/reference/security/oidc)에서 클레임을 확인 할 수 있습니다_

`이후 액세스 권한 부여`에 들어가 서비스 계정을 연결하고 속성조건을 추가합니다. 

<br>

**주의사항**

단순히 `subject=https://token.actions.githubusercontent.com `와 같이 너무 포괄적인 조건을 주면 인증이 거부될 가능성도 있습니다.

보통은 `assertion.repository`, `assertion.ref`, `assertion.aud` 등 세부 claim을 활용하여 특정 리포지토리와 브랜치에만 권한을 주는 것이 안전하고 안정적입니다.


<br>

### 1.3 Play Console에 서비스 계정 연결

Play Console에서 더이상 API access 항목을 지원하지 않아 직접 서비스 계정을 등록해야 합니다.

`사용자 및 권한`에서 생성한 서비스 계정을 추가하고 권한을 부여합니다.

트랙에 배포 수정, 스토어 정보 관리 등 필요한 권한을 추가하면 됩니다.

이제 생성한 Google Cloud 서비스 계정을 통해 Play Console에 접근 할 수 있는 Access Token을 발급받을 수 있습니다.

<br>

# 2. Github Actions에서 flutter 빌드

### 2.1 빌드 환경 구성

Github Actions를 사용한 flutter 빌드는 다음의 환경에서 동작합니다.
```
  - name: Setup JAVA 18 SDK
    uses: actions/setup-java@v4
    with:
        distribution: 'temurin'
        java-version: '18'

  - name: Setup Flutter SDK
    uses: flutter-actions/setup-flutter@v4
    with:
        channel: stable
        version: 3.29.2
```
>_[flutter-actions/setup-flutter@v4](https://github.com/flutter-actions/setup-flutter)_

<br>

### 2.2 서명키 생성하기
배포를 하기 위해서는 빌드 산출물에 대해 올바른 서명키가 필요합니다.

먼저 `app/android/app/build.gradle` 경로에서 서명 키 설정 파일 경로를 확인합니다.

```
## ~~~/andorid/app/buid.gradle

def keystorePropertiesFile = rootProject.file("key.properties")
```
여기서 rootProject위치는 flutter프로젝트가 아닌 flutter 내부의 android 패키지를 가리킵니다.

배포를 위해 발급받은 서명키파일의 `storepassword`와 `keypassword`를 repository secret으로 만들어줍니다.

암포화된 키파일 xxx.jks 를 base64로 인코딩하고 repository secret으로 만들어줍니다.

```
## working-directory: app 에서

## key파일 base64 decode
set -e
mkdir -p $HOME/.android
echo "${{ secrets.ANDROID_RELEASE_KEY_JKS_BASE64 }}" | base64 -d > $HOME/.android/release-key.jks
chmod 600 $HOME/.android/release-key.jks

## 키파일 검증
keytool -list -keystore $HOME/.android/release-key.jks -storepass "${{ secrets.ANDROID_STORE_PASSWORD }}"

## key.properties생성
cat > ./android/key.properties <<'EOF'
storePassword=${{ secrets.ANDROID_STORE_PASSWORD }}
keyPassword=${{ secrets.ANDROID_KEY_PASSWORD }}
keyAlias=upload
storeFile=/home/runner/.android/release-key.jks
storeType=pkcs12
EOF
```
이러한 형식으로 서명을 위한 키파일을 준비합니다.

### 2.3 Dependencies & Code Generating

현재 barlow app 프로젝트는 멀티 모듈로 이루어져 있어 각 패키지마다 독립적으로 `flutter pub get`을 통한 의존성 설정 및 `build_runner`를 통한 코드생성이 필요합니다.

현재 `melos`와 같은 자동화 도구를 사용하지 않고 있으므로 의존성에 따라 다음과 같은 순서로 진행합니다.

```
  - name: Install dependencies
    shell: bash
    run: |
        set -e
        for dir in app common core design_system features; do
        echo ">>> flutter pub get in $dir"
        (cd "$dir" && flutter pub get)
        done

  - name: Generating Injectable
    shell: bash
    run: |
        set -e
        (cd core && flutter pub get && flutter pub run build_runner build --delete-conflicting-outputs)
        (cd features && flutter pub get && flutter pub run build_runner build --delete-conflicting-outputs) 
        (cd app && flutter pub get && flutter pub run build_runner build --delete-conflicting-outputs)

  - name: Generate Native Splash
    shell: bash
    working-directory: app
    run: dart run flutter_native_splash:create
```


<br>

### 2.4 .aab 빌드

플레이스토어에 배포하기위해 bundle형식으로 빌드합니다.

각 브랜치마다 다른 트랙에 배포해야 하기 때문에 version_name의 suffix를 구분해서 변경합니다. 
예를들어 `1.0.1+31` 이 `release/*` 브랜치에 푸시되면
`1.0.1-rc+32` 로 빌드됩니다.

| branch    | suffix |
|-----------|--------|
| dev       | -alpha |
| main      |  없음  |
| release/* | -rc    | 
| hotfix | 없음   | 

<br>

빌드워크플로우는 다음과 같습니다.

```
[pupspec.yaml에서 version_name, version_code 추출]
    ↓
[version code 증가]
    ↓
[branch에 따른 suffix 확인]
    ↓
[증가된 version code와 해당 suffix로 빌드]
    ↓
[빌드 성공시 pubspec.yaml에 version code 증가]
    ↓
[변경된 pubspec.yaml commit & push]
    ↓
[아티팩트 업로드]
```
<br>
<br>

# 3. Play Console에 배포하기

artifact가 업로드가 성공하면 배포작업을 시작합니다.

| branch    | 트랙 | 변경사항 심사| 
|-----------|--------|----------|
| dev       |  내부테스트 | X
| main      |  프로덕션 | O
| release/* | 비공개체스트| O
| hotfix | 내부테스트| X

### 3.1 Google Oauth Access Token 발급

서비스 계정의 worload identity provider정보를 github의 repository secret으로 설정합니다. [항목참고](#1-사전-준비-사항---google-oauth-20-accesstoken-발급-환경-구성)

이후 `google-github-actions/auth@v2`를 통해 access token을 발급받을 수 있습니다.
```
# cd.yml
...
    - name: Authenticate to Google Cloud
      id: auth
      uses: google-github-actions/auth@v2
      with:
        workload_identity_provider: ${{ secrets.WORKLOAD_IDENTITY_PROVIDER }}
        service_account: <serviceAccountName>
        token_format: access_token
        access_token_scopes: https://www.googleapis.com/auth/androidpublisher
...
```
> _자세한 사용법은 [google-github-actions/auth@v2](https://github.com/google-github-actions/auth)로_

### 3.2 Edits를 통해 트랙에 업로드

빌드된 아티팩트를 브랜치별 해당 트랙에 업로드 합니다.

>_Play Developer API의 [Edits](https://developers.google.com/android-publisher/edits?hl=ko) 문서 참고_

```
[Edit 객체 생성]
    ↓
[Edit 객체에 .aab 파일 업로드]
    ↓
[브랜치에 따라 Track 설정]
    ↓
[Track에 Edit 할당]
    ↓
[Edit commit]
```

### 3.2.1 Edit 생성하기
```
EDIT_RESPONSE=$(curl -s -X POST \
        -H "Authorization: Bearer $ACCESS_TOKEN" \
        -H "Content-Type: application/json" \
        "https://androidpublisher.googleapis.com/androidpublisher/v3/applications/$PACKAGE_NAME/edits")
        
        echo "Full response: $EDIT_RESPONSE"
         
        EDIT_ID=$(echo "$EDIT_RESPONSE" | jq -r '.id')
```
<br>

### 3.2.2 .aab파일 업로드하기

빌드 파일이 작지 않으므로 `uploadType=reusamble`을 통해 업로드 합니다.

>_[Google API의 재개 가능한 업로드](https://cloud.google.com/storage/docs/resumable-uploads?hl=ko)는 네트워크가 불안정하거나 파일이 클 때 대용량 파일을 업로드할 수 있도록 지원하는 프로토콜입니다._

```
UPLOAD_URL="https://androidpublisher.googleapis.com/upload/androidpublisher/v3/applications/$PACKAGE_NAME/edits/$EDIT_ID/bundles?uploadType=resumable"
        
SESSION_URL=$(curl -s -D - -o /dev/null \
    -H "Authorization: Bearer $ACCESS_TOKEN" \
    -H "X-Upload-Content-Type: application/octet-stream" \
    -H "Content-Type: application/octet-stream" \
    -X POST "$UPLOAD_URL" \
    | grep -Fi Location | awk '{print $2}' | tr -d '\r')

echo "Got session URL: $SESSION_URL"

curl -X PUT \
    -H "Authorization: Bearer $ACCESS_TOKEN" \
    -H "Content-Type: application/octet-stream" \
    --data-binary @build-artifacts/app-release.aab \
    "$SESSION_URL"

VERSION_CODE=$(curl -s -X GET \
    -H "Authorization: Bearer $ACCESS_TOKEN" \
    "https://androidpublisher.googleapis.com/androidpublisher/v3/applications/$PACKAGE_NAME/edits/$EDIT_ID/bundles" \
    | jq -r '.bundles | max_by(.versionCode) | .versionCode')
echo "Uploaded bundle versionCode: $VERSION_CODE"
```
<br>

### 3.2.3 Track에 할당후 Commit
```
curl -s -X PUT \
-H "Authorization: Bearer $ACCESS_TOKEN" \
-H "Content-Type: application/json" \
-d "{\"releases\": [{\"versionCodes\": [\"$VERSION_CODE\"], \"status\": \"completed\"}]}" \
"https://androidpublisher.googleapis.com/androidpublisher/v3/applications/$PACKAGE_NAME/edits/$EDIT_ID/tracks/$TRACK"

curl -s -X POST \
-H "Authorization: Bearer $ACCESS_TOKEN" \
"https://androidpublisher.googleapis.com/androidpublisher/v3/applications/$PACKAGE_NAME/edits/$EDIT_ID:commit" 
```

<br>

# 마치며

## 전체 흐름 요약
![자동 빌드 및 배포](/assets/posts/2025-08-31/3.png){: width="500" height = "500"}
_CI/CD flow_

barlow app 프로젝트는 현재 `Github Actions`를 사용하여 위와 같은 형태로 자동 빌드 및 배포 파이프라인을 구축했습니다.

현재 프로젝트가 소규모로 진행되고 있는 만큼 간단하면서도 빠르게 구축할 수 있는 방식을 선택했습니다.

### 한계점?

- 빌드 시간 오래걸림
- 너무 github actions에 의존
- private repo로 변환시 돈내야됨
- 플랫폼 추가시(ios 혹은 web) 확장성 부재

이러한 제약이 존재하지만, 현재 규모에서는 충분히 괜찮은 방식으로 구성이라고 생각합니다. 결국 ios 빌드가 추가되는 순간 전체적으로 수정해야 하기 때문에 우선은 이 체계를 안정적으로 유지할 예정입니다.

