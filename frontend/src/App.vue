<!-- frontend/src/App.vue -->
<template>
    <div id="app">
      <h1>Streaming Chat Test</h1>
      <deep-chat
        ref="chatElementRef"
        :connect="{
          url: 'http://localhost:8080/chat-stream',
          stream: true
        }"
        :introMessage="{ text: 'Send a streamed chat message to an example server.' }"
        style="
          height: 550px;
          border: 1px solid #ccc;
          background-color: #ffffff;
          border-radius: 4px;
          width: 400px;
        "
      />
    </div>
  </template>

  <script setup>
  import { ref } from 'vue'
  import 'deep-chat'

  const chatElementRef = ref()

  const requestInterceptor = (requestDetails) => {
    const query = requestDetails.body.messages[0].text
    requestDetails.body = { user_id: 'test-user', query: query }
    return requestDetails
  }

  const responseInterceptor = (responseDetails) => {
    console.log('Raw response:', responseDetails);
    console.log('Response body:', responseDetails.body);

    try {
      const parsedBody = JSON.parse(responseDetails.body);
      console.log('Parsed response:', parsedBody);
      return { text: parsedBody.output || 'No output received' };
    } catch (error) {
      console.error('Error parsing response:', error);
      return { text: 'Error parsing server response' };
  }
}

  const handleStreamingResponse = (response) => {
    const reader = response.body.getReader()
    let accumulatedResponse = ''

    const readStream = async () => {
      while (true) {
        const { done, value } = await reader.read()
        if (done) break

        const chunk = new TextDecoder().decode(value)
        const lines = (accumulatedResponse + chunk).split('\n')

        for (let i = 0; i < lines.length - 1; i++) {
          if (lines[i].trim() !== '') {
            const parsedChunk = JSON.parse(lines[i])
            chatElementRef.value.addResponse({ text: parsedChunk.output })
          }
        }

        accumulatedResponse = lines[lines.length - 1]
      }

      if (accumulatedResponse.trim() !== '') {
        const parsedChunk = JSON.parse(accumulatedResponse)
        chatElementRef.value.addResponse({ text: parsedChunk.output })
      }
    }

    readStream()

    return new Response(null, { status: 200 })
  }
  </script>