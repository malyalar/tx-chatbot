<script lang="ts">
  //import { fetchEventSource } from "@microsoft/fetch-event-source";

  import { apiKeyStorage, chatsStorage, addMessage, clearMessages } from "./Storage.svelte";
  import type { Request, Response, Message, Settings } from "./Types.svelte";
  import Code from "./Code.svelte";

  import { afterUpdate, onMount } from "svelte";
  import SvelteMarkdown from "svelte-markdown";

  //import { similarity } from 'string-cosine-similarity';

  export let chatId: number;
  let updating: boolean = false;

  let input: HTMLTextAreaElement;
  let settings: HTMLDivElement;
  let recognition: any = null;
  let recording = false;

  // -------------------------------------
  // Define the interface for the facts
  interface Fact { fact: string; }

  // Define the interface for the similarity score
  interface SimilarityScore {fact: string; score: number;}
// -------------------------------------

  const settingsMap: Settings[] = [
    {
      key: "temperature",
      name: "Sampling Temperature",
      default: 1,
      type: "number",
    },
    {
      key: "top_p",
      name: "Nucleus Sampling",
      default: 1,
      type: "number",
    },
    {
      key: "n",
      name: "Number of Messages",
      default: 1,
      type: "number",
    },
    {
      key: "max_tokens",
      name: "Max Tokens",
      default: 0,
      type: "number",
    },
    {
      key: "presence_penalty",
      name: "Presence Penalty",
      default: 0,
      type: "number",
    },
    {
      key: "frequency_penalty",
      name: "Frequency Penalty",
      default: 0,
      type: "number",
    },
  ];

  $: chat = $chatsStorage.find((chat) => chat.id === chatId);
  const token_price = 0.000002; // $0.002 per 1000 tokens

  // Focus the input on mount
  onMount(() => {
    input.focus();

    // Try to detect speech recognition support
    if ("SpeechRecognition" in window) {
      // @ts-ignore
      recognition = new SpeechRecognition();
    } else if ("webkitSpeechRecognition" in window) {
      // @ts-ignore
      recognition = new webkitSpeechRecognition();
    }

    if (recognition) {
      recognition.interimResults = false;
      recognition.onstart = () => {
        recording = true;
      };
      recognition.onresult = (event) => {
        // Stop speech recognition, submit the form and remove the pulse
        const last = event.results.length - 1;
        const text = event.results[last][0].transcript;
        input.value = text;
        recognition.stop();
        recording = false;
        submitForm(true);
      };
    } else {
      console.log("Speech recognition not supported");
    }
  });



  // Scroll to the bottom of the chat on update
  afterUpdate(() => {
    // Scroll to the bottom of the page after any updates to the messages array
    window.scrollTo(0, document.body.scrollHeight);
    input.focus();
  });

  // Marked options
  const markedownOptions = {
    gfm: true,
    breaks: true,
  };

  const sendRequest = async (messages: Message[]): Promise<Response> => {
    // Send API request
    // Show updating bar
    updating = true;

    let response: Response;

    try {
      const request: Request = {
        model: "gpt-3.5-turbo",
        // Submit only the role and content of the messages, provide the previous messages as well for context
        messages: messages
          .map((message): Message => {
            const { role, content } = message;
            return { role, content };
          })
          // Skip error messages
          .filter((message) => message.role !== "error"),

        // Provide the settings by mapping the settingsMap to key/value pairs
        ...settingsMap.reduce((acc, setting) => {
          const value = (settings.querySelector(`#settings-${setting.key}`) as HTMLInputElement).value;
          if (value) {
            acc[setting.key] = setting.type === "number" ? parseFloat(value) : value;
          }
          return acc;
        }, {}),
      };

      response = await (
        await fetch("https://api.openai.com/v1/chat/completions", {
          method: "POST",
          headers: {
            Authorization: `Bearer ${$apiKeyStorage}`,
            "Content-Type": "application/json",
          },
          body: JSON.stringify(request),
        })
      ).json();
    } catch (e) {
      response = { error: { message: e.message } } as Response;
    }

    // Hide updating bar
    updating = false;

    return response;
  };



// -------------------------------------------------------------------------------------------
// ----------------------------STRING COSINE SIMILARITY CALCULATION --------------------------
// -------------------------------------------------------------------------------------------


// tokenize function
const tokenize = (text) => {
    return text.split(/[^A-Za-z0-9]+/);
  };

// cosineLib module
const cosineLib = {
    // Convert strings into arrays (Tokenize, toLowerCase) //
    getTokens(str) {
      return tokenize(str)
        .map((word) => word.toLowerCase());
    },

    // Find Common words Frequency //
    computeFrequency(arr, commons) {
      return commons.map((word) =>
        arr.reduce((f, element) => (element === word ? f + 1 : f), 0)
      );
    },

    // Calculate Vector A.B //
    computeVectorAB(v1, v2) {
      return v1.reduce((sum, f, index) => sum + f * v2[index], 0);
    },

    // Calculate ||a|| and ||b|| //
    absVector(v) {
      return Math.sqrt(v.reduce((sum, f) => sum + f * f, 0));
    },

    // Cosine Similarity //
    similarity(vAB, a, b) {
      return vAB / (a * b);
    },
  };

// cosineSimilarity module
const cosineSimilarity = {
    calculateSimilarity(string1, string2) {
      
      //console.log("testing new pair");                                            // log statement
      //console.log("comparison string 1: ", string1);                                            // log statement
      //console.log("comparison string 2: ", string2);                                            // log statement
      
      const arr1 = cosineLib.getTokens(string1);
      const arr2 = cosineLib.getTokens(string2);

      //console.log("tokenized string 1: ", arr1);                                            // log statement
      //console.log("tokenized string 2: ", arr2);                                            // log statement

      // Define Commons Array //
      const commons = arr1;

      // Word Frequency as Vectors //
      const v1 = cosineLib.computeFrequency(arr1, commons);
      const v2 = cosineLib.computeFrequency(arr2, commons);

      //console.log("wordfreqasvector string 1: ", v1);                                            // log statement
      //console.log("wordfreqasvector string 2: ", v2);                                            // log statement

      // Calculate Vector A.B //
      const vAB = cosineLib.computeVectorAB(v1, v2);

      //console.log("dot product string 1.2: ", vAB);                                            // log statement

      // Abs Vector A and B //
      const a = cosineLib.absVector(v1);
      const b = cosineLib.absVector(v2);

      //console.log("abs vector string 1: ", a);                                            // log statement
      //console.log("abs vector string 2: ", b);                                            // log statement

      // Cosine Similarity //
      const similarity = cosineLib.similarity(vAB, a, b);
      //console.log("similarity: ", vAB);                                                   // log statement

      // Check if the denominator is 0
      if (a === 0 || b === 0) {
        return 0;
      }

      return similarity;
    },
  };






// -------------------------------------------------------------------------------------------
// -------------------------------------------------------------------------------------------
// -------------------------------------------------------------------------------------------


  // Define the function to clean the input text

  async function loadStopwords() {
      const stopwordNumbersUrl = '/src/lib/textAssets/stopword_numbers.txt';
      const stopwordNumbersResponse = await fetch(stopwordNumbersUrl);
      const stopwordNumbersText = await stopwordNumbersResponse.text();
      const stopwordNumbers = stopwordNumbersText.replace(/\r/g, '').split('\n');

      // Repeat for stopword_symbols.txt 
      const stopwordSymbolsUrl = '/src/lib/textAssets/stopword_symbols.txt';
      const stopwordSymbolsResponse = await fetch(stopwordSymbolsUrl);
      const stopwordSymbolsText = await stopwordSymbolsResponse.text();
      const stopwordSymbols = stopwordSymbolsText.replace(/\r/g, '').split('\n');

      // Repeat for stopword_min3.txt
      const stopwordMin3Url = '/src/lib/textAssets/stopword_min3.txt';
      const stopwordMin3Response = await fetch(stopwordMin3Url);
      const stopwordMin3Text = await stopwordMin3Response.text();
      const stopwordMin3 = stopwordMin3Text.replace(/\r/g, '').split('\n');

      console.log(stopwordNumbers);
      console.log(stopwordSymbols);
      console.log(stopwordMin3);

      return {
        stopwordNumbers,
        stopwordSymbols,
        stopwordMin3,
      };
    }

  async function cleanText(txtin: string): Promise<string> {
      // Remove contractions
      txtin = txtin.replace(/\'m/g,' am')
      txtin = txtin.replace(/\'re/g,' are')
      txtin = txtin.replace(/\blet\'s\b/g,'let us')
      txtin = txtin.replace(/\'s/g,' is')
      txtin = txtin.replace(/ain\'t/g,' is not it')
      txtin = txtin.replace(/n\'t/g,' not')
      txtin = txtin.replace(/\'ll/g,' will')
      txtin = txtin.replace(/\'d/g,' would')
      txtin = txtin.replace(/\'ve/g,' have')
      txtin = txtin.replace(/\lemme/g,' let me')
      txtin = txtin.replace(/\gimme/g,' give me')
      txtin = txtin.replace(/\wanna/g,' want to')
      txtin = txtin.replace(/\gonna/g,' going to')
      txtin = txtin.replace(/r u /g,'are you')
      txtin = txtin.replace(/\bim\b/g,'i am')
      txtin = txtin.replace(/\bwhats\b/g,'what is')
      txtin = txtin.replace(/\bwheres\b/g,'where is')
      txtin = txtin.replace(/\bwhos\b/g,'who is')

    // Load the stopwords
    const stopwords = await loadStopwords();
    const stopwordNumbers = stopwords.stopwordNumbers;
    const stopwordSymbols = stopwords.stopwordSymbols;
    const stopwordMin3 = stopwords.stopwordMin3;

    // Remove numbers
    for (let i = 0; i < stopwordNumbers.length; i++) {
        const re = new RegExp(`\\b${stopwordNumbers[i]}\\b`, 'g');
        txtin = txtin.replace(re, '');
        txtin = txtin.replace(/[0-9]/g, ' ').replace(/ \. /g, ' ');
      }

      // Remove words (very long list!)
      for (let i = 0; i < stopwordMin3.length; i++) {
        const re = new RegExp(`\\b${stopwordMin3[i]}\\b`, 'g');
        txtin = txtin.replace(re, '');
      }

      // Remove symbols
      for (let i = 0; i < stopwordSymbols.length; i++) {
        const re = new RegExp(`\\${stopwordSymbols[i]}`, 'g');
        txtin = txtin.replace(re, '');
      }
                                      
      //console.log(txtin.trim().toLowerCase());                                                      // log statement
      return txtin.trim().toLowerCase();
  }

/*
  const getFactsFromFile = async (filePath: string): Promise<Fact[]> => {
      const response = await fetch(filePath);
      const text = await response.text();
      const facts = text.split("\n").map((fact) => ({ fact }));

      console.log("List of all facts: " +facts);                                                   // log statement
      return facts;
     };
*/

  export const submitConcourse = async (inputMessage: string): Promise<string[]> => {
    return new Promise((resolve, reject) => {
        // Read the facts file
        fetch('/src/lib/textAssets/facts.txt')
        .then(response => response.text())
        .then(async (factsText) => { // use async/await to call cleanText() function

          // Clean up the input message
          const cleanedSentence = await cleanText(inputMessage);

          // Split the text into facts
          const facts = factsText.split('\n').filter(fact => fact.trim() !== '');
          //console.log("List of all facts: " + facts);                                            // log statement

          // Compute the similarity between the cleaned sentence and each fact using cosine similarity
          const similarityScores: SimilarityScore[] = facts.map((fact) => {
            const factScore = cosineSimilarity.calculateSimilarity(cleanedSentence, fact.trim().toLowerCase()); // use cosineSimilarity() instead of similarity()
            console.log(fact, factScore);                                            // log statement
            return { fact, score: factScore };
          });

          // Sort the similarity scores in descending order based on the score
          similarityScores.sort((a, b) => b.score - a.score);

          // Pick the top 3 most relevant facts
          const numFactsToReturn = 3;
          const topFacts = similarityScores
            .slice(0, numFactsToReturn)
            .map((score) => score.fact);

          console.log("List of top facts: " + topFacts);                                            // log statement
          resolve(topFacts);
        })
        .catch(error => reject(error));
    });
  };

  // -------------------------------------------------------------------------------------------
  // -------------------------------------------------------------------------------------------
  // -------------------------------------------------------------------------------------------










  const submitForm = async (recorded: boolean = false): Promise<void> => {

    // reminder header message, as system messages are under-emphasized in the current model
    //const headerMessage: Message = { role: "system", content: "You are a chatbot for talking about kidney transplant only.\
    //                                                          Decline requests to talk about other domains. Here are some facts about kidney transplant:"};
    //addMessage(chatId, headerMessage);
    
    // Call submitConcourse function and await its result
    const facts = await submitConcourse(input.value);
    // Convert the facts to a string of concatenated facts
    const concatenatedFacts = facts.join("\n");
    console.log(concatenatedFacts);                                                 // log statement
    const primerMessage: Message = { role: "system", content: concatenatedFacts};
    addMessage(chatId, primerMessage);

    // Compose the input message
    const inputMessage: Message = { role: "user", content: input.value };
    addMessage(chatId, inputMessage);
    console.log(input.value);                                                       // log statement
    
    // Clear the input value
    input.value = "";
    input.blur();

    // Resize back to single line height
    input.style.height = "auto";

    const response = await sendRequest(chat.messages);

    if (response.error) {
      addMessage(chatId, {
        role: "error",
        content: `Error: ${response.error.message}`,
      });
    } else {
      response.choices.map((choice) => {
        choice.message.usage = response.usage;
        // Remove whitespace around the message that the OpenAI API sometimes returns
        choice.message.content = choice.message.content.trim();
        addMessage(chatId, choice.message);
        // Use TTS to read the response, if query was recorded
        if (recorded && "SpeechSynthesisUtterance" in window) {
          const utterance = new SpeechSynthesisUtterance(choice.message.content);
          speechSynthesis.speak(utterance);
        }
      });
    }
  };

  const suggestName = async (): Promise<void> => {
    const suggestMessage: Message = {
      role: "user",
      content: "Can you give me a 5 word summary of this conversation's topic?",
    };
    addMessage(chatId, suggestMessage);

    const response = await sendRequest(chat.messages);

    if (response.error) {
      addMessage(chatId, {
        role: "error",
        content: `Error: ${response.error.message}`,
      });
    } else {
      response.choices.map((choice) => {
        choice.message.usage = response.usage;
        addMessage(chatId, choice.message);
        chat.name = choice.message.content;
        chatsStorage.set($chatsStorage);
      });
    }
  };

  const deleteChat = () => {
    if (confirm("Are you sure you want to delete this chat?")) {
      chatsStorage.update((chats) => chats.filter((chat) => chat.id !== chatId));
      chatId = null;
    }
  };

  const showSettings = () => {
    settings.classList.add("is-active");
  };

  const closeSettings = () => {
    settings.classList.remove("is-active");
  };

  const clearSettings = () => {
    settingsMap.forEach((setting) => {
      const input = settings.querySelector(`#settings-${setting.key}`) as HTMLInputElement;
      input.value = "";
    });
  };

  const recordToggle = () => {
    // Check if already recording - if so, stop - else start
    if (recording) {
      recognition?.stop();
      recording = false;
    } else {
      recognition?.start();
    }
  };
</script>

<nav class="level chat-header">
  <div class="level-left">
    <div class="level-item">
      <p class="subtitle is-5">
        {chat.name || `Chat ${chat.id}`}
        <a
          href={"#"}
          class="greyscale ml-2 is-hidden has-text-weight-bold editbutton"
          title="Rename chat"
          on:click|preventDefault={() => {
            let newChatName = prompt("Enter a new name for this chat", chat.name);
            if (newChatName) {
              chat.name = newChatName;
              chatsStorage.set($chatsStorage);
            }
          }}
        >
          ✏️
        </a>
        <a
          href={"#"}
          class="greyscale ml-2 is-hidden has-text-weight-bold editbutton"
          title="Suggest a chat name"
          on:click|preventDefault={suggestName}
        >
          💡
        </a>
        <a
          href={"#"}
          class="greyscale ml-2 is-hidden editbutton"
          title="Delete this chat"
          on:click|preventDefault={deleteChat}
        >
          🗑️
        </a>
      </p>
    </div>
  </div>

  <div class="level-right">
    <p class="level-item">
      <button
        class="button is-warning"
        on:click={() => {
          clearMessages(chatId);
        }}><span class="greyscale mr-2">🗑️</span> Clear messages</button
      >
    </p>
  </div>
</nav>

{#each chat.messages as message}
  {#if message.role === "user"}
    <article
      class="message is-info user-message"
      class:has-text-right={message.content.split("\n").filter((line) => line.trim()).length === 1}
    >
      <div class="message-body content">
        <a
          href={"#"}
          class="greyscale is-pulled-right ml-2 is-hidden editbutton"
          on:click={() => {
            input.value = message.content;
            input.focus();
          }}
        >
          ✏️
        </a>
        <SvelteMarkdown
          source={message.content}
          options={markedownOptions}
          renderers={{
            code: Code,
          }}
        />
      </div>
    </article>
  {:else if message.role === "system" || message.role === "error"}
    <article class="message is-danger">
      <div class="message-body content">
        <SvelteMarkdown
          source={message.content}
          options={markedownOptions}
          renderers={{
            code: Code,
          }}
        />
      </div>
    </article>
  {:else}
    <article class="message is-success">
      <div class="message-body content">
        <SvelteMarkdown
          source={message.content}
          options={markedownOptions}
          renderers={{
            code: Code,
          }}
        />
        {#if message.usage}
          <p class="is-size-7">
            This message was generated using <span class="has-text-weight-bold">{message.usage.total_tokens}</span>
            tokens ~=
            <span class="has-text-weight-bold">${(message.usage.total_tokens * token_price).toFixed(6)}</span>
          </p>
        {/if}
      </div>
    </article>
  {/if}
{/each}

{#if updating}
  <progress class="progress is-small is-dark" max="100" />
{/if}

<form class="field has-addons has-addons-right" on:submit|preventDefault={() => submitForm()}>
  <p class="control is-expanded">
    <textarea
      class="input is-info is-focused chat-input"
      placeholder="Type your message here..."
      rows="1"
      on:keydown={(e) => {
        // Only send if Enter is pressed, not Shift+Enter
        if (e.key === "Enter" && !e.shiftKey) {
          submitForm();
          e.preventDefault();
        }
      }}
      on:input={(e) => {
        // Resize the textarea to fit the content - auto is important to reset the height after deleting content
        input.style.height = "auto";
        input.style.height = input.scrollHeight + "px";
      }}
      bind:this={input}
    />
  </p>
  <p class="control" class:is-hidden={!recognition}>
    <button class="button" class:is-pulse={recording} on:click|preventDefault={recordToggle}
      ><span class="greyscale">🎤</span></button
    >
  </p>
  <p class="control">
    <button class="button" on:click|preventDefault={showSettings}><span class="greyscale">⚙️</span></button>
  </p>
  <p class="control">
    <button class="button is-info" type="submit">Send</button>
  </p>
</form>

<svelte:window
  on:keydown={(event) => {
    if (event.key === "Escape") {
      closeSettings();
    }
  }}
/>

<!-- svelte-ignore a11y-click-events-have-key-events -->
<div class="modal" bind:this={settings}>
  <div class="modal-background" on:click={closeSettings} />
  <div class="modal-card">
    <header class="modal-card-head">
      <p class="modal-card-title">Settings</p>
    </header>
    <section class="modal-card-body">
      {#each settingsMap as setting}
        <div class="field is-horizontal">
          <div class="field-label is-normal">
            <label class="label" for="settings-temperature">{setting.name}</label>
          </div>
          <div class="field-body">
            <div class="field">
              <input
                class="input"
                type={setting.type}
                id="settings-{setting.key}"
                placeholder={String(setting.default)}
              />
            </div>
          </div>
        </div>
      {/each}
    </section>

    <footer class="modal-card-foot">
      <button class="button is-info" on:click={closeSettings}>Close settings</button>
      <button class="button" on:click={clearSettings}>Clear settings</button>
    </footer>
  </div>
</div>
