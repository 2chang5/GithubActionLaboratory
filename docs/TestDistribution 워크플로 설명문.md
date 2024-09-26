# Android CI 내 복잡한 내용을 정리한 파일입니다.

## Job/ReleaseNoteInputByIssue

이슈 본문내 텍스트 값을 추출하여 다음 Job로 전달해야하는 역할입니다.

-> 텍스트값 전달을 위해 outputs를 이용해서 다음 Jobs로 전달합니다.

output로 정의한 내용:

- tester_group / GetTesterGroup 스텝에서 값 주입 
- release_note / GetReleaseNote 스텝에서 값 주입



### step / GetWholeIssueString

```yml
WHOLE_ISSUE=$(jq -r '.issue.body' < $GITHUB_EVENT_PATH)
```

jq는 Json 데이터를 처리하는 도구입니다.

GITHUB_EVENT_PATH라는 경로로 현재 이벤트의 JSON 데이터가 내려옵니다.

제이슨을 객체 변환했을때 issue.body내 이슈의 본문이 내려오기 때문에 해당 값을 접근합니다.

-> https://github.com/2chang5/GithubActionLaboratory/blob/main/.github/workflows/CheckPayload.yml

이 워크 플로 실행시켜서 액션 돌려보면 Json 페이로드를 확인할 수 있습니다.

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



-r 옵션: Raw Output옵션으로 제이슨 형식이 아닌 순수 텍스트 형식으로 반환하겠다는 옵션입니다.



### step / GetTesterGroup

```yml
TESTER_GROUP=$(echo "${{ steps.get_whole_issue.outputs.whole_issue }}" | sed -n 's/^- *테스터:[[:space:]]*\(.*\)[[:space:]]*$/\1/p')
```

sed는 텍스트 스트림을 편집 및 변환하는 스트림 편집기 입니다.

나머지는 sed의 문법을 참고해주세요



## Job / BuildAAB

### step / checkout code
기본적으로 checkout이란 깃에서 제공하는 환경에 저장소에 있는 파일들을 내려받는 개념이라고 보면됩니다.
이때 with을 통해서 설정을 부여할 수 있습니다.

#### persist-credentials

actions/checkout이 실행될 때 사용하는 GitHub 액세스 토큰을 체크아웃 이후에도 git 자격 증명으로 유지할지 여부를 결정하는 옵션입니다.
즉 해당 설정을 false로 하면 이후 step에서 git 관련 명령어를 수행할때 추가 인정정보가 필요합니다.

보안상의 이유로 설정하는 옵션인데 check out이후 step에서 git관련 명령어를 사용하지 않는 경우 false로 설정하는것이 좋습니다.

기본 값은 true입니다.

### step / gradle cache

캐싱은 안드로이드진영에서 많이쓰이는 Gradle 캐시를 기본적으로 채용했습니다. 

만약 필요하다면 고도화가 필요할것 같습니다.

[참고 블로그](https://kotlinworld.com/399) -> 해당 블로그에서 설명을 참고하시면 됩니다.



### step / set up JDK 8

사용할 자바를 설정하는것으로 안드로이드 에서 사용하는 jdk를 세팅해주면 됩니다.

gradle의 경우 캐싱할거니 Gradle 빌드 캐시 활성화 하는 설정인  ```cache: gradle``` 를 설정해주었습니다.



### step / grant execute permission for gradlew

gradlew 파일에 실행 권한을 부여하는 설정을 해주는 스텝입니다.

Git 저장소에 커밋된 파일의 권한은 플렛폼에 따라 다르게 동작할 수 있다고 합니다.(맥, 윈도우, 리눅스 -> 가끔 sudo입력해야 되는 상황 같은거)

그래서 실행 권한을 부여하기 위해 거쳐야하는 step 입니다.



### step / Build Android AAB

AAB 파일 빌드하는 step 입니다.

기본적으로 구글링 해보 ```run: ./gradlew assembleDebug --stacktrace```을 많이 사용하는데 해당 명령어의 경우 apk를 뽑는 명령어로 현재와 같은 형태를 띄도록 변경하였습니다.

추후 flavor관련 부분 테스트 및 변경이 필요합니다. 



### step / Upload AAB as artifact

AAB 파일을 다음 Job로 전달하기위한 step 입니다.

 Job간 정보 전달은 output, artifact가 있습니다.

- output: 데이터 전달이지만 텍스트 값만 전달가능(빌드파일 등은 불가)
- artifact: 워크플로우가 실행되는 동안 생성된 파일이나 디렉토리를 말합니다. 파일등을 전달할때 유용하며 워크플로우간 공유, 내용물 github을 통해 90일간 다운로드도 가능합니다.

aab 생성경로를 path로 넣어주어 해당 파일을 aabFile이라는 이름으로 저장합니다.

