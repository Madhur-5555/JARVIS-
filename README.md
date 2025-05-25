/*

Jarvis AI Assistant - Basic Android Version

Features: Voice input, OpenAI response, Text-to-Speech output */


package com.example.jarvisai

import android.app.Activity import android.os.Bundle import android.speech.RecognizerIntent import android.speech.SpeechRecognizer import android.content.Intent import android.speech.RecognitionListener import android.speech.tts.TextToSpeech import android.widget.Button import android.widget.TextView import java.util.* import okhttp3.* import org.json.JSONArray import org.json.JSONObject import java.io.IOException

class MainActivity : Activity(), TextToSpeech.OnInitListener {

private lateinit var listenButton: Button
private lateinit var outputText: TextView
private lateinit var speechRecognizer: SpeechRecognizer
private lateinit var tts: TextToSpeech
private val client = OkHttpClient()
private val openaiApiKey = "YOUR_OPENAI_API_KEY"

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)

    listenButton = findViewById(R.id.listenButton)
    outputText = findViewById(R.id.outputText)
    tts = TextToSpeech(this, this)
    speechRecognizer = SpeechRecognizer.createSpeechRecognizer(this)

    listenButton.setOnClickListener {
        startListening()
    }

    speechRecognizer.setRecognitionListener(object : RecognitionListener {
        override fun onResults(results: Bundle?) {
            val matches = results?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)
            val spokenText = matches?.get(0) ?: ""
            outputText.text = "You said: $spokenText"
            getAIResponse(spokenText)
        }
        override fun onReadyForSpeech(params: Bundle?) {}
        override fun onBeginningOfSpeech() {}
        override fun onRmsChanged(rmsdB: Float) {}
        override fun onBufferReceived(buffer: ByteArray?) {}
        override fun onEndOfSpeech() {}
        override fun onError(error: Int) {}
        override fun onPartialResults(partialResults: Bundle?) {}
        override fun onEvent(eventType: Int, params: Bundle?) {}
    })
}

private fun startListening() {
    val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH)
    intent.putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
    intent.putExtra(RecognizerIntent.EXTRA_LANGUAGE, Locale.getDefault())
    speechRecognizer.startListening(intent)
}

private fun getAIResponse(prompt: String) {
    val json = JSONObject()
    json.put("model", "gpt-3.5-turbo")
    json.put("messages", JSONArray().put(JSONObject().put("role", "user").put("content", prompt)))

    val body = RequestBody.create(MediaType.parse("application/json"), json.toString())
    val request = Request.Builder()
        .url("https://api.openai.com/v1/chat/completions")
        .header("Authorization", "Bearer $openaiApiKey")
        .post(body)
        .build()

    client.newCall(request).enqueue(object : Callback {
        override fun onFailure(call: Call, e: IOException) {
            runOnUiThread { outputText.text = "Error: ${e.message}" }
        }

        override fun onResponse(call: Call, response: Response) {
            response.body()?.string()?.let { responseBody ->
                val responseJson = JSONObject(responseBody)
                val reply = responseJson.getJSONArray("choices")
                    .getJSONObject(0).getJSONObject("message")
                    .getString("content")

                runOnUiThread {
                    outputText.text = reply
                    speak(reply)
                }
            }
        }
    })
}

private fun speak(text: String) {
    tts.speak(text, TextToSpeech.QUEUE_FLUSH, null, "")
}

override fun onInit(status: Int) {
    if (status == TextToSpeech.SUCCESS) {
        tts.language = Locale.US
    }
}

override fun onDestroy() {
    super.onDestroy()
    tts.shutdown()
    speechRecognizer.destroy()
}

}

