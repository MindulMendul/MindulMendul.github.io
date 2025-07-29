---
layout: post
title: useState에서 Jotai + Context API + Tanstack-Query로 리팩토링하기
subtitle: 응집도는 높이고, 의존성은 낮추고!

categories: Front-End
tags: [Front End, Software Engineering, Tanstack-Query, Jotai, Context API]

toc: true
toc_sticky: true

date: 2025-07-29
---

## 일단 MVP 개발을 빠르게 하자!

AI를 활용해 프로젝트의 MVP를 빠르게 개발했습니다. 프로젝트에서 맡은 일은 AI 채팅창 구현을 하는 것입니다. 이게, 그냥 가벼운 채팅 프로젝트는 아니고, 채팅 내용, 채팅방 목록, 사이드바, 사이드 패널 등 생각보다 많은 기능이 포함된 애플리케이션이었습니다. 일단 개발을 빨리 끝내야한다는 사명감을 갖고, MVP의 모든 상태 관리는 useState로 처리하는 과감한(?) 선택을 했습니다. 예상하셨겠지만, 이 선택은 얼마 지나지 않아 거대한 기술 부채로 돌아왔습니다.

#### 문제점: useState의 한계와 Props Drilling

기능이 복잡해지고 컴포넌트 구조가 깊어지면서 문제는 명확해졌습니다.

1. 복잡한 상태 로직: 하나의 파일에 수많은 useState와 핸들러 함수가 얽혀 코드가 비대해졌습니다.
2. Props Drilling: 상위 컴포넌트의 상태를 하위의 하위 컴포넌트까지 전달하기 위해 의미 없는 props 전달이 반복되었습니다.
   아마 바이브코딩으로 나오는 MVP의 코드스멜은, 개발자들이 가장 잘 느낄 거라 생각합니다. 이제는 상태 관리 구조의 전면적인 리팩토링이 시급했습니다.

## 상태관리를 해보자!... 그런데 어떻게?

일단 해보자고는 생각했는데, useState말고 어떤 선택지가 있을지 고민해보았습니다. 사실 FrontEnd 개발자들은 이런 선택지 고민에서 자연스럽게 튀어나오는 스택들이 여러 개 있을 거라 생각합니다. React-Query, Context API, 전역 상태 라이브러리(Zustand, Jotai, valtio 등)... 사실 그냥 떠오르는 거 쓰면 되는 게 맞는데요. 그래도 나름 "근거 있는 사용"이 중요하니만큼, 한 번 고민해볼 가치는 있다고 봅니다. 일단 결론부터 말하자면, 네. 다 씁니다. 전부 다요. useState도 쓸 거고요. React-Query에 Context API에 Jotai까지 씁니다. 이제부터 그걸 왜 쓰는 건지 나름의 근거를 갖고 설명해볼게요.

## 1. 서버 상태 분리 (Tanstack-Query)

일단 가장 먼저 해결해야 하는 건 Polling 방식으로 구현된 채팅 메시지 관리였습니다. 백엔드에서는 생성되는 메시지를 Redis Queue에 저장해뒀다가, API를 호출할 때마다 Polling 해주는 식으로 구현했는데요. 프론트엔드에서의 요구사항은 3초마다 GET API를 요청해서 메시지를 받아오는 일이었습니다.

프론트엔드 개발자는 이를 setTimeout으로 할지, 다른 대안이 있는지 고민할 텐데요. 이 부분을 Tanstack-Query로 해결하면 깔끔하게 해결할 수 있으리라고 보았습니다. 나중에 web socket 방식으로 변경할 때에도, setTimeout 함수를 이용하는 것보다 구조 변경할 것도 줄어들 것이고, 제가 직접 커스텀 라이브러리르 짜는 것보다야 이미 잘 만들어진 Tanstack-Query를 쓰는 게 더 좋을 거라 생각했습니다.

(비동기 데이터를 다루는 데 거의 표준이 된 Tanstack-Query(React-Query)를 도입했습니다.Tanstack-Query를 사용함으로써 로딩, 에러 상태 처리는 물론, 캐싱과 데이터 동기화 로직을 컴포넌트에서 깔끔하게 분리할 수 있습니다... 라는 대본이 있는데, 블로그 작성할 때 AI가 한 번 검토를 때려줍니다. 그거 참고해서 쓰는 거라, 이건 그냥 살려서 여기다가 남깁니다;;)

![리액트 쿼리 폴더 구조](https://MindulMendul.github.io/assets/images/2025-07-29/image.png)

Tanstack-Query를 위해서 따로 src/queries 폴더에 useChatMessage 와 useChatRoom 이라는 커스텀 훅을 하나 만들었습니다. 원래 hooks로 분리를 했었지만, Tanstack-Query를 이용하는 훅이니까 일단 이름을 달리 해뒀습니다. 그런데, 여전히 어떻게 해야할지 잘 몰라서, 커스텀 훅 냄새가 더 난다싶으면 그냥 하나의 폴더로 합치는 것도 고려하고 있습니다.

```ts
/// src/queries/useChatMessage.ts

import { getChatReceive, postChatSend } from "@/api/chatAPI";
import { currentChatIdAtom, messagesAtomFamily } from "@/atoms/chatAtoms";
import { queryKeys } from "@/lib/queryKeys";
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { useAtom, useStore } from "jotai";
import { useEffect } from "react";

export const useChatMessage = () => {
  const [chatId, setChatId] = useAtom(currentChatIdAtom);
  const queryClient = useQueryClient();
  const store = useStore();
  const newChatMessageFetcherKey = queryKeys.chatMessages.fetcher(chatId!); // 채팅방 별로 API 받아오는 키

  const { data: queryResult, status } = useQuery({
    queryKey: newChatMessageFetcherKey, // 여기서는 API를 받아서 가공할 예정
    queryFn: () => getChatReceive(chatId!),
    refetchInterval: 3000,
    refetchIntervalInBackground: true,
    enabled: !!chatId,
  });

  // chat message accumulating
  useEffect(() => {
    if (!chatId || status !== "success" || !queryResult.ok) return;
    store.set(messagesAtomFamily(chatId), (oldMessages = []) => {
      const newMessage = queryResult.data;
      if (oldMessages.some((msg) => msg.id === newMessage.id))
        return oldMessages;
      return [...oldMessages, newMessage];
    });
  }, [queryResult, status, queryClient, chatId]);

  // 메시지 보내기 mutation
  const { mutate, isPending: isSending } = useMutation({
    mutationFn: (newMessage: Message) =>
      postChatSend(chatId, {
        content: newMessage.text,
        imageUrl: newMessage.images?.[0].src,
      }),
    onSuccess: (responseFromServer, newMessage) => {
      if (!responseFromServer.ok) return;
      const newChatId = responseFromServer.data;
      store.set(messagesAtomFamily(newChatId), (oldMessages) => [
        ...oldMessages,
        newMessage,
      ]);
      if (!chatId && newChatId) {
        setChatId(newChatId);
        queryClient.invalidateQueries({ queryKey: queryKeys.chatRooms.all() });
      }
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: newChatMessageFetcherKey });
    },
  });

  if (!chatId) {
    return {
      isLoading: false,
      error: false,
      sendMessage: mutate,
      isSending: isSending,
    };
  }

  return {
    isLoading: status === "pending",
    error: status === "error",
    sendMessage: mutate,
    isSending: isSending,
  };
};
```

리액트 쿼리가 나오는 가장 대표적인 커스텀 훅인, useChatMessage.ts 를 한 번 봅시다 ㅎㅎ 코드 자체의 양은 굉장히 많은데, 중요한 건 messages가 어느 채팅방에서 갱신되는지만 보면 됩니다.

채팅은 보내기와 받기, 두 가지 상황에 대해서 서버 상태가 관리되어야 할 겁니다.

받는 것에 대해서는 단순히 GET API를 tanstack-query를 이용해 관리하면 됩니다. 3초마다 API를 호출해서 받아진 데이터를, 적절히 수정해서 messages에 담아주면 됩니다. 아 물론, 실질적으로 모든 컴포넌트들은 상태를 jotai를 통해서 받을 예정이므로, 리액트 쿼리는 단순히 서버 데이터를 받아서, jotai에 데이터를 넣어주는 역할만 해주면 됩니다.

보내는 것에 대해서는 조금 더 복잡한데요. mutate를 활용하여, messages를 갱신해주는 식으로 구현합니다. 여기서 조금 어려웠던 건,

```ts
store.set(messagesAtomFamily(newChatId), (oldMessages) => [
  ...oldMessages,
  newMessage,
]);
```

이 부분이었는데요. 원래 코드는

```ts
const [messages, setMessages] = useAtom(messagesAtomFamily(chatId));
...

setMessages(( (oldMessages) => [...oldMessages, newMessage]));
```

이런 식으로 구현했었습니다. 근데 이게 조금 짜증났던 게, chatId 들어가는 부분에서 조금 복잡한 부분이 있더라고요.

1. 최초로 채팅방에 들어간 경우 chatId 는 null 값을 가짐.
2. messages 는 chatId=null 인 채팅방을 참조하고 있음.
3. 채팅을 치는 순간, 백엔드에서는 chatId 값을 확인함.
4. 만약 chatId값이 null 이라면, 새로운 chatId 값을 응답으로 보내줌.
5. 새로운 chatId 값을 전달받았으므로, 그에 대해서 messages가 참조하는 채팅방을 chatId=response_data로 갱신해주어야 함.
6. 그러나, **"const messages = ..."** 이런 식으로 위에서 선언하니, 정작 messages 자체는 chatId=null을 계속 참조하고 있음.
7. 결국 mutate에서는 갱신되지 않은 messages를 사용하니, chatId=null에 있는 채팅방에 사용자 채팅이 올라감.
8. 이후에 tanstack-query에서 캐싱을 통해 chatId=response_data로 온전히 교체해줌
9. 7번에서 mutate된 사용자 채팅은 무시되고, 그 이후의 채팅내역만 저장됨

이라는 괴랄한 시퀀스때문에 엄청 고통받았습니다 ㅠㅠ 그래서 그것을 해결해주기 위해서, useStore를 이용해서, 직접 데이터를 쏴주는(?) 형식으로 구현했다죠 ㅎㅎ

아무튼 이런 식으로 서버 상태만 떼어내어 Tanstack-Query가 온전히 관리할 수 있도록 구현해두었습니다. 리액트쿼리가 jotai에게 일방적으로 데이터를 보내주니, jotai는 상태에 대한 불신을 가질 이유도 없습니다.

## 2. 클라이언트 상태 재구성 (Jotai)

다음은 클라이언트 상태였습니다. Zustand와 Jotai를 두고 고민했지만, 원자(atom) 단위로 상태를 관리하는 Jotai의 방식이 더 세밀한 제어에 유리하다고 판단했습니다. 개별 상태 조각을 useAtom으로 간편하게 구독하고 업데이트하는 방식이 제가 추구하는 방향과 잘 맞았습니다. 또, 이미 Zustand는 한 번 써봤으니까요. 새로운 걸 한 번 써볼 때도 됐죠. 두 라이브러리의 성능, 최적화 차이가 크게 없으니, 이왕 김에 새로운 기술 스택을 도전해보겠다는 심산이죠.

채팅과 관련된 대부분의 UI 상태, 사용자 인터랙션 상태를 useState에서부터 Jotai로 전환하기 시작했습니다.

![Jotai Atom 폴더 구조](https://MindulMendul.github.io/assets/images/2025-07-29/image1.png)

Jotai는 전역상태 라이브러리인 만큼, src/atoms 폴더를 하나 만들어줬습니다. 지금은 /chat 페이지에서만 활용하지만, 나중에 /graph 라든가, /archive 라든가, 페이지를 다양하게 만들 상황이 생길 것 같았습니다. 그래서, 사진에는 authAtoms가 추가로 있는 것을 볼 수 있는데, 이것의 일환이라고 생각해주시면 될 듯싶습니다.

```ts
/// src/atoms/chatAtoms.ts

import { atom } from 'jotai';
import { atomFamily } from 'jotai/utils';

export const inputValueAtom = atom<string>('');
export const inputImageAtom = atom<MessageImage | undefined>(undefined);
export const currentChatIdAtom = atom<number | null>(null);
export const isAIRespondingAtom = atom<boolean>(false);
export const hasUserSentMessageAtom = atom<boolean>(false);
export const isSidebarOpenAtom = atom<boolean>(false);

...

export const messagesAtomFamily = atomFamily((chatId: number | null) => atom<Message[]>([]));
export const examplesAtomFamily = atomFamily((chatId: number | null) =>
  atom<string[]>([]));
```

필요한 상태를 전부 atom으로 저장해서 관리합니다.(아, 물론, 여기서는 모든 상태를 보여드리진 않습니다. ㅎㅎ) store에다가 저장해서 관리하던 zustand랑 다르게, jotai는 개별 상태를 하나하나 atom에 넣어서 관리한다는 점이 매력입니다. 뭐, 여전히 어떤 라이브러리를 고르든 간에, 성능이나 개발 경험에 큰 차이는 없습니다.

아무튼 이렇게 관리할 상태를 세팅해두고 나면, 그에 따라 하나하나 꺼내쓰면 됩니다. 꺼내쓰는 예시가 이미 1. Tanstack-Query 에서 스포일러된 느낌이 있는데, 그런 느낌이구나 까지만 알면 될 듯싶습니다.

조금 재미있는 건, messagesAtomFamily 쪽에 보면, atomFamily 를 사용하고 있는 걸 알 수 있는데요. 위에서 채팅방 뭐시기 얘기하면서 나왔던 녀석입니다. 저렇게 chatId를 기준으로, 마치 hashmap처럼 묶어서 관리할 수 있는 자료구조인 듯싶습니다. messsages의 역할은 "chatId값을 가진 채팅방의 채팅내역"이므로, atomFamily를 활용하기에 가장 적합하다고 볼 수 있겠습니다 ㅎㅎ

## 3. 핸들러 함수 분리 (Context API)

Jotai로 상태를 옮기고 나니, 이 상태들을 조작하는 핸들러 함수들이 또다시 한곳에 뭉쳐 불편함을 야기했습니다. 이 함수들은 상태와 강하게 결합되어 있지만, 모든 컴포넌트에 필요한 것은 아니었죠.

여기서 Context API를 활용했습니다. 특정 페이지나 기능 범위에서만 필요한 핸들러 함수들을 모아 Provider 컴포넌트를 만들었습니다. 이렇게 하니, 필요한 컴포넌트만 Provider로 감싸서 핸들러를 주입해줄 수 있었고, 불필요한 전역 노출을 막을 수 있었습니다.

## 4. 리펙토링 전과 후의 코드 품질 비교

page의 모양이 완전히 바뀝니다..!

```ts
/// 리펙토링 이전 app/chat/page.tsx

"use client";

import { useCallback, useEffect, useState } from "react";
import { AppSidebar } from "@/components/AppSidebar";
import { SidebarInset } from "@/components/ui/sidebar";
import ChatHeader from "@/components/ChatHeader";
import ChatSubmit from "@/components/ChatSubmit";
import ChatArea from "@/components/ChatArea";
import { useChat } from "@/hooks/useChat";
import { useChatRooms } from "@/hooks/useChatRoom";
import ImagePanel from "@/components/ImagePanel";

export default function Chat() {
  const [inputValue, setInputValue] = useState<string>("");
  const [inputImage, setInputImage] = useState<MessageImage | undefined>(
    undefined
  );
  const [currentChatId, setCurrentChatId] = useState<number | null>(null);
  const [isAIResponding, setIsAIResponding] = useState<boolean>(false);
  const [hasUserSentMessage, setHasUserSentMessage] = useState<boolean>(false);
  const [openTabs, setOpenTabs] = useState<MessageImage[]>([]);
  const [activeTabId, setActiveTabId] = useState<string | null>(null);
  const [isPanelOpen, setIsPanelOpen] = useState(false);
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);

  /** chat state 관리하는 hook */
  const {
    messages,
    examples,
    isLoading: isChatLoading,
    sendMessage,
    addMessageToCache,
  } = useChat(currentChatId);
  const { rooms: chatRooms, error: chatRoomError } = useChatRooms();

  // AI 응답이 오면 스피너를 숨기는 효과
  useEffect(() => {
    if (messages.length > 0 && hasUserSentMessage) {
      const lastMessage = messages[messages.length - 1];
      if (lastMessage && lastMessage.user.userId !== "asdf")
        setIsAIResponding(false);
    }
  }, [messages, hasUserSentMessage]);

  useEffect(() => {
    if (isSidebarOpen) {
      setIsPanelOpen(false);
    }
  }, [isSidebarOpen]);

  const sendMsg = useCallback(
    (inputValue: string, inputImage?: MessageImage) => {
      const userMessage: Message = {
        id: Date.now().toString(),
        text: inputValue,
        user: { userId: "asdf", username: "mindul" },
        images: inputImage && [inputImage],
        timestamp: new Date(),
      };

      sendMessage(
        { roomId: currentChatId, newMessage: userMessage },
        {
          onSuccess: (responseFromServer) => {
            if (responseFromServer.ok) {
              addMessageToCache(userMessage, responseFromServer.data);
              if (currentChatId == null)
                setCurrentChatId(responseFromServer.data);
            }
          },
          onError: () => {
            setIsAIResponding(false); // 에러 발생 시에도 스피너 숨김
          },
        }
      );
    },
    [inputValue]
  );

  const handleExampleSelect = (exampleText: string) => {
    setHasUserSentMessage(true);
    setIsAIResponding(true);
    sendMsg(exampleText);
    setInputImage(undefined);
  };

  const handleSendMessage = () => {
    if (!inputValue.trim()) return;
    setHasUserSentMessage(true);
    setIsAIResponding(true);
    sendMsg(inputValue, inputImage);
    setInputValue("");
    setInputImage(undefined);
  };

  const handleChatSelect = (chatId: number) => {
    setIsAIResponding(false);
    setHasUserSentMessage(false);
    setCurrentChatId(chatId);
    setInputImage(undefined);
  };

  const handleNewChat = () => {
    if (chatRoomError) return;
    setCurrentChatId(null);
    setIsAIResponding(false);
    setHasUserSentMessage(false);
    setInputImage(undefined);
  };

  const handleOpenTab = (newImageData: MessageImage) => {
    const isAlreadyOpen = openTabs.some((tab) => tab.src === newImageData.src);
    if (!isAlreadyOpen) {
      setOpenTabs((prevTabs) => [...prevTabs, newImageData]);
    }
    setActiveTabId(newImageData.src);
    setIsPanelOpen(true); // 이미지 클릭 시 패널 열기
  };

  // 3. 탭을 닫는 함수
  const handleCloseTab = (tabSrcToClose: string) => {
    const remainingTabs = openTabs.filter((tab) => tab.src !== tabSrcToClose);
    setOpenTabs(remainingTabs);
    if (activeTabId === tabSrcToClose) {
      if (remainingTabs.length > 0) {
        setActiveTabId(remainingTabs[remainingTabs.length - 1].src);
      } else {
        setActiveTabId(null);
      }
    }
  };

  const handleTogglePanel = () => {
    setIsPanelOpen(!isPanelOpen);
  };

  return (
    <div className="flex min-h-screen w-full relative">
      <div className="hidden lg:block">
        <AppSidebar
          currentChatId={currentChatId}
          onChatSelect={handleChatSelect}
          onNewChat={handleNewChat}
          chatRooms={chatRooms}
        />
      </div>
      <SidebarInset className="flex flex-col h-[100dvh] lg:transition-all lg:duration-300">
        <div className="flex flex-col h-full">
          <ChatHeader
            currentChatId={currentChatId}
            onChatSelect={handleChatSelect}
            onNewChat={handleNewChat}
            chatRooms={chatRooms}
            isSidebarOpen={isSidebarOpen}
            setIsSidebarOpen={setIsSidebarOpen}
            setIsPanelOpen={setIsPanelOpen}
          />
          <div className="flex-1 min-h-0">
            <ChatArea
              userID={"asdf"}
              messages={messages}
              isLoading={isChatLoading && messages.length === 0}
              isAIResponding={isAIResponding && hasUserSentMessage}
              examples={examples}
              onExampleSelect={handleExampleSelect}
              setSelectedImage={handleOpenTab}
            />
          </div>
          <ChatSubmit
            inputValue={inputValue}
            setInputValue={setInputValue}
            inputImage={inputImage}
            setInputImage={setInputImage}
            handleSendMessage={handleSendMessage}
          />
        </div>
      </SidebarInset>
      <ImagePanel
        isOpen={isPanelOpen}
        onToggle={handleTogglePanel}
        openTabs={openTabs}
        activeTabId={activeTabId}
        onTabSelect={setActiveTabId}
        onTabClose={handleCloseTab}
      />
    </div>
  );
}
```

```ts
/// 리펙토링 이후 app/chat/page.tsx

import { ChatSidebar } from "@/components/chat/ChatSidebar";
import { SidebarInset } from "@/components/ui/sidebar";
import ChatHeader from "@/components/chat/ChatHeader";
import ChatInputBox from "@/components/chat/ChatInputBox";
import ChatArea from "@/components/chat/ChatArea";
import { ChatContextProvider } from "@/components/chat/ChatContextProvider";
import ChatSidePanel from "@/components/chat/ChatSidePanel";

export default function Chat() {
  return (
    <ChatContextProvider>
      <div className="flex min-h-screen w-full relative">
        <div className="hidden lg:block">
          <ChatSidebar />
        </div>
        <SidebarInset className="flex flex-col h-[100dvh] lg:transition-all lg:duration-300">
          <div className="flex flex-col h-full">
            <ChatHeader />
            <div className="flex-1 min-h-0">
              <ChatArea />
            </div>
            <ChatInputBox />
          </div>
        </SidebarInset>
        <ChatSidePanel />
      </div>
    </ChatContextProvider>
  );
}
```

리팩토링 결과, 그냥 컴포넌트만 불러오는 수준으로 깔끔하게 정리가 되었습니다. 상태를 보려면 각 기능에 맞는 파일을 보러 가면 되는 거고, page에서는 더이상 상태를 신경쓸 필요 없이, 그냥 구조만 신경쓰면 되는 상황으로 바뀌었습니다.

이건 컴포넌트를 봐도 마찬가지인데요.

```ts
/// 리펙토링 이전 src/components/chatArea.tsx

import { messageColor } from '@/styles/chat';
import { useEffect, useRef } from 'react';

export default function ChatArea({
  userID,
  messages,
  isLoading,
  isAIResponding,
  examples,
  onExampleSelect,
  setSelectedImage,
}: {
  userID: string;
  messages: Message[];
  isLoading: boolean;
  isAIResponding?: boolean;
  examples: string[];
  onExampleSelect?: (text: string) => void;
  setSelectedImage: Function;
}) {
   ...
}
```

```ts
/// 리펙토링 이후 src/components/chatArea.tsx

'use client';

import { userAtom } from '@/atoms/authAtoms';
import { currentChatIdAtom, examplesAtomFamily, isAIRespondingAtom, messagesAtomFamily } from '@/atoms/chatAtoms';
import { useChatHandlers } from '@/components/chat/ChatContextProvider';
import { useChatMessage } from '@/queries/useChatMessage';
import { useAtomValue } from 'jotai';
import { useCallback, useEffect, useRef } from 'react';

export default function ChatArea() {
  const { isLoading: isChatLoading } = useChatMessage();

  const isAIResponding = useAtomValue(isAIRespondingAtom);
  const currentChatId = useAtomValue(currentChatIdAtom);
  const messages = useAtomValue(messagesAtomFamily(currentChatId));
  const messageExamples = useAtomValue(examplesAtomFamily(currentChatId));

  const { handleOpenTab } = useChatHandlers();

  ...
}
```

pages에서는 극명한 비교를 위해서 원본을 가져왔으나, 컴포넌트 단위에서는 그 정도까지는 필요없어보여서, 상태관리 부분만 가져왔습니다. 컴포넌트에서도 확실히 그 모양이 개선되었음을 알 수 있습니다. props drilling이 필수적으로 보이는 리펙토링 전 컴포넌트 모양과는 다르게, 리펙토링 이후 컴포넌트 모양은, 그냥 import 딸깍으로 바뀐 것을 알 수 있습니다.

이 차이는 컴포넌트가 많아지면 극명하게 갈릴 텐데요. 일례로, 리펙토링 전 chatArea는 children이 요구하던 examples 상태를 보내주기 위해서 props로 받고 있는 모습을 보여줍니다. 하지만 리펙토링 후 chatArea는 examples 상태를 굳이 받고 있지 않죠. 이렇게 리펙토링을 거친 결과, 컴포넌트가 상태를 참조하고 변경하기에 상당히 간단해집니다. jotai로 구현한 기능은 useState로 구현해둔 기능과 완전히 동일한 기능이지만, 모양은 완전히 다르다는 걸 느낄 수 있습니다.

## 5. 최종 정리: 상태 관리 역할의 명확한 분리

리팩토링을 통해 각 상태 관리 도구는 명확한 역할을 갖게 되었습니다.

- Tanstack-Query (서버 통신 담당): 채팅방 목록, 대화 내역 등 서버 데이터를 가져와 캐싱하고, 주기적으로 동기화하여 최신 상태를 유지합니다. 가져온 데이터는 Jotai 아톰을 업데이트하는 데 사용됩니다.

- Jotai (상태의 중심 - Source of Truth): 애플리케이션의 모든 상태(서버에서 온 데이터, UI 상태 등)를 아톰으로 관리합니다. 모든 컴포넌트는 Jotai를 통해 상태를 읽습니다.

- Context API (핸들러 제공자): Jotai 아톰을 변경하는 핸들러 함수들을 묶어 필요한 컴포넌트 트리에만 제공합니다.

이 구조를 통해 useState와 핸들러 함수를 전역적으로 사용하던 초기의 방식보다 응집도는 높이고 의존성은 낮추는 효과를 얻을 수 있었습니다. 각 도구가 자신의 역할에만 충실하게 되면서, 코드의 추적과 유지보수가 훨씬 용이해졌습니다. 이제는 새로운 기능을 추가할 때도 어떤 도구를 사용해 어디에 코드를 작성해야 할지 명확하게 판단할 수 있게 되었습니다.
