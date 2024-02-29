# Early Reflections - A Fictional Audio Plugin Website

### Summary

This project is the website for a fictional audio plugin company, **Early Reflections**.

The key feature of this site is the interactive demos of each audio plugin. With one pre-loaded sample, users can control the parameters of various audio effects and toggle them individually to combine effects.

### Motivation

I have been curious about audio plugin software since starting programming a couple years ago, and this project started for simple experiments with this in the browser.

Once I realized how powerful the Web Audio API is, I decided it would be a fun project to completely flesh out. I also threw in some 3D programming in the form of an audio visualizer, as I was learning about 3D web programming at the time.

# Features

The site has a couple features for you to play around with:

- Audio effect bypassing / toggling. Make an effect process the audio signal or not and compare the difference.
- Control audio parameters either by scrolling on the knobs, or inputting them manually.
- A sleek 3D Audio Visualizer

# Technical

### How to create audio effects in Javascript

I'm deciding to write a little about how you can create audio effects in Javascript as there aren't many good resources online and I wish there was some write ups when I started this project.

Your HTML needs to include an audio and source element.

```html
<audio>
  <source src="path/to/source" />
  Your browser does not support the audio element.
</audio>
```

To add reverb to this audio element (you will need to fetch and decode your impulse response and set your ConvolverNode's buffer to the response buffer first), you can use the following code:

```javascript
const audioContext = new AudioContext(); // note to combine multiple plugins, must share this between them
/* Create nodes for effect*/
const dryGain = ac.createGain();
const wetGain = ac.createGain();
const preDelayNode = ac.createDelay();

/* Connect nodes to source and audio context */
source.connect(dryGain);
dryGain.connect(ac.destination);

source.connect(preDelayNode);
preDelayNode.connect(convolver);
convolver.connect(wetGain);
wetGain.connect(ac.destination);
```

#### Technology Stack
NextJS, Typescript + TailwindCSS.

#### Code
https://github.com/jamisonrobey/mock-audio-plugin-website
