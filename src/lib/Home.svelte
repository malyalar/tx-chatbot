<script lang="ts">
  import { addChat, apiKeyStorage } from "./Storage.svelte";

  $: apiKey = $apiKeyStorage;

  export let activeChatId: number;
</script>

<article class="message">
  <div class="message-body">

    <strong><a href="https://github.com/malyalar/tx-chatbot">Transplant Chatbot</a></strong>
    is a simple one-page web patient education tool that implements unique strategies on top of OpenAI's ChatGPT interface to customize the responses to our domain of interest (kidney transplant) and practice setting (British Columbia). To use it, you need to register for
    <a href="https://platform.openai.com/account/api-key" target="_blank" rel="noreferrer">an OpenAI API key</a>
    first.
    <br>
    <br>
    All messages are stored in your browser's local storage (including the API key itself), but the data submitted to OpenAI can be stored and used in the future for other activities, such as training on the next iterations of GPT models. 
    As a result, we would recommend <strong>NOT</strong> discussing confidential patient matters in this iteration of the chatbot. You can also close the browser tab and come back later to continue the conversation.

  </div>
</article>
<article class="message" class:is-danger={!apiKey} class:is-warning={apiKey}>
  <div class="message-body">
    Set your OpenAI API key below:

    <form
      class="field has-addons has-addons-right"
      on:submit|preventDefault={(event) => {
        apiKeyStorage.set(event.target[0].value);
      }}
    >
      <p class="control is-expanded">
        <input
          aria-label="OpenAI API key"
          type="password"
          autocomplete="off"
          class="input"
          class:is-danger={!apiKey}
          value={apiKey}
        />
      </p>
      <p class="control">
        <button class="button is-info" type="submit">Save</button>
      </p>
    </form>

    {#if !apiKey}
      <p class="help is-danger">
        Please enter your <a href="https://platform.openai.com/account/api-key">OpenAI API key</a> above to use ChatGPT-web.
        It is required to use ChatGPT-web.
      </p>
    {/if}
  </div>
</article>
{#if apiKey}
  <article class="message is-info">
    <div class="message-body">
      Select an existing chat on the sidebar, or
      <a
        href={"#"}
        on:click|preventDefault={() => {
          activeChatId = addChat();
        }}>create a new chat</a
      >
    </div>
  </article>
{/if}
