---
layout: post
title:  "Of frequency delays, ghost delays and Aetherborne"
date:   2022-12-22 02:42:11 +0000
categories: Releases
---
<head>
    <script type="text/javascript" id="MathJax-script" async
  src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js">
</script></head>
<i>"god gives his hardest tasks to his strongest soldiers" - Ryan Letourneau</i>
<br>
<i>"take a look at these hands" - Stretch armstrong</i>
<br>
<h2>Prologue</h2>
A while back I was chatting to Geraint (of Signalsmith fame) about possible approaches to emulating a spring reverb fully algorithmically - he asked how I felt about latency... 
<br> 
<br>
A bit of background, I listen to a lot of rocksteady, dub and roots reggae, and was initially aiming to make a "dub spring" sort of thing, maybe with a way to "twang" the springs as it's playing back, so I 
I started looking into spectrograms of spring IRs named things like "KingTubbyIR1,wav", to get an idea of what I was even hearing and what gave it its character. 

![image](https://raw.githubusercontent.com/MeijisIrlnd/MeijisIrlnd.github.io/master/_posts/img_1.png)

The pattern here is pretty immediately obvious, what I initially called "hook" shapes losing high end each repeat, layered on top of a bunch of diffuse "wash" 
(I later found out the "hooks" are usually called <i>chirps</i>, so lets roll with that from now on). It also weirdly kinda looks like a spring if you're creative, and need to get your eyes tested, and don't think about it too hard. 
What this means though, is that the characteristic "twang" of a spring is a frequency domain chirp, and a frequency domain chirp looks like it uses some curved distribution to delay each sine/cosine component of a spectrum by a slightly different amount.
<br>
<br>
So far so good, time to spam Geraint's dm-s with screenshots of spectrograms and a bunch of question marks, and Geraint, with the patience of a very patient saint, suggested looking into.... 
<br>
<h2>Frequency Domain Delay</h2>
So STFTs right? Whats up with that? Aren't they just slightly more annoying to think about ways of splitting a signal into sines and cosines, a kind of <i>extra</i> version of a regular fft? That you unfortunately have to use for real time applications?? 
<br> 
Well yeah, and straight up I had such a horrible time trying to implement my own that I nabbed Geraint's implementation from his SignalsmithDSP library, threw it all into a single header, and rolled with that instead. 
The other (and in my opinion much radder) thing about STFTs is that you can think of them as not just sines/cosines, but as a <b>filter bank</b> of $N / 2$ bandpass filters, where $N$ is our FFT size, all running at a sample rate of our hop size $H$. All of a sudden you're into a world of <b>bands</b> 
instead of a world of boring regular floating point samples, and you can pretty much apply any processing there you want, with the caveat of everything now needing to be a complex number for any maths you may need to do. 
<br>
<br>
An aside with this that's still tripping me up, is that while yeah, all thats true, I'm glossing over a step. Secure your socks firmly to your sock receptacles, we're doing <b>`m a t h s`</b>.
<br><br> 
So (and depending on your background with this stuff you might have to trust me here) you can write the formula for an STFT as follows: <br>
$H_m(\omega_k) = \sum_{-\infty}^{\infty}[x(n)e^{-j\omega_kn}]w(n - m)$
<br> 
So what's going actually going on here? First lets define some terms. $X_m(\omega_k)$ is the spectrum we're producing, $x(n)$ is our time domain signal we want to transform, $e^{-j}$ is the "complex exponential" (euler's number to some imaginary power, $\omega_k$ is a term relating sample rate and fft size, $n$ is a time, $m$ is a delay, and $w(x)$ is our window function. We're essentially repeatedly calculating the DTFT, and then sliding over by $H$ samples. 
<br><i><sub>side note: from euler's formula, $e^{j\theta} = cos(\theta) + j sin(\theta)$, and in the case of the complex exponential, $e^{-j\omega t} = cos(\omega t) + jsin(\omega t)$, so there's an argument to be made for writing the entire stft out using sin and cos instead of euler's number for clarity and intuitiveness, but I digress..</sub></i>
<br><br>take a breath<br><br> 
Cool, we know roughly what the symbols under the dresser mean, and have ballpark idea of what the stft does, why does this matter? The <b>thing is</b> with this configuration, our signal is being kept constant, and we're sliding a window along it. That's kinda dumb right? This thing is supposed to be realtime. So what we can do instead is rephrase as<br>
$H_m(\omega_k) = \sum_{-\infty}^{\infty}[x(n + m)e^{-j\omega_kn}]w(n)$
<br>
and now we're sliding the <i>signal</i> along, while the window is held in place. The problem is, now the $e^{-j\omega_kn}$ term doesn't get adjusted, as technically it should be $e^{-j\omega_kn + m}$. So to compensate, we need to apply a <i>phase twist</i> to our band when we do our processing.
<br><br> 
So "applying a phase twist" sounds pretty fuckin metal right? So it's pretty disappointing that it literally just involves multiplying by another complex exponential. That being said, finding WHICH complex exponential to multiply by is where I've been getting caught, so take the following with a grain of salt, (and I'd be remissed if I didn't cite Geraint for this, especially the part about actually calculating the phase shift, I'm essentially rephrasing his explanation).
<br><br>First off, it's kinda helpful to understand WHY a phase shift is a multiplication by a complex exponential. Because we're dealing with complex numbers, if we move to polar coordinates its pretty intuitive to think of these complex numbers as rotations around the unit circle in the imaginary plane, but even more than that, as mentioned earlier, using euler's formula we can show that all of this abstract euler's number business is just talking about a combination of sines and cosines, which, yknow, sines and rotations go pretty hand in hand. 
all of this to say that multiplying by $e^{j\theta}$ will <i>spin</i> your value along the unit circle by theta radians. So we're essentially trying to "spin" our bin to DC (the 0th bin's phase) to do our processing, then spin it the exact inverse to get it back to normal. 
<br><br>
For example then, lets say we're dealing with a frequency of $0.2 f_s$. We can find its phase using $2\pi b_i 0.2$, where $b_i$ is our bin index. We need to account for hop size here, seeing as we're using an stft which has overlapping and other self-hatred triggering properties. so to link hop size into our phase calculation, we can say that the phase at block number $p$ is $2\pi (pH) 0.2$ ($p$ increments each hop, just a way of dealing with the overlap). We can get the central frequency of bin $b_i$ with $b_i / N$. 
So combining the two, we can find the phase of a bin with $2\pi (p * H) * (b_i / N)$, lets call that $S$, for <b>shift</b>. SO then to do our phase twist, we can just multiply our value by $e^{-j S}$, do our processing, then multiply by $e^{j S}$ to get back to the original phase. 
<br><br>Here's a horrible drawing to illustrate it:<br>
![img_7.png](img_7.png)
<br>
<br>
<br>
take a breath 


<br> 

<br>
  <script>
  MathJax = {
    tex: {inlineMath: [['$', '$'], ['\\(', '\\)']]}
  };
  </script>
  <script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-chtml.js"></script>


