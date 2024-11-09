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
