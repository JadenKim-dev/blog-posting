## 0. 배포 환경

때로는 원하는 앱을 subpath 하위 경로에 배포를 해야 할 때가 있다.
예를 들어서 다음과 같은 URL과 애플리케이션 간 매핑이 있을 수 있다.

`https://mysite.com/react` -> React 앱
`https://mysite.com/vue` -> Vue 앱

위 상황에서 nginx 단에 대한 설정은 아래와 같이 이루어질 수 있다.

```conf
server {
	listen 80;

	location /react/ {
		# React App에 대한 경로
		proxy_pass http://localhost:3000;
	}

	location /vue/ {
		# Vue App에 대한 경로
		proxy_pass http://localhost:4000;
	}
}
```

이 때, `https://mysite.com/react/post` 에 대한 요청이 들어온다면, nginx를 거치면서 subpath가 제외된 `/post` 경로에 대한 요청으로 React App에 전달 된다.

`https://mysite.com/react/post` -> `http://localhost:3000/post`  
(React App에 `/post` 경로의 요청이 전달됨)

`https://mysite.com/vue/user` -> `http://localhost:4000/user`  
(Vue App에 `/user` 경로의 요청이 전달됨)

위와 같이 각각의 subpath에 서로 다른 앱을 배포해야 하는 상황이라면, 어플리케이션 단에서 하위 경로에 대한 추가적인 처리를 해야 한다.  
CSR 방식으로 작동하는 React CRA 프로젝트를 배포할 때 조치 사항은 다음과 같다.

## 1. 빌드 커맨드 설정

CRA 프로젝트를 배포할 때에는 보통 npm run build로 프로젝트에 대한 빌드를 수행하고, serve를 이용하여 빌드한 파일들을 서빙하게 된다.

```script
serve -s build
```

이 때 CRA 프로젝트는 s 옵션을 사용해야 한다.  
s 옵션을 주고 서빙을 하면 등록되지 않는 페이지에 접근할 때, not found 에러를 발생시키는 대신 index.html 파일을 서빙해준다.

이와 같이 작동해야 하는 이유는, CRA 프로젝트는 라우팅을 처리할 때 모든 페이지에서 기본으로 index.html을 서빙한 후 각 페이지에서 필요로 하는 컴포넌트를 추가로 요청하는 식으로 작동하기 때문이다.  
따라서 모든 페이지에서 기본으로 index.html 파일을 불러오도록 설정이 되어야 한다.

> 특정 URL로 요청 시 엉뚱하게 index.html이 반환되었다면, 해당 경로에서 not found 에러가 발생했다고 생각하면 된다.

## 2. PUBLIC_URL 환경 변수 설정

PUBLIC_URL 환경 변수는 static 파일을 불러오는 경로를 정의한다.  
public 폴더 하위의 index.html 파일에는 다음과 같이 PUBLIC_URL 환경 변수를 이용하여 static 파일들에 대한 요청 경로를 설정하고 있다.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <link rel="icon" href="%PUBLIC_URL%/favicon.ico" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="theme-color" content="#000000" />
    <meta
      name="description"
      content="Web site created using create-react-app"
    />
    <link rel="apple-touch-icon" href="%PUBLIC_URL%/logo192.png" />
    <link rel="manifest" href="%PUBLIC_URL%/manifest.json" />
    <title>React App</title>
  </head>
  <body>
    <noscript>You need to enable JavaScript to run this app.</noscript>
    <div id="root"></div>
  </body>
</html>
```

위 파일 내용 중 다음 부분이 PUBLIC_URL을 사용하고 있다.

```html
<link rel="icon" href="%PUBLIC_URL%/favicon.ico" />
<link rel="apple-touch-icon" href="%PUBLIC_URL%/logo192.png" />
```

위와 같이 필요한 static 파일을 비롯한 빌드 파일들을 해당 경로에 요청하여 불러온다.  
따라서 PUBLIC_URL 환경 변수에는 해당 앱이 라우팅 될 subpath를 입력해서 설정해줘야 한다.

위의 예제에서는 .env에 다음의 내용을 추가하여 환경변수에 값을 넣어줄 수 있다.

```
PUBLIC_URL=/react
```

> npm run build 시 package.json에 homepage 값이 세팅되어 있으면, 해당 값이 .env의 PUBLIC_URL로 삽입되어 빌드된다.

이제 favicon.ico를 불러오기 위해서는 다음의 url로 요청을 보내게 된다.

```
https://mysite.com/react/favicon.ico
```

해당 url은 nginx를 거치면서 /react path가 제거된 /favicon.ico 경로로 요청이 전달된다.

```
https://mysite.com/react/favicon.ico -> nginx -> http://localhost:3000/favicon.ico
```

해당 요청에 대해서 React App에서는 정상적으로 favicon.ico를 서빙하게 된다.

> serve -s로 배포할 경우 static path에 대한 설정이 잘못되어서 이상한 path로 요청이 이루어지면, 404 에러가 발생하는 대신 엉뚱하게 index.html 파일이 서빙된다.  
> 특정 URL의 결과로 흰 화면이 표시된다면, network 탭에서 static 파일들에 대한 요청이 제대로 처리되었는지 확인해보자.  
> 위 부분이 혼동된다면 static path 설정 시에는 -s 옵션을 제거하고 배포해서 테스트하고, 설정을 마친 뒤에는 -s 옵션으로 배포를 진행하면 된다.

## 3. react-router-dom basename 설정

React CRA 프로젝트에서 라우팅 시에는 보통 react-router-dom을 사용한다.  
이 때 보통 다음과 같이 상대 경로로 라우팅 설정을 한다.

```jsx
import { BrowserRouter, Link } from "react-router-dom";

function App() {
  return (
    <BrowserRouter>
      <Link to="/post" />
      <Link to="/user" />
    </BrowserRouter>
  );
}
```

Link 컴포넌트는 `/post`, `/user` 로 이동하는 a tag를 생성한다.  
이 때 현재 nginx 라우팅 구조로는 해당 link가 정상 동작하지 못한다.  
각 링크가 기본 루트 도메인 기준으로 작동하기 때문이다.

```
/post 클릭 -> https://mysite.com/post
/user 클릭 -> https://mysite.com/user
```

subpath를 포함한 상태로 컴포넌트들이 생성되게 하기 위해서는 아래와 같이 BrowserRouter의 basename을 설정해야 한다.

```jsx
import { BrowserRouter, Link } from "react-router-dom";
function App() {
  return (
    <BrowserRouter basename={process.env.PUBLIC_URL}>
      <Link to="/post" />
      <Link to="/user" />
    </BrowserRouter>
  );
}
```

이렇게 하면 Link 컴포넌트들이 각각 subpath를 반영한 url로 연결하는 a tag들로 렌더링된다.

```
<Link to="/post"/> ::: <a href="/react/post">
(https://mysite.com/react/post 에 대한 링크)

<Link to="/user"/> ::: <a href="/react/user">
(https://mysite.com/react/user 에 대한 링크)
```
