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







