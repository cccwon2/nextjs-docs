이 설정에서는 **`httpOnly` 쿠키를 사용해 `accessToken`과 `refreshToken`을 저장**하고, 사용자 정보를 **`zustand`로 관리**하며, **`middleware`를 통해 보호된 라우트를 제어**합니다. 이를 기반으로 설정을 설계하고 구현 예제를 작성해 보겠습니다.

---

## 📂 폴더 구조

```plaintext
src/
├── app/
│   ├── api/
│   │   ├── auth/
│   │   │   ├── login/route.ts         # 외부 API로 로그인 요청
│   │   │   └── logout/route.ts        # 로그아웃 처리
│   │   ├── protected/route.ts         # 보호된 API 라우트 예제
│   ├── dashboard/
│   │   └── page.tsx                   # 보호된 페이지
│   ├── login/
│   │   └── page.tsx                   # 로그인 페이지
├── middleware.ts                      # Middleware로 토큰 검증
├── hooks/
│   ├── useAuth.ts                     # 인증 상태 관리 훅
├── store/
│   ├── userStore.ts                   # Zustand로 사용자 상태 관리
├── lib/
│   ├── cookies.ts                     # 쿠키 유틸리티
│   ├── jwt.ts                         # JWT 관련 유틸리티
```

---

## 1️⃣ **`httpOnly` 쿠키를 통한 토큰 저장**

### **쿠키 유틸리티 (`lib/cookies.ts`)**

`httpOnly` 쿠키는 클라이언트 자바스크립트에서 직접 접근할 수 없으므로 서버에서 처리해야 합니다.

```typescript
import { cookies } from "next/headers";
import { NextRequest, NextResponse } from "next/server";

// 쿠키 설정 함수
export const setCookie = (name: string, value: string, options?: Partial<CookieOptions>) => {
  const cookieStore = cookies();
  cookieStore.set(name, value, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "strict",
    path: "/",
    ...options,
  });
};

// 쿠키 삭제 함수
export const deleteCookie = (name: string) => {
  const cookieStore = cookies();
  cookieStore.set(name, "", { maxAge: -1 });
};

// 쿠키 가져오기 함수
export const getCookie = (name: string): string | null => {
  const cookieStore = cookies();
  return cookieStore.get(name)?.value || null;
};
```

---

## 2️⃣ **외부 API로 로그인 요청**

### **`app/api/auth/login/route.ts`**

로그인 시 외부 API로 요청을 보내고, 받은 `accessToken`과 `refreshToken`을 `httpOnly` 쿠키에 저장합니다.

```typescript
import { NextResponse } from "next/server";
import { z } from "zod";
import { setCookie } from "@/lib/cookies";

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

export async function POST(request: Request) {
  try {
    const body = await request.json();
    const { email, password } = loginSchema.parse(body);

    // 외부 API로 로그인 요청
    const response = await fetch(`${process.env.EXTERNAL_API_BASE_URL}/auth/login`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email, password }),
    });

    if (!response.ok) {
      return NextResponse.json({ error: "Invalid credentials" }, { status: 401 });
    }

    const { accessToken, refreshToken, user } = await response.json();

    // 쿠키에 토큰 저장
    setCookie("accessToken", accessToken, { maxAge: 3600 });
    setCookie("refreshToken", refreshToken, { maxAge: 7 * 24 * 60 * 60 }); // 7일

    // 사용자 정보 반환
    return NextResponse.json({ user });
  } catch (error) {
    return NextResponse.json({ error: error.message }, { status: 400 });
  }
}
```

---

### **로그아웃 API (`app/api/auth/logout/route.ts`)**

```typescript
import { NextResponse } from "next/server";
import { deleteCookie } from "@/lib/cookies";

export async function POST() {
  deleteCookie("accessToken");
  deleteCookie("refreshToken");

  return NextResponse.json({ message: "Logged out successfully" });
}
```

---

## 3️⃣ **`zustand`로 사용자 상태 관리**

### **Zustand 상태 관리 (`store/userStore.ts`)**

사용자 정보를 상태로 관리하며, 로그인 후 `user` 정보를 저장합니다.

```typescript
import { create } from "zustand";

type User = {
  id: string;
  name: string;
  email: string;
};

type UserState = {
  user: User | null;
  setUser: (user: User) => void;
  clearUser: () => void;
};

export const useUserStore = create<UserState>((set) => ({
  user: null,
  setUser: (user) => set({ user }),
  clearUser: () => set({ user: null }),
}));
```

---

## 4️⃣ **Middleware로 인증 확인**

### **`middleware.ts`**

Middleware에서 `accessToken`을 검증하여 보호된 라우트를 관리합니다.

```typescript
import { NextRequest, NextResponse } from "next/server";
import { verifyJWT } from "@/lib/jwt";

export function middleware(req: NextRequest) {
  const token = req.cookies.get("accessToken");

  if (!token) {
    return NextResponse.redirect(new URL("/login", req.url));
  }

  try {
    verifyJWT(token);
    return NextResponse.next();
  } catch {
    return NextResponse.redirect(new URL("/login", req.url));
  }
}

export const config = {
  matcher: ["/dashboard/:path*"], // 보호된 경로
};
```

---

## 5️⃣ **React 클라이언트에서 사용자 상태 관리**

### **로그인 페이지 (`app/login/page.tsx`)**

로그인 요청 후 사용자 상태를 업데이트합니다.

```tsx
"use client";

import { useState } from "react";
import { useUserStore } from "@/store/userStore";

export default function LoginPage() {
  const setUser = useUserStore((state) => state.setUser);
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");

  const handleLogin = async () => {
    const response = await fetch("/api/auth/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email, password }),
    });

    if (response.ok) {
      const { user } = await response.json();
      setUser(user); // Zustand로 사용자 상태 업데이트
      window.location.href = "/dashboard";
    } else {
      alert("Login failed");
    }
  };

  return (
    <div>
      <h1>Login</h1>
      <input type="email" placeholder="Email" value={email} onChange={(e) => setEmail(e.target.value)} />
      <input type="password" placeholder="Password" value={password} onChange={(e) => setPassword(e.target.value)} />
      <button onClick={handleLogin}>Login</button>
    </div>
  );
}
```

---

### **대시보드 페이지 (`app/dashboard/page.tsx`)**

`zustand`를 활용해 사용자 정보를 표시합니다.

```tsx
"use client";

import { useUserStore } from "@/store/userStore";

export default function DashboardPage() {
  const user = useUserStore((state) => state.user);

  if (!user) {
    return <p>Loading...</p>;
  }

  return (
    <div>
      <h1>Welcome, {user.name}</h1>
      <p>Email: {user.email}</p>
    </div>
  );
}
```

---

## 6️⃣ **JWT 유틸리티 (`lib/jwt.ts`)**

외부 API로부터 받은 JWT를 검증합니다.

```typescript
import jwt from "jsonwebtoken";

const JWT_SECRET = process.env.JWT_SECRET || "your-secret-key";

export const verifyJWT = (token: string) => {
  try {
    return jwt.verify(token, JWT_SECRET);
  } catch (error) {
    throw new Error("Invalid or expired token");
  }
};
```

---

위 구조는 다음을 충족합니다:

1. **`httpOnly` 쿠키를 사용하여 보안 강화.**
2. **Zustand로 클라이언트에서 사용자 상태를 효율적으로 관리.**
3. **React에서 사용자 친화적인 인증 및 상태 관리 구현.**
4. **Middleware로 보호된 경로 접근 제어.**

필요에 따라 리프레시 토큰 로직이나 더 세부적인 에러 처리를 추가하면 됩니다.

`zod` 스키마를 폴더와 파일로 분리하여 관리하는 것은 매우 좋은 접근입니다. 이를 통해 스키마를 모듈화하고, 재사용성을 높이며, 파일 간 의존성을 명확히 할 수 있습니다. 제안하신 구조에 따라 `zod` 스키마를 설계하고 구현하겠습니다.

---

## 📂 폴더 구조 제안

```plaintext
src/
├── zod/
│   ├── commonSchema.ts                 # 공통 스키마
│   ├── authSchema.ts                   # 인증 관련 스키마
│   └── dashboardSchema.ts              # 대시보드 관련 스키마
```

---

## 🛠️ 구현 예제

### 1️⃣ **`commonSchema.ts`**

공통으로 사용되는 스키마를 정의합니다. 예를 들어, 이메일, 비밀번호, ID 등 재사용 가능한 유효성 검사 규칙을 작성합니다.

```typescript
import { z } from "zod";

// 이메일 스키마
export const emailSchema = z.string().email({ message: "Invalid email format" });

// 비밀번호 스키마
export const passwordSchema = z
  .string()
  .min(8, { message: "Password must be at least 8 characters" })
  .max(64, { message: "Password cannot exceed 64 characters" });

// ID 스키마
export const idSchema = z.string().uuid({ message: "Invalid ID format" });
```

---

### 2️⃣ **`authSchema.ts`**

로그인, 회원가입 등 인증과 관련된 스키마를 정의합니다. 공통 스키마를 가져와 활용합니다.

```typescript
import { z } from "zod";
import { emailSchema, passwordSchema } from "./commonSchema";

// 로그인 스키마
export const loginSchema = z.object({
  email: emailSchema,
  password: passwordSchema,
});

// 회원가입 스키마
export const registerSchema = z
  .object({
    email: emailSchema,
    password: passwordSchema,
    confirmPassword: z.string().min(8, { message: "Password confirmation must match the password" }),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: "Passwords must match",
    path: ["confirmPassword"],
  });
```

---

### 3️⃣ **`dashboardSchema.ts`**

대시보드와 관련된 API 요청이나 입력값 유효성 검사를 정의합니다.

```typescript
import { z } from "zod";
import { idSchema } from "./commonSchema";

// 대시보드 항목 생성 스키마
export const createDashboardItemSchema = z.object({
  title: z.string().min(1, { message: "Title is required" }),
  description: z.string().optional(),
});

// 대시보드 항목 ID 기반 요청 스키마
export const dashboardItemIdSchema = z.object({
  id: idSchema,
});
```

---

## 사용 예시

### **API에서 사용**

API 라우트에서 요청 데이터를 검증할 때 `zod` 스키마를 사용합니다.

#### **로그인 API (`app/api/auth/login/route.ts`)**

```typescript
import { NextResponse } from "next/server";
import { loginSchema } from "@/zod/authSchema";

export async function POST(request: Request) {
  try {
    const body = await request.json();
    const validatedData = loginSchema.parse(body); // 스키마로 요청 데이터 검증

    // 외부 API 요청
    const response = await fetch(`${process.env.EXTERNAL_API_BASE_URL}/auth/login`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(validatedData),
    });

    if (!response.ok) {
      return NextResponse.json({ error: "Invalid credentials" }, { status: 401 });
    }

    const { accessToken, refreshToken, user } = await response.json();

    return NextResponse.json({ user });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json({ errors: error.errors }, { status: 400 });
    }
    return NextResponse.json({ error: error.message }, { status: 500 });
  }
}
```

---

### **프론트엔드에서 사용**

React 컴포넌트나 상태 관리 로직에서도 `zod` 스키마를 사용할 수 있습니다.

#### **로그인 폼 예제 (`app/login/page.tsx`)**

```tsx
"use client";

import { useState } from "react";
import { loginSchema } from "@/zod/authSchema";

export default function LoginPage() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);

  const handleLogin = async () => {
    try {
      // 입력 데이터 검증
      const validatedData = loginSchema.parse({ email, password });

      const response = await fetch("/api/auth/login", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(validatedData),
      });

      if (!response.ok) {
        throw new Error("Login failed");
      }

      const { user } = await response.json();
      console.log("Logged in user:", user);
    } catch (error) {
      if (error instanceof z.ZodError) {
        setError(error.errors.map((e) => e.message).join(", "));
      } else {
        setError(error.message);
      }
    }
  };

  return (
    <div>
      <h1>Login</h1>
      <input type="email" placeholder="Email" value={email} onChange={(e) => setEmail(e.target.value)} />
      <input type="password" placeholder="Password" value={password} onChange={(e) => setPassword(e.target.value)} />
      <button onClick={handleLogin}>Login</button>
      {error && <p style={{ color: "red" }}>{error}</p>}
    </div>
  );
}
```

---

## 📌 장점

1. **재사용성**: 공통 스키마를 다양한 기능에서 재활용할 수 있습니다.
2. **모듈화**: 각 스키마가 독립적인 파일로 관리되어 유지보수가 용이합니다.
3. **명확한 분리**: 스키마를 기능별로 나누어 코드 가독성과 개발 속도 향상.
4. **에러 처리 일관성**: API와 프론트엔드 모두 같은 스키마를 사용하여 일관된 유효성 검사가 가능.

이 구조는 스키마의 가독성과 관리 효율성을 높이며, 프로젝트의 확장성을 지원합니다. 👍
