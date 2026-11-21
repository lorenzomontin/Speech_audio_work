# Field notes

Audio is a continuous sound wave, but a computer needs numbers to represent it.

- Array: Stores those numbers—one value for each measured point on the sound wave. Each value is a sample of the sound’s amplitude. Stereo audio usually has a separate array (or channel) for the left and right sides.
- Sampling rate: Tells you how many samples are recorded per second, measured in hertz (Hz). For example, 44,100 Hz means 44,100 amplitude values per second per channel.

You need both to interpret audio correctly: the array contains the sound’s measurements, while the sampling rate tells you when each measurement occurred. Together they determine the audio’s duration and how it sounds. If you play an array at the wrong sampling rate, it can sound faster or slower and higher- or lower-pitched.


Automatic speech recognition model broadly fall into one of two categories:

1. Connectionist Temporal Classification (CTC): encoder-only models with a linear classification (CTC) head on top
2. Sequence-to-sequence (Seq2Seq): encoder-decoder models, with a cross-attention mechanism between the encoder and decoder



## Seq2seq
The encoder plays the same role as before, computing hidden-state representations of the audio inputs, while the decoder plays the role of a language model. The decoder processes the entire sequence of hidden-state representations from the encoder and generates the corresponding text transcriptions. With global context of the audio input, the decoder is able to use language modelling context as it makes its predictions, correcting for spelling mistakes on-the-fly and thus circumventing the issue of phonetic predictions.

There are two downsides to Seq2Seq models:

- They are inherently slower at decoding, since the decoding process happens one step at a time, rather than all at once
- They are more data hungry, requiring significantly more training data to reach convergence

In particular, the need for large amounts of training data has been a bottleneck in the advancement of Seq2Seq architectures for speech, as labelled speech data is difficult to come by. This all changed upon the release of Whisper. Whisper is a pre-trained model for speech recognition published in 2022 from OpenAI. Unlike its CTC predecessors, which were pre-trained entirely on un-labelled audio data, Whisper is pre-trained on a vast quantity of labelled audio-transcription data, 680,000 hours to be precise. This results in checkpoints that can be applied to over 96 languages, many of which are considered low-resource, meaning the language lacks a large corpus of data suitable for training. Of particular importance is Whisper’s ability to handle long-form audio samples, its robustness to input noise and ability to predict cased and punctuated transcriptions. This makes it a viable candidate for real-world speech recognition systems.

## Whisper 
There are two problems with forwarding this long audio sample directly to the model:
- Whisper is inherently designed to work with 30 second samples: anything shorter than 30s is padded to 30s with silence, anything longer than 30s is truncated to 30s by cutting of the extra audio, so if we pass our audio directly we’ll only get the transcription for the first 30s
- Memory in a transformer network scales with the sequence length squared: doubling the input length quadruples the memory requirement, so passing super long audio files is bound to lead to an out-of-memory (OOM) error

The way long-form transcription works in 🤗 Transformers is by chunking the input audio into smaller, more manageable segments. Each segment has a small amount of overlap with the previous one. This allows us to accurately stitch the segments back together at the boundaries, since we can find the overlap between segments and merge the transcriptions accordingly. 
The advantage of chunking the samples is that we don’t need the result of chunkii to transcribe the subsequent chunki+1i+1. The stitching is done after we have transcribed all the chunks at the chunk boundaries, so it doesn’t matter which order we transcribe chunks in. The algorithm is entirely stateless, so we can even do chunki+1i+1 at the same time as chunkii! This allows us to batch the chunks and run them through the model in parallel, providing a large computational speed-up compared to transcribing them sequentially.

Whisper is also able to predict segment-level timestamps for the audio data. These timestamps indicate the start and end time for a short passage of audio, and are particularly useful for aligning a transcription with the input audio. Suppose we want to provide closed captions for a video - we need these timestamps to know which part of the transcription corresponds to a certain segment of video, in order to display the correct transcription for that time. https://www.alphaxiv.org/abs/2212.04356