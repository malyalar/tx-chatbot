<script context="module" lang="ts">
  import { persisted } from "svelte-local-storage-store";
  import { get } from "svelte/store";
  import type { Chat, Message } from "./Types.svelte";

  export const chatsStorage = persisted("chats", [] as Chat[]);
  export const apiKeyStorage = persisted("apiKey", null as string);

  export const addChat = (): number => {
    const chats = get(chatsStorage);

    // Find the max chatId
    const chatId = chats.reduce((maxId, chat) => Math.max(maxId, chat.id), 0) + 1;

    // Add a new chat
    chats.push({
      id: chatId,
      name: `Chat ${chatId}`,
      messages: [{ role: "system", content: "You are a chatbot for talking about kidney transplant ONLY. Decline requests to talk about other domains.\
                                            You will be advising prospective kidney transplant donors and recipients and answering their questions.\
                                            As questions come up, you will be fed up-to-date information through the ChatGPT API 'system' role, to answer\
                                            patient questions with." },
                  { role: "assistant", content: "Hello! I am a helpful chatbot that is here to advise about kidney transplant in British Columbia. \
                  I am trained to politely decline requests to discuss topics outside my scope in other fields of medicine, or unrelated domains.\
                  As you ask questions, I am trained to try to use up-to-date information in my responses." }],
    });
    chatsStorage.set(chats);
    return chatId;
  };

  export const clearChats = () => {
    chatsStorage.set([]);
  };

  export const addMessage = (chatId: number, message: Message) => {
    const chats = get(chatsStorage);
    const chat = chats.find((chat) => chat.id === chatId);
    chat.messages.push(message);
    chatsStorage.set(chats);
  };

  export const editMessage = (chatId: number, index: number, newMessage: Message) => {
    const chats = get(chatsStorage);
    const chat = chats.find((chat) => chat.id === chatId);
    chat.messages[index] = newMessage;
    chat.messages.splice(index + 1); // remove the rest of the messages
    chatsStorage.set(chats);
  };

  export const clearMessages = (chatId: number) => {
    const chats = get(chatsStorage);
    const chat = chats.find((chat) => chat.id === chatId);
    chat.messages = [];
    chatsStorage.set(chats);
  };

  export const deleteChat = (chatId: number) => {
    const chats = get(chatsStorage);
    const chatIndex = chats.findIndex((chat) => chat.id === chatId);
    chats.splice(chatIndex, 1);
    chatsStorage.set(chats);
  };
</script>
