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


### 포스트 생성

포스트 생성은 새 파일을 커밋 + 푸시하는 것과 동일하다.
- owner : 저장소 주인 = 내 아이디

- repo : 저장소 이름
- path : 저장 경로
- message : 커밋 메시지
- content : 글 (텍스트 인코딩 필요)

path 는 어떻게 요청하느냐에 따라 폴더 두개를 뚝딱 생성할 수 있다.
`posts/direc1/direc2/title.md` 라면 저장소 폴더 3겹 안에 해당 파일이 생성된다.


```js
// POST (equest: NextRequest, { params }: PostPathParams)

const body = (await request.json()) as PostCreateRequestBody;
const { title, content } = body;

// 클라이언트에서 온 url parameter 로 저장경로 만들기
const dir = joinPath((await params).path);
// 저장경로에는 파일명과 확장자도 포함되어야 한다
const path = `posts/${joinPathWithExtention([dir, title])}`;
// 파일 내용 인코딩
const encodedContent = Buffer.from(content, "utf-8").toString("base64");

const { data } = await octokit.rest.repos.createOrUpdateFileContents({
  owner,
  repo,
  path,
  content: encodedContent,
  message: `create post: ${title}`,
});

```
클라이언트에서 `POST .../dir1/dir2/title` 요청이 온다면, 깃헙 저장소에는 dir1/dir2 경로에 title.md 파일이 저장되는 것이다.

### 포스트 조회

path 로 파일을 저장한 것처럼, git api 는 path 에 해당하는 파일을 반환한다.
이 기능들에서는 모든 속성 중에 path 값이 제일 중요하며, 거의 파일의 id 역할을 한다.

포스트(파일) 조회는 owner, repo, path 만 보내주면 된다.

```js
// GET (equest: NextRequest, { params }: PostPathParams)

const path = joinPathWithExtention((await params).path);
const { data } = octokit.rest.repos.getContent({
  owner,
  repo,
  path,
});
```

- 중첩 파라미터의 경우, handler 로 들어오는 params.path 는 문자열 배열이다. 따라서 합치고 확장자를 추가하는걸 꼭 해주어야 한다. 예시에서는 joinPathWithExtention 유틸이다.

- 저기서 얻어지는 data 는 매우매우 방대하다. 이후에 필요한 값만 반환해주었다. 
(title, content, sha ...)


### 포스트 수정 & 삭제

수정은 생성 코드랑 거의 동일하지만, 생성이나 조회처럼 path 값만으로는 불가능하다. 

최근의 파일 응답에서 온 `sha` 라는 값을 함께 넣어주어야 한다. 
깃허브는 버전 관리를 해주는 서비스이기 때문에, 수정 또는 삭제 = 최신 작업 id 를 가져와서 적용하기 라고 생각해야 한다.


#### 수정하기

```js
// PUT (equest: NextRequest, { params }: PostPathParams)

const body = (await request.json()) as PostCreateRequestBody;
// 클라이언트에서는 파일의 sha 를 넣어서 요청
const { title, content, sha } = body; 

// 클라이언트에서 온 url parameter 로 저장경로 만들기
const dir = joinPath((await params).path);
// 저장경로에는 파일명과 확장자도 포함되어야 한다
const path = `posts/${joinPathWithExtention([dir, title])}`;
// 파일 내용 인코딩
const encodedContent = Buffer.from(content, "utf-8").toString("base64");

await octokit.rest.repos.createOrUpdateFileContents({
  owner,
  repo,
  path,
  content: encodedContent,
  message: `update post: ${title}`,
  sha,
});

```

#### 삭제하기

```js
// DELETE(request: NextRequest, { params }: PostPathParams)
// 생략
await octokit.rest.repos.deleteFile({
  owner,
  repo,
  path,
  sha,
  message: `delete post: ${removePathExtention(fileData.name)}`,
});
```

