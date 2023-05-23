---
layout: post
title:  "A matrix based approach to modifying reverb structures"
date:   2023-05-22 02:42:11 +0000
categories: Releases
---
<head>
    <script type="text/javascript" id="MathJax-script" async
  src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js">
</script></head>

<i>I dreamed a dream and now that dream has gone from me<i> - Stretch Armstrong

<h1>Prologue</h1>
Of all the audio effects you can write, probably the most mysterious (and possibly gatekept) is a reverb. It's not that the techniques aren't publicised, or that there isn't a vast amount of eye watering literature about it, but there's a lot of hand waving about specifics, and understandably so - someone might spend 6 months tuning delay times, allpass coefficients, whatever, and not want someone to be able to recreate all their work within an hour,  not to mention the corporate overlord factor, Jon Datorro, and his unusually readable and fun series of papers was apparently the subject of a court case by either Ensoniq or Lexicon, for sharing company secrets (if anyone can find the transcript and send it to me I'll buy you a pint). 
<br>
So all of this to say, it's pretty hard to know where to start, if you want to write your own, or know how to tweak an existing well defined reverb structure, so first lets take a bit of a dive into 

<h1>Reverb architectures</h1>

In general, digital approaches to reverb are usually either delay based, convolutional, or physically modelled. For this blog post I'll be focussing on delay based structures, as the other two don't really fit for the perspective shift I'm going for. <br> 
Within that category then, the main architectures I've come across are Schroeder, Figure 8, Nested Allpass (ala Gardner's hall reverbs), a super long chain of allpasses, delays and friends, and FDN. In college (admittedly a music production degree) we were only really shown the Schroeder architecture, and it seems to be a trend to introduce that as "digital reverb 101", the problem being Schroeder wrote those papers 60 years ago, they were hugely influential on digital reverb (the man invented allpass filters...), but sound a bit... shit* (sorry Schroeder xxxx) 
<br>
<br>
Of all of these, oxymoronically, the FDN (Feedback Delay Network, or Free Danish Nationalists), probably the most <i>theoretically</i> complex, is probably actually the easiest to get a good sound out of without weeks or months of tuning. Geraint Luff (of Signalsmith fame)'s "Let's Write A Reverb" [article](https://signalsmith-audio.co.uk/writing/2021/lets-write-a-reverb/) to me at least is one of the most interesting, well written and useful articles on the internet, in which he details a novel approach to writing an FDN that essentially allows you to <s>fuck around and find out</s> use random coefficients and delay times to get a decent sounding reverb. I'll be referring to ideas from it for this whole article, so if you haven't already, go read that and come back.
<br>
<br>
Done? Good, my favourite bit is when he put the picture of the banana in the feedback loop, Gerzon could <i>never</i>.
<br>

<sub>*To be fair, Sean Costello of Valhalla DSP fame has an article about how maybe they're not actually that shit, trust him not me and check out the article [here](https://valhalladsp.com/2009/05/30/schroeder-reverbs-the-forgotten-algorithm/)
</sub>

<h1>The Matrix (not that one)</h1>
A quick once over then. An FDN can be broken into stages - splitting, input diffusion and feedback loop (Also splitting the input into N channels, reading from some (or all, with weighting) of these channels to form your output, filtering, etc etc). What we're gonna focus on for the time being though is the feedback loop. In Geraint's FDN it will end up looking something like this: 
![image](https://scontent.fdub3-2.fna.fbcdn.net/v/t1.15752-9/343715384_6380866878669880_2474117304829896249_n.png?_nc_cat=106&ccb=1-7&_nc_sid=ae9488&_nc_ohc=wfJkpD3YCw4AX9qy49s&_nc_oc=AQlOU-kt1DYCDZroK_wr7ES7CC1z1bRwMs8Swg-Uex4ihH38o2Dyev-yDe68H2YtZXM&_nc_ht=scontent.fdub3-2.fna&oh=03_AdR3wz18k8rW-ENkQ9c0ZshR8qh7eHQ1tnlMlWAZb9qyGg&oe=64935033)
<br><br>
The mix matrix he recommends here is a [householder matrix](https://nhigham.com/2020/09/15/what-is-a-householder-matrix/), which in the 4x4 case might look something like this pragmatic lil guy*:<br>
<br>

$$ A = \frac{1}{2} \begin{bmatrix}
\phantom{-}1 & -1 & -1 & -1\\  
-1 & \phantom{-}1 & -1 & -1\\
-1 & -1 &  \phantom{-}1 & -1\\
-1 & -1 & -1 & \phantom{-}1\\
\end{bmatrix}
$$

<br>
Cool so what is that actually doing? In Geraint's article, the purpose of that matrix is scatter the echoes around the channels, to increase echo density. But for the sake of argument, lets say we just want to the output of each FDN channel to feedback into itself, not worrying about intermixing or any of that fancy stuff, so what we essentially have on a each channel is a super simple delay loop, like this:
<br>
![image](https://scontent.fdub3-2.fna.fbcdn.net/v/t1.15752-9/343416075_983432222673681_2763912434955997882_n.png?_nc_cat=108&ccb=1-7&_nc_sid=ae9488&_nc_ohc=1wbg7nHqBu0AX8HXW7q&_nc_ht=scontent.fdub3-2.fna&oh=03_AdT9rin_KOjU81xBApuJ8fj9mUDZSqaNyxHKG5Y523ECRg&oe=649356F4)

<br>
<br>

To express this as a matrix, lets take a leap of faith, and label our columns as the outputs for our channels, and our rows as the inputs for our channels. Let's also fill in an x for the input we want our output routed to, and an O for the other slots - in this case, we know that we want the output of channel 1 routed to the input of channel 1, the output of channel 2 routed to the input of channel 2, etc. So we end up with the most one sided game of connect four the world has ever encountered: 
<br>
![image](https://scontent.fdub3-2.fna.fbcdn.net/v/t1.15752-9/343747352_121047027657542_7977978219463630559_n.png?_nc_cat=101&ccb=1-7&_nc_sid=ae9488&_nc_ohc=ilWm39DGpGkAX-0UATx&_nc_ht=scontent.fdub3-2.fna&oh=03_AdTDrI63M9TY6BwBtTF5lLweCONSVHXYF-512q9CRIgtTA&oe=6493656E)
<br>

Now, lets switch from the nice graphics to inline maths, and replace our X's with 1s and our O's with 0s: 

$$ A = \begin{bmatrix}
1 & 0 & 0 & 0\\  
0 & 1 & 0 & 0\\
0 & 0 & 1 & 0\\
0 & 0 & 0 & 1\\
\end{bmatrix}
$$

<br> 
Buckle up boys, we got ourselves a mix matrix**!!!!<br>
That matrix in and of itself isn't very interesting, it just keeps all our channels independent which obviously isn't condusive to dense reverb, but the crucial part is the labelling of columns as outputs and channels as inputs. <br> 
<br> 
So let's put our labelling goggles on and go back to the 4x4 householder matrix from Geraint's algorithm: 
<br>
![image](https://scontent.fdub3-2.fna.fbcdn.net/v/t1.15752-9/345027210_3063068820661454_4315982669073535712_n.png?_nc_cat=102&ccb=1-7&_nc_sid=ae9488&_nc_ohc=H2dbP0-WFv8AX_20R5p&_nc_ht=scontent.fdub3-2.fna&oh=03_AdSWTliofkDNPs9ZTPQxvAqSWLaBBprY8dgSOrqNo2TbWQ&oe=64938005)

<br>
Following the diagonal from the top left to the bottom right, you can see the same structure as our shitty little independent channel matrix, but the 0s have been replaced by -1s. So effectively, it's routing each output to each other channel's input, but flipping the sign. The entire structure is also multiplied by $\frac{1}{2}$ presumably to preserve orthogonality (which I'll get to in a second..)
<br>
<br> 
To summarize then, the mix matrix is there to handle routing between the channels. An identity matrix (1s on the diagonal, 0s everywhere else) will keep each channel independent, and something like a householder will intermix each channel between each other channel's inputs pretty significantly. Armed with this knowledge, lets take a trip down other reverb structure lane. First stop, 
<br> 
<sub> *Turns out a 4x4 householder is actually 1/2 times a hadamard matrix, very cool
<br> 
** which turns out to be an identity matrix!
</sub>
<h1> Jon Datorro (or in English, "The Raging Bull") </h1>
Datorro's 1997 paper, ["Effect Design Part 1: Reverberator and Other Filters"](https://ccrma.stanford.edu/~dattorro/EffectDesignPart1.pdf) is pretty unique in tone, walking the metaphorical tightrope between engaging and dense in theory. In it, he details (delay times, coefficients and all) an approach apparently "inspired by" the work of David Griesinger (of Lexicon fame). I've read rumours that the design published in his Effect Design paper was arrived at by reverse engineering one of the Lexicon models (potentially 224, confirmed to be designed by Griesinger), and the tips section of the freeverb site has this to offer: 

> The room reverb looks just like Griesinger 'plate' reverb covered by the AES paper 'Effect Design - Part 1, Reverberator and other filters'. Except that there is one additional stage of diffusion and one additional stage of delay through each leg of the tank. Also, the placement of the damping low-pass filters are somewhat different. The input diffusion is done differently. Rather than having four cascaded input diffusors, there are two pairs of cascaded diffusors, each feeding one leg of the tank. Both are fed from the predelay line. The output tap summation uses a lot more taps, including some in the predelay. 


  <script>
  MathJax = {
    tex: {inlineMath: [['$', '$']]}
  };
  </script>
  <script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-chtml.js"></script>