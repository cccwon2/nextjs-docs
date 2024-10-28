**React Query**는 서버 상태 관리를 쉽게 하기 위한 라이브러리로, **서버에서 데이터를 가져오고, 캐싱하고, 갱신하고, 동기화**하는 작업을 매우 간단하게 만들어 줍니다. React Query는 특히 **데이터 페칭, 캐싱, 동기화**에 특화되어 있으며, 이를 통해 개발자가 API 통신을 더욱 효율적이고 깔끔하게 처리할 수 있습니다.

서버에서 데이터를 가져오고 유지할 때 생기는 여러 가지 문제를 손쉽게 해결해 주는 도구로, **쿼리 캐싱, 자동 재요청, 페이지네이션, 데이터 리프레시** 등의 기능을 지원합니다. 아래에서 React Query의 특징, 사용법, 장점, 단점 등을 자세히 설명하겠습니다.

### 1. **React Query의 주요 개념과 특징**

React Query의 가장 큰 강점은 **데이터 페칭과 캐싱**을 단순하게 처리해주는 것입니다. React Query를 통해 다음과 같은 개념들을 이해하고 사용할 수 있습니다.

#### a. **Query**

- **쿼리(Query)**는 특정 데이터를 가져오는 **비동기 요청**을 의미합니다. React Query는 `useQuery` 훅을 통해 데이터를 가져옵니다.
- **자동 갱신**: 쿼리 데이터를 자동으로 **갱신(refresh)** 하거나 **캐싱**된 데이터를 재사용할 수 있습니다.
- 데이터를 가져오거나 특정 조건에서 갱신을 트리거할 때 유용하며, 쿼리는 **고유한 키**로 식별됩니다.

#### b. **Mutation**

- **뮤테이션(Mutation)**은 데이터를 **변경**하거나 **수정, 삭제** 등의 요청을 수행할 때 사용됩니다. React Query는 `useMutation` 훅을 통해 이러한 작업을 제공합니다.
- 성공적인 뮤테이션 이후에 특정 쿼리를 자동으로 **갱신**하도록 설정할 수도 있습니다.

#### c. **캐싱 및 자동 동기화**

- **캐싱**: React Query는 가져온 데이터를 **자동으로 캐싱**하며, 동일한 키로 쿼리 요청이 들어오면 **캐시된 데이터를 재사용**합니다. 이를 통해 불필요한 네트워크 요청을 줄이고 성능을 최적화할 수 있습니다.
- **자동 동기화**: 백그라운드에서 **자동으로 데이터를 갱신**하며, 데이터가 변경되었을 가능성이 있는 상황에서 최신 데이터로 동기화하여 항상 최신 정보를 유지하도록 합니다.

#### d. **리트라이와 폴링**

- **자동 리트라이**: 네트워크 오류 발생 시 자동으로 데이터를 재시도합니다. 리트라이 횟수와 지연 시간 등을 설정할 수 있어, 네트워크 불안정 상황에 유연하게 대처할 수 있습니다.
- **폴링(Polling)**: 주기적으로 데이터를 다시 가져오는 **폴링** 기능을 통해 특정 주기마다 서버로부터 최신 데이터를 가져올 수 있습니다.

### 2. **React Query 사용 예시**

React Query의 기본적인 사용 예제를 통해 데이터 페칭을 어떻게 수행하는지 보여드리겠습니다.

#### a. **Query를 사용한 데이터 페칭**

```tsx
import { useQuery } from "react-query";
import axios from "axios";

const fetchUser = async (userId: string) => {
  const response = await axios.get(`https://api.example.com/users/${userId}`);
  return response.data;
};

function UserProfile({ userId }: { userId: string }) {
  // useQuery 훅을 사용하여 데이터 가져오기
  const { data, isLoading, error } = useQuery(["user", userId], () => fetchUser(userId));

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error loading user data</div>;

  return (
    <div>
      <h1>{data.name}</h1>
      <p>{data.email}</p>
    </div>
  );
}
```

#### **설명**

1. **`useQuery` 훅 사용**: `useQuery`는 두 개의 주요 인자를 받습니다.

   - **쿼리 키** (`['user', userId]`): 데이터를 고유하게 식별하기 위한 키입니다. 이 키는 React Query가 캐시 데이터를 구분하는 데 사용됩니다.
   - **데이터 가져오기 함수** (`fetchUser`): 데이터를 가져오는 비동기 함수입니다. React Query는 이 함수를 호출해 데이터를 페칭합니다.

2. **쿼리 상태 관리**: `useQuery`는 `isLoading`, `error`, `data` 등의 상태를 관리하여 데이터를 요청하는 동안 로딩 상태와 오류 발생 시의 상태를 쉽게 처리할 수 있도록 합니다.

#### b. **Mutation을 사용한 데이터 변형**

```tsx
import { useMutation, useQueryClient } from "react-query";
import axios from "axios";

const updateUser = async (user: { id: string; name: string }) => {
  await axios.put(`https://api.example.com/users/${user.id}`, user);
};

function UpdateUserForm({ user }: { user: { id: string; name: string } }) {
  const queryClient = useQueryClient();

  // useMutation 훅을 사용하여 데이터 변형
  const mutation = useMutation(updateUser, {
    // 뮤테이션이 성공했을 때 캐시된 데이터를 갱신
    onSuccess: () => {
      queryClient.invalidateQueries(["user", user.id]);
    },
  });

  const handleSubmit = () => {
    mutation.mutate({ id: user.id, name: "New Name" });
  };

  return (
    <div>
      <button onClick={handleSubmit}>Update User</button>
    </div>
  );
}
```

#### **설명**

1. **`useMutation` 훅 사용**: 데이터를 수정, 추가, 삭제 등 변형할 때 사용하는 훅입니다.
   - **`updateUser`** 함수는 사용자 데이터를 수정하는 요청을 담당합니다.
2. **`onSuccess` 콜백**: 데이터 변형이 성공했을 때 특정 쿼리를 **무효화** (`invalidateQueries`)하여 캐시된 데이터를 갱신하도록 합니다. 이로 인해 사용자는 최신의 데이터로 UI가 갱신된 것을 보게 됩니다.

### 3. **React Query의 주요 기능들**

- **캐시 타임과 스테일 타임**:

  - **캐시 타임(cacheTime)**: 쿼리 데이터가 메모리에 **얼마나 오래 유지**되는지를 제어합니다. 기본값은 5분입니다.
  - **스테일 타임(staleTime)**: 쿼리 데이터가 "신선한" 상태로 간주되는 시간을 설정합니다. 스테일 타임이 지나면 React Query는 데이터를 **백그라운드에서 자동 갱신**하려고 시도합니다.

- **백그라운드 데이터 리패칭**:
  - **포커스 리패칭**: 사용자가 페이지를 다시 방문하거나 브라우저 탭을 포커싱할 때 데이터를 다시 가져옵니다.
  - **네트워크 재연결 리패칭**: 네트워크 연결이 다시 활성화되면 데이터를 다시 가져오는 기능을 제공합니다.

### 4. **React Query의 장점**

- **자동 캐싱 및 데이터 재사용**: 동일한 쿼리에 대한 요청이 중복되지 않도록 데이터를 캐싱하고 필요시 캐시된 데이터를 재사용하여 성능을 최적화합니다.
- **데이터 일관성 유지**: 특정 쿼리를 **무효화(invalidate)** 하면 자동으로 최신 데이터를 가져와 캐시를 갱신해줍니다.
- **상태 관리 간소화**: 데이터의 로딩, 오류, 성공 상태를 자동으로 관리해 주기 때문에 별도의 상태 관리 코드가 줄어들어 개발 생산성을 높여줍니다.
- **복잡한 데이터 갱신 로직 간편화**: `onSuccess`, `onError`, `onSettled` 등과 같은 뮤테이션 훅의 콜백을 통해 특정 조건에서 자동으로 데이터 갱신을 쉽게 처리할 수 있습니다.

### 5. **React Query의 단점**

- **초기 학습 곡선**: React Query의 API와 여러 가지 옵션들을 처음 접할 때는 약간의 **학습 곡선**이 있습니다. 특히 쿼리 키 관리와 캐싱 전략을 이해하는 데 시간이 걸릴 수 있습니다.
- **추가적인 의존성**: React Query를 프로젝트에 추가하면 코드가 React Hook 기반으로 설계되어 있더라도 약간의 의존성이 추가되는 셈입니다.
- **프로젝트 크기에 따른 필요성**: 아주 간단한 프로젝트에서는 React Query를 도입하는 것이 오히려 과할 수 있습니다. 복잡하지 않은 API 호출에서는 상태 관리 라이브러리 없이 처리하는 것이 더 간단할 수 있습니다.

### 6. **React Query의 사용 시기**

- **데이터 중심**의 애플리케이션에서 특히 유용합니다. 자주 데이터를 **갱신하거나** 여러 곳에서 **동일한 데이터**를 사용해야 할 때 효과적입니다.
- **리모트 데이터** (서버에서 가져오는 데이터) 관리가 필요할 때 매우 효율적입니다.
- **자동 동기화**가 필요하거나 데이터를 **백그라운드에서 갱신**할 필요가 있는 경우 React Query는 매우 큰 이점을 제공합니다.

### 7. **React Query와 비교할 수 있는 도구들**

- **Redux**: Redux는 클라이언트 상태(로컬 상태)에 집중하고 서버 데이터 관리에는 부적합합니다. 서버 상태를 관리하려면 추가적인 설정(RTK Query)이 필요합니다.
- **Apollo Client**: GraphQL을 사용하는 경우 Apollo Client가 좋은 선택일 수 있습니다. 그러나 RESTful API의 경우 React Query가 더 가볍고 간단하게 사용할 수 있습니다.

### **결론**

React Query는 **서버 상태 관리**를 위한 강력한 도구로, **데이터의 자동 캐싱, 갱신, 동기화** 등의 기능을 통해 API 데이터를 더욱 효율적으로 처리하게 해줍니다. 특히 자주 데이터가 변경되거나 많은 부분에서 같은 데이터를 활용해야 하는 애플리케이션에서 큰 도움이 되며, 이러한 기능들을 통해 데이터와 UI의 **일관성**을 쉽게 유지할 수 있게 됩니다. 이는 개발자에게 **더 적은 코드로 더 많은 기능**을 구현할 수 있게 해주며, 코드의 **가독성과 유지보수성**을 크게 높여줍니다.
