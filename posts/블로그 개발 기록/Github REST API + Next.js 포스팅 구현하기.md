<!--|tags:Next.js|-->
`Next.js` 와 `Github REST API` 를 활용하여 포스트 생성/수정/삭제/조회 기능을 구현한 부분을 기록하고자 한다.

---

## [`Octokit`](https://docs.github.com/ko/rest/guides/scripting-with-the-rest-api-and-javascript?apiVersion=2022-11-28)

- Github REST API 요청을 간단하게 만들어주는 라이브러리이다.
- 요청 url 을 ".../github/api/...." 이렇게 직접 다 작성하지 않아도 되며, response 타입과 requestBody 타입이 지정되어 있기 때문에 개발시간이 매우 단축되었다.

```js
// libs/octokit.js
const octokit = new Octokit({
  auth: process.env.GITHUB_TOKEN,
});
```

- 위와 같이 인스턴스를 생성한 후, 원하는 엔드포인트에 맞게 호출해주면 된다.

```js
// 레포지토리 파일을 생성 또는 수정
const response = await octokit.rest.repos.createOrUpdateFileContents({...})
```
- 블로그 코드와 포스트 문서를 분리하고 싶었기 때문에, .md 파일로 직접 페이지를 만드는 방법이 아니라 원격 repo 로 데이터를 요청하고 받아오는 방식을 사용하였다. 
- (*next.js 에서는 .md 파일로 페이지 구성하는 것에 대해 아주아주 친절하다!*)

#### 포스트 저장소에 대한 시행착오들..

- discussion : repository 의 discussion 을 가져오는 방법이 아무리 찾아도 없었다. organization/team 의 discussion 을 가져오는 api 가 있긴 했지만, 지금은 없어진 기능이라고 한다.
- issue : 이모지, 코멘트 기능을 차후에 추가하기에는 적합하다. 하지만 포스트를 파일로 저장할 수 없다는 단점이 있다. 카테고리 나누는 방법도 생각해내기 어려웠다..!
- 따라서, repositry 에 포스트를 .md 확장자로 저장하고, 폴더로 카테고리를 나누기로 결정했다. 

---

## `Next.js Route Handler`
 - 사용하는 각종 토큰들, 상당히 날것인 깃헙 응답 등등을 브라우저에 노출하지 않기 위해 .. 
Next.js API 라우트 기능을 활용하여 서버 로직을 작성하였다.

- 게다가, github 는 블로그 전용 서비스가 아니기 때문에 파싱이나 인코딩/디코딩 작업이 꽤 이루어져야 했다. 이러한 로직들을 화면과 분리하고 싶었다.

```js
async function handleGet(req, res) {
  const { path } = req.query;
  try {
    const response = await octokit.rest.repos.getContent({
      owner: process.env.GITHUB_REPO_OWNER,
      repo: process.env.GITHUB_REPO,
      path,
    });
    const content = Buffer.from(response.data.content, 'base64').toString('utf8');
    res.status(200).json({ content });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
}
```