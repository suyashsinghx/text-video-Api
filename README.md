import streamlit as st
from moviepy.editor import VideoFileClip, AudioFileClip
from google.cloud import speech, texttospeech
import openai
import os

# Set up Google Cloud and OpenAI keys
os.environ['GOOGLE_APPLICATION_CREDENTIALS'] = '/Users/suyash/Downloads/speechtext-apis-72cfce3d9fd1.json'
openai.api_key = '22ec84421ec24230a3638d1b51e3a7dc`'

# Streamlit interface
st.title("Video Audio Replacement with AI-Generated Voice")

uploaded_file = st.file_uploader("Upload a video file", type=["mp4"])

if uploaded_file is not None:
    # Save uploaded video
    with open("uploaded_video.mp4", "wb") as f:
        f.write(uploaded_file.getbuffer())
    st.video(uploaded_file)

    # Extract audio from the uploaded video
    def extract_audio(video_path):
        clip = VideoFileClip(video_path)
        clip.audio.write_audiofile("extracted_audio.wav")
        return "extracted_audio.wav"

    st.write("Extracting audio from video...")
    audio_file = extract_audio("uploaded_video.mp4")
    
    # Step 2: Transcribe audio using Google Speech-to-Text
    def transcribe_audio(audio_path):
        client = speech.SpeechClient()
        with open(audio_path, 'rb') as audio_file:
            audio_data = audio_file.read()
        audio = speech.RecognitionAudio(content=audio_data)
        config = speech.RecognitionConfig(
            encoding=speech.RecognitionConfig.AudioEncoding.LINEAR16,
            language_code='en-US'
        )
        response = client.recognize(config=config, audio=audio)
        transcript = ' '.join([result.alternatives[0].transcript for result in response.results])
        return transcript

    st.write("Transcribing audio...")
    transcript = transcribe_audio(audio_file)
    st.write("Original Transcription:")
    st.text(transcript)

    # Step 3: Correct transcription using GPT-4
    def correct_transcription(transcript):
        prompt = f"Correct the following text by removing filler words and fixing grammatical errors: {transcript}"
        response = openai.ChatCompletion.create(
            model="gpt-4",
            messages=[{"role": "system", "content": prompt}]
        )
        return response['choices'][0]['message']['content']

    st.write("Correcting transcription...")
    corrected_transcript = correct_transcription(transcript)
    st.write("Corrected Transcription:")
    st.text(corrected_transcript)

    # Step 4: Synthesize corrected transcript using Google Text-to-Speech
    def synthesize_speech(text):
        client = texttospeech.TextToSpeechClient()
        input_text = texttospeech.SynthesisInput(text=text)
        voice = texttospeech.VoiceSelectionParams(
            language_code="en-US", name="en-US-Neural2-J"
        )
        audio_config = texttospeech.AudioConfig(audio_encoding=texttospeech.AudioEncoding.LINEAR16)
        response = client.synthesize_speech(input=input_text, voice=voice, audio_config=audio_config)
        output_audio_file = "generated_audio.wav"
        with open(output_audio_file, "wb") as out:
            out.write(response.audio_content)
        return output_audio_file

    st.write("Synthesizing speech from corrected transcription...")
    new_audio_file = synthesize_speech(corrected_transcript)
    
    # Step 5: Merge the new audio with the original video
    def merge_audio_with_video(video_path, audio_path):
        video_clip = VideoFileClip(video_path)
        new_audio = AudioFileClip(audio_path)
        final_clip = video_clip.set_audio(new_audio)
        final_output = "final_output.mp4"
        final_clip.write_videofile(final_output, codec="libx264", audio_codec="aac")
        return final_output

    st.write("Merging new audio with the original video...")
    final_video_file = merge_audio_with_video("uploaded_video.mp4", new_audio_file)

    # Provide download link for the final video
    st.write("Download the final video with AI-generated audio:")
    with open(final_video_file, "rb") as video_file:
        st.download_button(label="Download Video", data=video_file, file_name="final_output.mp4")
