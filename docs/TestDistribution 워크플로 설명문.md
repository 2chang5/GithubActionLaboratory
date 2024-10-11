# TestDistribution 내 복잡한 내용을 정리한 파일입니다.

## 트리거 조건

issue 생성시(activity type : opened)

-> 현재 한캐의 상황을 고려하여 트리거 조건을 선정하였습니다.



## Job/CheckTrigger

이슈에 "테스트_배포" 라벨이 붙어있는지 조건 검사하여 해당 워크플로우를 동작시킬지 결정합니다.

최초 시작점이므로 라벨조건이 맞지 않는 경우 후속작업이 모두 Skipped 상태로 처리되도록 작업하였습니다.

이벤트 트리거 정보가 내려오는 json 데이터값에서 labels.name값을 추출 내부에 원하는 라벨이 있는지 확인합니다.



## Job/ReleaseNoteInputByIssue

이슈 본문내 텍스트 값을 추출하여 다음 Job로 전달해야하는 역할입니다.

-> 텍스트값 전달을 위해 outputs, artifact 를 이용해서 다음 Jobs로 전달합니다.

output로 정의한 내용:

- tester_ group / get tester group 스텝에서 값 주입 
- branch_name / get branch name 스텝에서 값 주입

output 사용 이유: 한줄짜리 데이터는 txt 파일로 전달할 경우 원치않는 공백 혹은 줄바꿈이 들어가 추가 처리가 필요하여 불필요한 작업이 늘어납니다.

이에 output을 이용하여 간단하게 처리하였습니다.



artifact로 정의한 내용:

- release_note / upload release note as artifact 스텝에서 텍스트 파일 주입

artifact 사용 이유: 줄바꿈이 있는 데이터의 경우 output으로 처리가 불가능합니다. -> 한줄 짜리만 처리가능 이에 txt파일 형태로 전달할 수 있도록 artifact를 이용하였습니다.



### step / get whole issue string

```yml
WHOLE_ISSUE="${{ github.event.issue.body }}"
```

트리거에 대한 정보가 Json 형태로 전달됩니다.[관련 공식문서](https://docs.github.com/ko/actions/writing-workflows/choosing-when-your-workflow-runs/triggering-a-workflow#viewing-all-properties-of-an-event)

제이슨을 객체 변환했을때 issue.body내 이슈의 본문이 내려오기 때문에 해당 값을 접근합니다.

제이슨 값 볼수 있는 워크플로우

- [제이슨 형식에 맞는 버전](https://github.com/2chang5/GithubActionLaboratory/blob/main/.github/deprecatedworkflows/CheckPayload.yml)
- [공식문서에서 제시한 방법](https://github.com/2chang5/GithubActionLaboratory/blob/main/.github/deprecatedworkflows/CheckPayloadByDocs.yml)

트리거 정보를 통해 분기등의 작업이 필요하다면 해당 워크플로우를 실행하여 참고 부탁드립니다.

예시)

```json
{
  "action": "labeled",
  "issue": {
    "active_lock_reason": null,
    "assignee": null,
    "assignees": [],
    "author_association": "OWNER",
    "body": "- 테스터: cashwalk-2\r\n- 출시노트:\r\n```\r\n#702\r\n[동네 돼지-뼈탄집 가서 삼겹살 조지게 먹기]\r\n[동네 산책-좀비짐 산책하고 소감문 쓰기]\r\n```\r\n",
.... 훨씬 엄청 많은 값 나옴 필요하면 액션 돌려보자.
```



### step / get branch name 

빌드할 브랜치를 이슈 본문 내용으로부터 추출하는 step 입니다.

```yml
BRANCH_NAME=$(echo "$WHOLE_ISSUE" | sed -n 's/^- *브랜치:[[:space:]]*\(.*\)[[:space:]]*$/\1/p')
```

sed는 유닉스 계통의 텍스트 파일 조작 도구 입니다.

브랜치 명을 입력시 휴먼에러를 방지라기 위해 입력한 브랜치 앞뒤에 공백을 제거하는 코드를 추가하였습니다.

줄바꿈이 필요없어 출력을 output으로 설정하였습니다.



### step / get tester group

테스터 그룹을 이슈 본문 내용으로부터 추출하는 step 입니다.

```yml
TESTER_GROUP=$(echo "$WHOLE_ISSUE" | sed -n 's/^- *테스터:[[:space:]]*\(.*\)[[:space:]]*$/\1/p')
```

테스터 그룹을 입력시 휴먼에러를 방지라기 위해 입력한 브랜치 앞뒤에 공백을 제거하는 코드를 추가하였습니다.

줄바꿈이 필요없어 출력을 output으로 설정하였습니다.



### step / get release note

릴리즈 노트를 이슈 본문 내용으로부터 추출하는 step입니다.

```yaml
RELEASE_NOTE=$(echo "$WHOLE_ISSUE" | awk '/^- 출시노트:/ {flag=1; next} flag && /^+++++/ {count++; next} count==1 {print} count==2 {exit}')
```

awk 또한 유닉스 계통의 텍스트 파일 조작도구입니다.

sed대신 awk를 채택한 이유는 여러줄의 줄바꿈이 있는 텍스트의 경우 awk가 처리하기 더 수월하여 채택하였습니다.

+++++ 를 경계로 릴리즈 노트를 추출해 냅니다.

경계지점을 +++++로 나눈 이유는 특수문자중 예약어로 사용되지 않는 것들 중 경계를 나누는데 제일 명확하여 사용하였습니다.



### step / upload release note as artifact

추출한 릴리즈노트 txt파일을 job간 공유를 위하여 artifact로 업로드하는 step 입니다.

artifact로 업로드 하여 추후 일정 기간 동안 github action을 통하여 파일을 다운 및 확인 또한 가능합니다.



## Job / BuildAAB

### step / checkout code
기본적으로 checkout이란 깃에서 제공하는 환경에 저장소에 있는 파일들을 내려받는 개념이라고 보면됩니다.
이때 with을 통해서 설정을 부여할 수 있습니다.

#### persist-credentials

actions/checkout이 실행될 때 사용하는 GitHub 액세스 토큰을 체크아웃 이후에도 git 자격 증명으로 유지할지 여부를 결정하는 옵션입니다.
즉 해당 설정을 false로 하면 이후 step에서 git 관련 명령어를 수행할때 추가 인정정보가 필요합니다.

보안상의 이유로 설정하는 옵션인데 check out이후 step에서 git관련 명령어를 사용하지 않는 경우 false로 설정하는것이 좋습니다.

기본 값은 true입니다.

#### ref

어떤 브랜치의 코드를 내려받을지 지정해주는 옵션입니다.

ReleaseNoteInputByIssue/get branch name 스텝에서 추출한 브랜치 명을 이용하여 브랜치를 설정합니다.



### step / gradle cache

캐싱은 일반적으로 안드로이드 진영에서 많이쓰이는 Gradle 캐시를 기본적으로 채용했습니다. 

만약 필요하다면 고도화가 필요할것 같습니다.

[참고 블로그](https://kotlinworld.com/399)를 참고하였습니다.



### step / set up JDK 17

사용할 자바를 설정하는것으로 안드로이드 에서 사용하는 jdk를 세팅해주면 됩니다.

Gradle 빌드 캐시 활성화 하는 설정인  ```cache: gradle``` 를 설정해주었습니다.



### step / grant execute permission for gradlew

gradlew 파일에 실행 권한을 부여하는 설정을 해주는 스텝입니다.

Git 저장소에 커밋된 파일의 권한은 플렛폼에 따라 다르게 동작할 수 있다고 합니다.(맥, 윈도우, 리눅스 -> 가끔 sudo입력해야 되는 상황)

이에 실행 권한을 부여하기 위해 거쳐야하는 step 입니다.



### step / Create google-services.json

숨김 파일인 google-services를 생성해서 깃헙액션 환경에 추가하는 step입니다.

해당 값은 base64로 인코딩되어 secrects에 저장 되어있습니다.



### step / Build Android AAB(테스트 환경에서는 APK)

AAB 파일 빌드하는 step 입니다.

현재는 ```./gradlew assembleDebug --stacktrace``` 명령어를 이용하여 APK를 생성합니다.(테스트환경의 경우 google play store에 등록되지않아 현재 aab로 firebase appDistribution을 이용할 수 없습니다.)

추후 한캐에 적용하는 경우에는 ```./gradlew bundleDebug``` 명령어를 이용하여 AAB를 추출하여 사용할 예정입니다.

또한 flavor관련 부분 테스트 및 변경이 필요합니다. 



### step / Upload AAB as artifact(테스트 환경에서는 APK)

AAB 파일을 다음 Job로 전달하기위한 step 입니다.

 Job간 정보 전달은 output, artifact가 있습니다.

- output: 데이터 전달이지만 텍스트 값만 전달가능(빌드파일 등은 불가)
- artifact: 워크플로우가 실행되는 동안 생성된 파일이나 디렉토리를 말합니다. 파일등을 전달할때 유용하며 워크플로우간 공유, 내용물 github을 통해 90일간 다운로드도 가능합니다.

artifact를 이용하여 파일을 전달하도록 하였습니다.

aab 생성경로를 path로 넣어주어 해당 파일을 apk_file이라는 이름으로 저장합니다.

-> name이 해당 파일에대한 구분할수있는 요소로 작용합니다.



## Job / DistrubutionByFirebaseAppTester

기본 조건:

하기 두가지 job에 종속적

- ReleaseNoteInputByIssue
- BuildAAB

필터: BuildAAB가 성공시에만 동작 



### step / download AAB artifact

빌드된 aab를 다운받는 과정입니다.



### step / download release note artifact

추출된 릴리즈 노트 파일을 다운받는 과정입니다.



### step /  upload to firebase app distribution

```
wzieba/Firebase-Distribution-Github-Action@v1.7.0
```

해당 명세에 맞춰서 파이어 베이스 배포를 위한 정보를 넣어주었습니다.

🚨 주의사항 

기존 token으로 간단하게 설정하던 값이 serviceCredentialsFileContent로 변경되었습니다. ([마이그레이션 문서](https://github.com/wzieba/Firebase-Distribution-Github-Action/wiki/FIREBASE_TOKEN-migration))

개인계정으로 해당 값을 뽑으면 추후 문제가 있을 수 있어 파이어베이스에 접근 가능한 공용계정을 통해 해당 값을 추출하는것이 좋을것이라 판단됩니다.



# 한캐 적용시 고려 사항

1. Grade.properties(Global properties) 파일 생성 및 적용 / 신규 step
2. aab로 빌드하도록 변경 / Build Android AAB
3. Buildflavor 관련 적용 및 테스트 / Build Android AAB
4. aab파일명 단순화하도록 gradle 파일 변경 혹은 생성된 파일명을 넣도록 수정 / Build Android AAB
5. google cloud 공용계정 생성 / upload to Firebase App Distribution
