# Reduce-the-Number-of-API-Calls-by-Using-WebSockets-and-Frontend-Storage
How I Reduce the Number of API Calls by Using WebSockets and Frontend Storage

![Alt text](https://miro.medium.com/v2/resize:fit:4800/format:webp/0*s-m16PA0BFJIjhQg.jpeg)

# Optimizing API Calls with WebSockets

It doesn’t matter if you are a frontend or a backend developer; we all use APIs in our applications.  
APIs (Application Programming Interfaces) are the core mechanisms that handle communication between frontend, backend, and third-party services.  

Because of this, it’s essential for any developer to understand not only how APIs work, but also how to **optimize API calls** to make them faster, more efficient, and more scalable.

---

## The Problem I Faced

While developing my full-stack side project, I needed to retrieve the user from the backend.  

- **Backend**: NestJS with all APIs ready  
- **Frontend**: React with UI set up  

I wrote a custom function that fetched the user from the server and displayed it on the screen. Everything worked fine — until I integrated the rest of the APIs.  
Almost all APIs altered the main user’s data.

To ensure the UI always displayed the latest user data, I was fetching the user **on every page load**.  
This was inefficient and needed optimization.

---

## The Solution

To reduce redundant API calls, caching alone wasn’t enough since user data frequently changed.  
After some research and discussions with senior developers, I implemented the following:

✅ **WebSocket connection** between client and server  
✅ **Emit update-user events** on the server when user data changes  
✅ **Client listens for update events** and only then fetches new user data  

---

## Backend Implementation (NestJS)

We’ll use NestJS Gateways to set up WebSockets.

### Install Dependencies
```bash
npm i --save @nestjs/websockets @nestjs/platform-socket.io
````

### WebSocket Gateway

```ts
// user.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
  OnGatewayConnection,
  OnGatewayDisconnect,
} from '@nestjs/websockets';
import { Server } from 'socket.io';

@WebSocketGateway({
  cors: {
    origin: 'http://localhost:3001',
    methods: ['GET', 'POST'],
  },
})
export class UserGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  private clients = new Set<string>();

  handleConnection(client: any) {
    console.log(`Client connected: ${client.id}`);
    this.clients.add(client.id);
  }

  handleDisconnect(client: any) {
    console.log(`Client disconnected: ${client.id}`);
    this.clients.delete(client.id);
  }

  updateUser(user: any) {
    this.server.emit('update-user', user);
  }

  @SubscribeMessage('requestUserUpdate')
  handleUserUpdateRequest(@MessageBody() data: any) {
    console.log("WS: ", data)
    this.updateUser(data);
  }
}
```

### Gateway Module

```ts
// gateway.module.ts
import { Module } from "@nestjs/common";
import { UserGateway } from "./user.gateway";

@Module({
  providers: [UserGateway]
})
export class GatewayModule {}
```

### Import into App Module

```ts
// app.module.ts
import { Module } from '@nestjs/common';
import { GatewayModule } from './user/presentation/gateways/gateway.module';
import { UserModule } from './user/infrastructure/modules/user.module';

@Module({
  imports: [
    GatewayModule,
    UserModule,
    // other modules...
  ],
})
export class AppModule {}
```

### Triggering Update Events

```ts
// update-user.command.handler.ts (simplified)
this.userGateway.updateUser(updatedUser);
```

---

## Frontend Implementation (React)

### Install Dependencies

```bash
npm i socket.io-client
```

### WebSocket Hook

```ts
// useSocket.ts
import { useEffect } from 'react';
import { io, Socket } from 'socket.io-client';
import { useDispatch } from 'react-redux';
import { setUser } from '@/lib/features/user/userSlice';
import axiosInstance from '../functions/axios-instance';
import { API_URLS, BASE } from '../constants/api-endpoints';
import { storeUserInLocalStorage } from '../functions/manage-user-local-storage';

let socket: Socket;

const useSocket = () => {
  const dispatch = useDispatch();

  useEffect(() => {
    socket = io(BASE);

    socket.on('update-user', () => {
      console.log('User data updated');
      getUser();
    });

    return () => {
      socket.disconnect();
    };
  }, [dispatch]);

  const getUser = async () => {
    try {
      const response = await axiosInstance.get(`${API_URLS.AUTH}`);
      dispatch(setUser(response.data.data));
      storeUserInLocalStorage(response.data.user);
    } catch (error) {
      console.error('Failed to fetch user data', error);
    }
  };

  return {};
};

export default useSocket;
```

### Using the Hook in Layout

```tsx
// Layout.tsx
import useSocket from '../utilities/hooks/useSocket';

const Layout = ({ children }: { children: React.ReactNode }) => {
  useSocket();

  return (
    <main>
      {children}
    </main>
  )
}

export default Layout;
```

---

## Local Storage Helpers

```ts
// manage-user-local-storage.ts
export const storeUserInLocalStorage = (user: any) => {
  localStorage.setItem('user', JSON.stringify(user));
};

export const getUserFromLocalStorage = () => {
  const user = localStorage.getItem('user');
  return user ? JSON.parse(user) : null;
};
```

---

## useGetUser Hook

```ts
// useGetUser.ts
import { useState, useEffect } from "react";
import { getUserQuery } from "@/app/services/queries/auth.query";
import { getUserFromLocalStorage, storeUserInLocalStorage } from "../functions/manage-user-local-storage";

const useGetUser = () => {
  const [user, setUser] = useState<any>(null);
  const { data: apiData, isLoading: apiLoading, isError, refetch } = getUserQuery();

  useEffect(() => {
    const localStorageData = getUserFromLocalStorage();
    if (localStorageData) {
      setUser(localStorageData);
    } else {
      refetch();
    }
  }, []);

  useEffect(() => {
    if (apiData) {
      setUser(apiData.data);
      storeUserInLocalStorage(apiData.data);
    }
  }, [apiData]);

  return { user, isLoading: apiLoading };
};

export default useGetUser;
```

Usage:

```ts
const { user, isLoading } = useGetUser();
```

---

## Additional Notes

* There are many ways to optimize API calls.
* I also use **TanStack Query (React Query)** for caching and error handling.
* Instead of LocalStorage, you could use Redux Persist or SecureStorage for better security.

---

## Conclusion

By switching from **constant API polling** to a **WebSocket-based approach**, the app became:

* More responsive
* Cleaner in logic
* More efficient in network usage

If you’re dealing with frequently updated data, this approach is worth considering.

---

## References

* [MDN Web Docs — WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)
* [React Query — Official Documentation](https://tanstack.com/query)
* [WebSockets vs HTTP Polling — A Practical Comparison](https://ably.com/concepts/websockets-vs-polling)
* [How to Use WebSockets in React](https://socket.io/how-to/use-with-react)
* [Best Practices for WebSockets — Mozilla](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_client_applications)
* [RFC 6455 — The WebSocket Protocol (IETF)](https://www.rfc-editor.org/rfc/rfc6455)

---

**Article written by David Aslanyan**
