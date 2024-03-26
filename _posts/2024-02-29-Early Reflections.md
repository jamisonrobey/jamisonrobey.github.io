# Early Reflections - A Fictional Audio Plugin Website

## Visit the website:

### [Early Reflections](https://earlyreflections.pages.dev/)

## Summary

Early Reflections is a website for a fictional audio plugin company that offers a unique feature: interactive demos of their plugins with pre-loaded samples. This allows users to try out the audio plugins right in their browser, attracting potential customers.

## Motivation

This project began as a simple experiment to see if I could create audio effects in a browser environment. Once I realised the power of the Web Audio API, I decided to create a more ambitious project, which ultimately became Early Reflections.

# Features

The site offers several features for users to explore:

- Audio effect bypassing / toggling:
  - Users can choose to process the audio signal with an effect or bypass it to compare the difference.
- Control audio parameters:
  - Users can adjust parameters either by scrolling on the knobs or inputting them manually.
- A sleek 3D Audio Visualiser:
  - Adds a visual element to the audio experience.

# Technical

### How to create audio effects in Javascript

I'm deciding to write a little about how you can create audio effects in Javascript as there aren't any resources online and I wish there was some write ups when I started this project.

Your markup needs to include an audio and source element.

```html
<audio>
  <source src="path/to/source" />
  Your browser does not support the audio element.
</audio>
```

A simple example is a compressor; as noted above the Web Audio API is powerful, so setting this up to work on our audio element is very simple.

```javascript
const audioSource = // audio source for DOM, get through DOM access or using React patterns T<MediaElementAudioSourceNode>;
const audioContext = new AudioContext(); // note to combine multiple plugins, must share this between them

// create compressor
const compressor = ac.createDynamicsCompressor();

// default values - best to not leave default otherwise can cause strange processing
compressor.threshold.value = -30;
compressor.knee.value = 30;
compressor.ratio.value = 8;
compressor.attack.value = 0.01;
compressor.release.value = 0.1;

// connect compressor to audio source
audioSource.connect(compressor);
```

# Limitations / Improvements

A limitation of the site is I did not consider the effect chaining. This is an oversight I didn't realise until far into the project, which could be added at a later date, but is not crucial to the site's functionality.

Ideally, it would be good to have standard effect chaining i.e. reverb effect is the last effect, and processes the single from all effects first. In the current implementation, the effects do not chain into each other properly.

# Technology Stack

NextJS, Typescript + TailwindCSS.

# Code

[Repo](https://github.com/jamisonrobey/mock-audio-plugin-website)
[Deployment](earlyreflections.pages.dev)
