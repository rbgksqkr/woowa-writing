# Suspense와 ErrorBoundary 를 활용하여 선언적으로 관심사 분리하기

# 비동기 에러 처리

사용자가 있는 프로덕트를 개발한다면 에러 처리는 필수다. 사용자가 서비스를 이용할 때는 개발자가 의도한 시점에 의도한 동작만을 하지 않기 때문에, 다양한 케이스에 대한 에러 핸들링이 필요하다. 

> 따라서 비동기 에러를 처리하는 방법에 대해 소개하고, 선언형으로 처리하면서 상태에 대한 관심사를 분리한 경험을 소개하려고 한다.

## 명령형으로 처리하기

에러를 명령형으로 처리하는 방법에 대해 먼저 알아보자.

비동기 호출에 대한 에러를 다룰 때 일반적으로 `try-catch` 문을 사용한다.

```tsx
const getUser = () => {
	try {
	  const response = await fetch(`URL`);
	  const data = await response.json();
	  
	  return data;
	  } catch(error) {
		  console.error(error);
		}
}
```

위의 형태를 리액트 함수 컴포넌트에서 사용하기 어렵다. 

함수 컴포넌트에서 비동기 에러를 핸들링하려면 useState와 useEffect를 활용해야 한다. 

따라서 API 호출부에선 사용처로 에러 처리를 위임하고, 나아가 UI로부터 로직을 분리해 커스텀 훅으로 분리한다면 아래와 같이 구현할 수 있다.

```tsx
const getUser = async (id: number) => {
  const response = await fetch(`URL`);
  const data = await response.json();
  return data;
};

const useUserInfo = (id: number) => {
  const [data, setData] = useState<UserInfo>();
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState("");

  useEffect(() => {
    const fetchUser = async () => {
      try {
        setIsLoading(true);
        const user = await getUser(id);

        if (!ignore) {
          setData(user);
        }

        setIsLoading(false);
        setError('');
      } catch (err) {
        const error = err as Error;
        setError(error.message);
      }
    };

    let ignore = false;
    fetchUser();
    return () => {
      ignore = true;
    };
  }, [id]);

  return { data, isLoading, error };
};
```

커스텀 훅으로 데이터 페칭 로직을 추출함으로써 컴포넌트에서의 데이터의 흐름이 명확해졌다.

```tsx
export const UserInfo = ({ id }: { id: number }) => {
  const { data, isLoading, error } = useUserInfo(id);

  if (isLoading) return <LoadingFallback />;

  if (error) return <ErrorFallback error={error} />;

  return (
    <div>
      <h1>name: {data?.name}</h1>
      <h2>Email: {data?.email}</h2>
    </div>
  );
};
```

### 정리

비즈니스 로직이 UI 로직과 분리되면서 컴포넌트가 깔끔해졌지만 몇 가지 문제점이 존재한다.

1. 커스텀 훅에서 반환하는 로딩 상태와 에러 상태에 따라 매번 컴포넌트 내에서 분기처리가 필요하다.
2. 비동기 호출이 여러 개일 경우 이에 대한 처리가 복잡해지고, 코드를 유지보수하기 어려워진다.

>비슷한 에러 핸들링 코드가 수많은 컴포넌트 내에 위치하는 게 적절한가?

## 선언형으로 처리하기

커스텀 훅으로 데이터 페칭 로직을 분리했지만, 컴포넌트에 동작 과정이 드러나면서 로직이 명령형으로 이뤄져 있다. 

isLoading이 true일 때 LoadingFallback을 반환하고, error가 있을 때 ErrorFallback을 반환하고, 성공 케이스일 때 원하는 데이터를 반환한다. 

해당 로직이 현재는 문제가 되지 않는다. 문제라고 느끼지 못할 수도 있다. 하지만 각각의 로딩 상태와 에러 상태에 따라 다르게 처리하거나, 문제가 발생했을 때 에러를 추적하기 어렵다. 
또한 컴포넌트 내에 상태에 따라 분기처리하는 로직이 있는 게 유지보수 관점에서 좋지 않다. 따라서, 선언형으로 처리하여 각각의 관심사별로 분리해보자.

### **Suspense**

React 18 부터 Suspense는 React.lazy 뿐만 아니라, 모든 비동기 작업을 처리할 수 있게 되었다. (코드, 데이터, 이미지 로드 등)

따라서, Suspense를 활용하면 명령형으로 처리하고 있던 비동기 로딩 상태를 선언형으로 처리할 수 있다.

Suspense로 비동기 호출이 발생하는 컴포넌트를 감싸면 로딩 상태일 때 fallback UI를 보여주고, 비동기 호출이 완료되면 자식 컴포넌트를 렌더링한다.

공식문서의 말을 빌리면 개념적으로 catch 문과 유사하지만 오류를 잡는 대신 일시 중지된 컴포넌트를 잡는다.

>Conceptually, you can think of `Suspense` as being similar to a `catch` block. However, instead of catching errors, it catches components "suspending".

## 비동기 데이터 렌더링 방식

1. **Fetch-on-render (fetch in useEffect)**

>컴포넌트를 빈 상태로 렌더링시키고 데이터를 가져오면 리렌더링하는 방식

- 컴포넌트 렌더링 → useEffect에서 데이터 페칭 시작 → 데이터 페칭 완료 시 리렌더링
- 각 컴포넌트는 useEffect 에서 데이터 페칭을 트리거
- 이 접근 방식은 종종 `waterfall` 로 이어짐

<img width="500" alt="image2" src="https://github.com/user-attachments/assets/a594380a-c0f5-4b82-bba7-107ab924f99b">

<img width="400" alt="image2" src="https://github.com/user-attachments/assets/3fa088fb-d025-4565-813d-fa5fdf7e4b47">


2. **Fetch-then-render**

>필요한 데이터를 모두 가져온 다음 렌더링하는 방식

- 필요한 모든 데이터 페칭 시작 (Promise.all) → 데이터 도착 시 렌더링
- 데이터가 도착할 때까지는 아무것도 할 수 없음

<img width="500" alt="image4" src="https://github.com/user-attachments/assets/31ec1a4d-f9c4-41f3-b2ad-81269b89e93d">

<img width="400" alt="image4" src="https://github.com/user-attachments/assets/047f20dc-7168-48c4-9030-a45500b61843">


3. **Render-as-you-fetch (Suspense)**

>data fetching과 동시에 화면 렌더링을 시작하고, 데이터를 비동기적으로 받아서 데이터가 준비되는 대로 컴포넌트를 렌더링하는 방식

- 데이터 페칭 시작 → 컴포넌트 렌더링 시작 → 데이터 도착 시 리렌더링
- Suspense를 사용하면 네트워크 요청을 시작한 직후에 렌더링 시작
- 필요한 데이터가 준비되면 컴포넌트 렌더링 재시도

<img width="500" alt="image6" src="https://github.com/user-attachments/assets/7da86dd8-de91-4d8e-9999-dc2c3641d0dd">

<img width="400" alt="image6" src="https://github.com/user-attachments/assets/121ce484-523f-417c-b15e-56363dce9957">



## **ErrorBoundary**

error 상황에 대한 처리를 ErrorBoundary에게 위임해보자.

ErrorBoundary는 하위 컴포넌트 트리의 어디에서든 **깨진 컴포넌트 트리 대신 폴백 UI를 보여주는 컴포넌트**다.

렌더링 도중 생명주기 메서드 및 그 아래에 있는 전체 트리에서 에러를 잡아낸다.

---

기본적으로 애플리케이션이 렌더링 도중 에러를 발생시키면 React는 화면에서 해당 UI를 제거한다.

이를 방지하기 위해 UI의 일부를 ErrorBoundary로 감싸면, 에러가 발생한 부분 대신 fallback UI를 표시할 수 있다.

React 공식문서에서 기본적으로 제공해주는 ErrorBoundary 클래스 컴포넌트다.

이를 커스텀하려면 추가적인 공부가 필요하고, 함수형 컴포넌트로 사용하고 싶다면 `react-error-boundary` 라이브러리를 설치하여 구현할 수 있다.

기본적인 fallback UI만 보여준다고 한다면 render함수에서 hasError가 true일 때 반환하는 JSX에 ErrorFallback 컴포넌트를 추가하면 된다.

```tsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    // 다음 렌더링에서 폴백 UI가 보이도록 상태를 업데이트 합니다.
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    // 에러 리포팅 서비스에 에러를 기록할 수도 있습니다.
    logErrorToMyService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      // 폴백 UI를 커스텀하여 렌더링할 수 있습니다.
      return <h1>Something went wrong.</h1>;
    }

    return this.props.children;
  }
}
```

ErrorBoundary를 구현하고 에러를 잡을 컴포넌트를 감싼다.

아래와 같이 감싸주기만 하면 TodoInfo에서 에러가 발생했을 때 ErrorBoundary의 fallback UI를 보여줄 수 있다.

```tsx
const TestApp = () => {
	return (
		<ErrorBoundary>
			<TodoInfo />
		</ErrorBoundary>
  )
}
```

React 공식문서에서는 아래와 같은 상황에서 ErrorBoundary 가 에러를 잡지 못한다고 설명한다.

비동기 에러를 못잡는 이유만 간단히 생각해보자.

비동기 로직은 콜스택이 비워진 다음 실행되는데, 비동기 로직에서 에러가 발생한다면 ErrorBoundary 경계 내에 위치하지 않게 되므로 에러를 잡지 못하게 되는 것이다.

<img width="500" alt="new1" src="https://github.com/user-attachments/assets/9f5cb0be-041b-4229-ba3e-840ebca55b61">


# **어떤 에러를 처리할 수 있을까?**

에러를 명령형, 선언형으로 처리하는 방법에 대해서 알아봤는데, 에러를 어떻게 처리하냐에 따라 사용자 경험에 큰 영향을 줄 수 있다. 다양한 에러 상태를 효과적으로 다루기 위해서 어떤 에러 종류들이 존재하는지 알아보자.

## **예상 가능한 에러 vs 예상 불가능한 에러**

**에러가 언제 어떻게 발생할 지를 예상할 수 있는지**에 대한 관점으로 에러를 바라볼 수 있다.

특정 시점에 발생한 에러를 예측하고 대비할 수 있는지를 기준으로 예상 가능한 에러와 예상 불가능한 에러로 나눌 수 있다.

### 예상 가능한 에러

예상 가능한 에러란 **애플리케이션** **실행 전에 개발자가 미리 예상하고 대응할 수 있는 에러**다.

해당 에러는 주로 `외부 환경` 이나 `사용자 입력` 에 의해 발생한다.

try-catch 문 또는 ErrorBoundary로 예상 가능한 에러를 처리할 수 있다.

> - **사용자 입력 오류** : 잘못된 이메일 형식 또는 잘못된 필드를 제출한 경우
> - **잘못된 페이지 접근 오류** : URL로 잘못된 경로에 접근하는 경우

권한이 없거나 잘못된 접근에 대한 에러를 상황에 맞게 처리할 수 있다.

401, 403 등의 HTTP status code 내에서도 에러 코드를 정의하여 다양하게 로직을 처리할 수 있다.

### 프로젝트 적용 예시

프로젝트에서는 폼 형식으로 제출하는 영역이 적어 잘못된 페이지를 접근하는 경우에 대해 처리하였다.

참여할 수 없는 방에 접근하는 경우, 좌측처럼 잘못된 링크에 접속했다는 안내 문구가 뜬다. 다시 서비스를 진행할 수 있도록 추가적인 가이드를 제공할 예정이다.

도메인만 같고 없는 페이지에 접근하는 경우, 우측처럼 페이지 이동 시 에러가 발생했다는 안내 문구와 홈화면으로 가는 가이드를 제공한다.

<img width="350" alt="new2" src="https://github.com/user-attachments/assets/77c30926-f348-47c4-b570-584eb13ee81a"> <img width="350" alt="new2-2" src="https://github.com/user-attachments/assets/25aa3ad4-490b-4ed4-ac85-34337c394682">
### 예상 불가능한 에러

개발자가 **통제할 수 없는 외부 요인이나 예측하기 힘든 상황에서 발생하는 에러**다.

서버 API로부터 전달받는 에러 중 500번대 에러를 예측할 수 없는 에러로 분류한다.

> - **네트워크 오류 :** 네트워크가 일시적으로  중단되거나 타임아웃이 발생하는 경우
> - **런타임 타입 오류 :**  서버 장애로 API 응답이 없거나 잘못된 형식의 데이터를 반환하는 경우

같은 내용도 다른 관점에서 바라보면 예상 가능한 에러에서 예상 불가능한 에러로 나눌 수 있다.

일시적인 네트워크 오류는 어느 정도 예상하여 개발 단계에서 처리할 수 있지만, 언제 어떻게 발생할 지를 예측할 수 없기 때문에 해당 기준에서는 예상 불가능한 에러로 분류하였다.

예상 불가능한 에러는 ErrorBoundary 를 활용해 전역 에러 핸들링을 하거나 Sentry 와 같은 모니터링 시스템을 통해 대응책을 마련할 수 있을 것이다.

### 프로젝트 적용 예시

예상 불가능한 에러를 ErrorBoundary를 활용하여 처리하였다.

런타임 에러와 API 에러를 잡는 ErrorBoundary를 각각 분리하였고, Sentry로 모니터링 시스템을 구축해 에러 단계를 구분하여 Discord로 알림이 오도록 설정하였다.

tanstack-query의 useQueryErrorResetBoundary를 활용하면 가장 가까운 QueryErrorResetBoundary 컴포넌트 하위에 있는 모든 쿼리 오류를 재설정한다.

현재는 일부만 fallback UI를 띄우는 상황이 없어서 별도로 설정하지 않아 기본값인 전역으로 설정되었다.

<img width="300" alt="new3" src="https://github.com/user-attachments/assets/7eae708f-4c93-425f-a6b2-395b93896b06"> <img width="500" alt="new3" src="https://github.com/user-attachments/assets/41ba7327-1f22-4dc1-8267-b102e2fc49bd">


## **해결 가능한 에러 vs 해결 불가능한 에러**

**에러 발생 후 사용자가 즉시 해결할 수 있는지**에 대한 관점으로 에러를 바라볼 수 있다.

### 해결 가능한 에러

사용자가 직접 해결하거나, 해당 에러에 대한 처리 로직이 구현되어 있어 복구할 수 있는 에러다.

에러가 발생하더라도 적절한 조치를 통해 프로그램의 정상적인 흐름으로 돌아갈 수 있다.

> - **권한 문제** : 인증 토큰이 만료되었을 때 로그인 화면 라우팅 또는 로그인 요청
> - **API 호출 실패** : 적절한 피드백을 제공하여 문제가 발생했음을 알리고 안내 메세지 출력

사용자가 서비스를 이탈하지 않도록 에러 상황을 해결할 수 있는 가이드를 제공한다.

사용자에게 액션을 가이드하지 않더라도 문제 상황을 알려줌으로써, 해결할 수 있는 상황인지를 사용자가 판단할 수 있도록 안내한다.

### 프로젝트 적용 예시

에러가 발생했을 때 사용자에게 알려야하는 에러라면 안내 메세지를 제공한다.

아래 예시는 투표 시간이 지난 후에 투표를 하여 발생한 에러를 모달로 안내하는 상황이다.

tanstack-query의 mutation에서 에러 핸들링 로직을 처리하여 에러가 전파되지 않도록 구현하였다.
<img width="300" alt="new4" src="https://github.com/user-attachments/assets/be969de7-7106-4120-8e14-79618dadb98f"> <img width="500" alt="new4" src="https://github.com/user-attachments/assets/693dddca-8a03-4b36-abfd-8a49711daaab">

### 해결 불가능한 에러

말그대로 사용자가 해결할 수 없는 에러다. 

서비스를 정상적으로 사용할 수 없는 상태로, 사용자에게 어떤 에러 상황인지 말해줘도 도움이 되지 않는 에러다.

> - **사용자 환경 문제** : 저사양 기기 또는 브라우저에서 동작하지 않는 코드가 포함되어 있는 경우

# **Suspense 는 어떻게 로딩 상태를 감지하여 처리하는가?**

**Suspense가 어떻게 비동기 호출을 감지하여 fallback UI를 대신 렌더링하는지 고민해보자.**

- 비동기 호출을 하는 자식 컴포넌트가 부모 컴포넌트에게 `무언가` 를 줘야 부모 컴포넌트인 Suspense가 이를 감지할 것이다.
- `무언가` 는 로딩 상태와 완료 상태를 모두 갖고 있어야 한다. 그래야 로딩 상태일 때 fallback UI를 렌더링하고, 완료 상태일 때 자식 컴포넌트를 렌더링할 것이다.
- 리액트는 비동기 작업을 처리하는 객체인 `Promise` 를 활용하여 Suspense에서 비동기 호출을 감지하도록 구현하였다.

> 핵심은 **Promise를 캐치**하고, **로딩 상태를 관리하는 컴포넌트를 구현**하는 것
>
> 1. 비동기 작업이 발생하는 즉시 Promise 던지기 (pending)
> 2. Suspense 내부에서 로딩 상태를 관리하고, 로딩 상태면 fallback, 아니면 children 반환
> 3. Promise가 resolve되면 children 반환

- Promise 객체는 pending, fulfilled, rejected 3가지 상태를 갖고 있기 때문에 로딩 상태에 대한 분기처리가 모두 가능하다.
- 비동기 호출을 시작할 때 Promise를 throw하여 Suspense가 감지하도록 한다.
- Promise가 resolve 됨 → React는 다시 컴포넌트를 렌더링 → useUserInfo는 캐시된 데이터를 반환 → children 렌더링.

```tsx
const useUserInfo = (id: number): UserInfo => {
  if (!cache[id]) {
    const promise = getUser(id).then((data) => {
      console.log("resolve promise");
      cache[id] = { data };
    });

    console.log("throw promise");
    cache[id] = { promise };
    throw promise;
  }

  if (cache[id].promise) {
    throw cache[id].promise;
  }

  console.log("return cache data", cache[id].data!);
  return cache[id].data!;
};

const UserInfo = ({ user }: { user: User }) => {
  console.log("UserInfo render");

  const user = useUserInfo(1);

  return (
    <div>
      <h1>name: {user.name}</h1>
      <h2>Email: {user.email}</h2>
    </div>
  );
};
```


<img width="500" alt="new5" src="https://github.com/user-attachments/assets/93078601-4dfb-40ea-b9f5-0de9e50ca819">

렌더링을 시작한 후 Promise 객체를 던져 Suspense가 이를 감지하고 fallback UI를 보여준다.

던진 Promise 객체가 resolve 되면 Suspense는 비동기 요청이 반환하는 데이터로 children을 렌더링한다.

따라서 Promise가 pending 상태일 때 fallback UI, fulfilled 상태일 때 children을 반환하여 로딩 상태의 관심사를 분리할 수 있게 되는 것이다.

# 래퍼런스

[https://jbee.io/articles/react/효율적인 프런트엔드 에러 핸들링](https://jbee.io/articles/react/%ED%9A%A8%EC%9C%A8%EC%A0%81%EC%9D%B8%20%ED%94%84%EB%9F%B0%ED%8A%B8%EC%97%94%EB%93%9C%20%EC%97%90%EB%9F%AC%20%ED%95%B8%EB%93%A4%EB%A7%81)

### suspense 내부 동작 원리

https://velog.io/@shinhw371/React-suspense-throw

https://maxkim-j.github.io/posts/suspense-argibraic-effect/

https://velog.io/@tap_kim/react-learn-suspense

### suspense 공식문서

[리액트 공식문서 - suspense](https://ko.react.dev/reference/react/Suspense)

[리액트 공식문서(legacy) - suspense](https://17.reactjs.org/docs/concurrent-mode-suspense.html)

[리액트 공식문서 - react 18 suspense new feature](https://ko.react.dev/blog/2022/03/29/react-v18#new-suspense-features)

https://github.com/reactjs/rfcs/blob/main/text/0213-suspense-in-react-18.md

### ErrorBoundary 공식문서

[리액트 공식문서 - error boundary로 렌더링 에러 잡기](https://ko.react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)

[리액트 공식문서(legacy) - ErrorBoundary](https://ko.legacy.reactjs.org/docs/error-boundaries.html)

[Use react-error-boundary to handle errors in React - Kent C. Dodds](https://kentcdodds.com/blog/use-react-error-boundary-to-handle-errors-in-react)
