[color=#007FFF][b][size=5]3.1 Encode :: Encoding[/size][/b][/color]
[color=#007FFF][b][size=3]3.1.0 Remuxing[/size][/b][/color]

Obviously, if your end goal is a remux, you should skip this step.

[color=#007FFF][b][size=3]3.1.1 Preparing your video source[/size][/b][/color]

First things first, we need to use [b]ffmsindex[/b] to index our video. (You can index it from AvsPmod, but doing it from the command line will give you a progress meter; AvsPmod will also be usable during this indexing period.) [quote][pre]ffmsindex source.mkv[/pre][/quote] If you are not using a remux as your source, you will have to mux the video track to mkv first, jump to the next section really quick to see how to mux to mkv if needed (launch the GUI, drag and drop the video track in, set the FPS - check eac3to log for this, probably 24000/1001p, and mux).

Once you have indexed your video, open AvsPmod. The source filter for ffms2 is [b]ffvideosource(source)[/b]. Your script might look like [quote][pre]ffvideosource("/home/db/encoding/source.mkv")[/pre][/quote] Next step is to crop any black bars if present. You should skip around the video to check that the black bars are consistent, quite a few movies have variable aspect ratios. Once you're ready to crop, go to [b]Tools > Crop Editor[/b]. Adjust the values for cropping until all black bars have been removed. AvsPmod will enforce mod2 cropping for you. There is also an auto crop feature; I recommend doing this manually. If there are black bars in both directions, [i]go ahead and crop them all off.[/i] If you have a single black pixel left do not over crop it until you've read Appendix C and understand if it can be fixed or not.

Now you will read Appendix C and determine if your source needs filtering.

[color=#007FFF][b][size=3]3.1.2 Test Script + Av*Synth[/size][/b][/color]

We will now resize the video, [b]if we are encoding 720p[/b]. If you cropped in two directions, [b]do not resize to fill out 1080p![/b] Go to [b]Tools > Resize Calculator...[/b]. Go to [b]Configure[/b], set [b]Resize block constraints[/b] to [b]2x2[/b] and [b]Avisynth resize[/b] to [b]Spline36Resizemod(%width%, %height%)[/b]. Adjust the slider until either the horizontal resolution is 1280 or the vertical resolution is 720, the final picture should fit inside a 1280x720 pixel box, with one or both sides touching the box. Click [b]apply[/b] and AvsPmod should append another line to your script which will be your resize function. There are multiple resize filters that you can choose from, read [url=http://avisynth.nl/index.php/Resize]this[/url] for more info. Spline36 is used here since it is a neutral filter. I HIGHLY recommend using [url=https://pastebin.com/NapEHdh8]THIS[/url] modified version of Spline36Resize called Spline36ResizeMod which accounts for a slight chroma shift bug in the default Spline36.

Also note you can removed a duplicated line when scaling
[quote=pafnucy]If you duplicated a line at a border using FillMargins, you can omit it during scaling. For example, to skip one line on left and one on right edge use the following:
Spline36ResizeMod(1278, 720,1,0,-1,0)
You can't cheat and directly remove a single black line this way, as it would still be sampled during resizing and 'dirty up' the adjacent lines. That is why in the filtering stage we used FillMargins on it first.[/quote]

To run test encodes, we will use a built in function to grab y frames from every x frames starting with frame z. [quote][pre]SelectRangeEvery(x,y,z)[/pre][/quote] is what we're looking for. Typically, the values for (x, y, z) are [b](2000, 50, 10000)[/b] which will take 50 frames, every 2000 frames, starting at frame 10000. You are welcome to change these values to what ever you feel is best. [b]For your final encode, remove this line.[/b]

Go to [b]File > Save Script[/b] or [b]Ctrl+S[/b] to save your new Av*synth script. The extension is largely irrelevant (I can't remember if Simple x264 Launcher imposes any such limitation), but might as well save it with [b].avs[/b] for the extension if AvsPmod doesn't do that automatically.

[color=#007FFF][b][size=3]3.1.3 Limitations on Ubuntu and OS X (advanced)[/size][/b][/color]

Windows users have a variety of external filters that they can choose from, which have not all been ported to AvxSynth. If you feel you need one of these filters, use a Windows VM to create your script, and use avs2yuv to save the result to a y4m file, which you can open directly with x264. Alternatively (and this is a better idea), pipe the output to x264 and encode a lossless file, which will save a great deal of space (up to 5.5x in my limited testing).

To encode to lossless x264: [quote][pre]avs2yuv in.avs -o - | x264 --qp 0 -o out.mkv --demuxer y4m -[/pre][/quote] You can then treat this file like any other mkv when encoding under your native OS.

[color=#007FFF][b][size=3]3.1.4 Settings[/size][/b][/color]

I'm going to classify x264 settings into three categories: [b]always, tweak, forget[/b]. The [b]always[/b] category consists of settings that you will [i]always[/i] set to a certain value, for either quality or DXVA reasons. The [b]tweak[/b] category contains the settings that you will be [i]tweaking[/i] to get the most out of x264 and your encode. The [b]forget[/b] category is made of settings that you can [i]forget[/i] about because they are either largely irrelevant to typical users or their defaults are desired. I will go over the [b]always[/b] and [b]tweak[/b] categories.

For those of you interested in the why of these settings there is a good outline hidden below that explains the justification for the most important settings. This outline could also be helpful for you to understand how encoding with x264 works and what you're actually doing when you changes these values.

[b]x264 and You: An empiricist's insight into encoding theory, and settings (*by Mael at TC*)[/b][spoiler]
Motion Picture, Frames and Frame-types: [spoiler]

A movie is a contraction for 'motion picture'. A movie consists of a certain number of pictures (frames) that are shown to you every second to create the perception of motion. Note that movie is not an arbitrary collection of images. Most of the images in a motion picture are dependent on the preceding and succeeding images, so for storage purposes, each image need not be stored in entirety.

So what we have are three types of frames/pictures:

I-frame or Intracoded frame -> Simply put - this frame is an independent frame, that does not use any information from frames that precede or succeed it. A rough analogy would mean this is our reference frame. They consume the most space.

P-frame or Predicted frame -> These types of frames use information from the previous frame. In other words p-frames basically encode how the current image is different from the preceding image. For this reason [presumably] they are also called as Delta-frames. Delta in calculus indicates change. They consume less space than I, but more than B.

B-frames or Bi-predictive frames -> These frames are the most interesting ones.They experience most compression and have the lowest space allocation.

I will do some illustrating here. Imagine a sequence of four images where Jerry appears in the screen from nowhere/another dimension over a course of four frames – and a possible compression strategy.

In the first image there is a static background alone;

In the second image there is a transparent jerry starts manifesting from nowhere, Jerry appears but with 99% transparency [almost see-through]. This could be a p-frame.

In the third image, Jerry is about 50% transparent, he is almost there but not yet. This frame borrows information from the next-frame as well as previous frame. Jerry is almost there but not yet.

In the final image, Jerry is 100% physically present. In a particular encoding strategy, this could be an I-frame.

Note that – this is a very rudimentary example only to illustrate things – what gets compressed as what frame, depends on compression and encoding strategy.
[/spoiler]
Source, Encode, Transparency, and Bloating: [spoiler]
A source can be defined as a set of images/frames – B, P, and I.

An encode can be defined as a set of images/frames – B, P, and I that are supposed to represent the source. Encodes are invariably smaller than the sources.

A transparent encode is defined as an encode that represents the source accurately enough – so that the differences between the encode and the source are indiscernible. That is, if I play the source and encode alongside you shouldn’t be able to tell which is which. So how do we test for transparency?

Well, if you take a particular I-frames of the encode and compare them with the corresponding source frames – you won’t learn a thing. That is because, I-frames are treated with great sanctity; and thus the encoder tries replicates these frames as faithfully w.r.t. the source as possible even when you use a relatively bad setting.

A B-frame, on the other hand, takes advantage of compression techniques the most – so B-frames are the best indicators.

When a frame that was P-frame in source [P-frames are space-heavier than B-frames]; but is encoded as a B-frame in encode – such a frame best indicates how much work [or damage] your encoder has done.

When a frame that was P in source and B in encode resemble each other – that means that your encoder has created an encode that is very faithful to the source. You have achieved transparency.

If you could have achieved transparency at a significantly lower bit-rate, your encode is termed bloated.
[/spoiler]
--bframes: [spoiler]
--bframes: This setting defines how many consecutive bframes can occur in sequence. To illustrate, visualise a sequence where two samurais are facing each other for the right moment to strike. They face off each other for 10 seconds and nothing changes! [Referencing a Kurosawa movie - Sanjuro]. While you are anxiously waiting for the change, even your encoder is waiting for some sort of change to document. So suppose you use --bframes 3, here is what happens.

Frame 1: Reference, the good guy and bad guy are facing off – starting and each other.

Frame 2: Nothing has changed, dammit. Nothing has changed, except for some grains and so shall call it a P-frame? or a B-frame?

Frame 3: Nothing has changed, this frame is the same as the I-frame. So let me call this a P-frame.

Now here is the trick, if I was encoding with -–bframes 2; my encoder tells itself

Frame 2: Frame 3 and Frame 1 are identical, except for some grain-movement, so let me encode Frame 3 as P-frame dependent on frame 1; and Frame 2 a B-frame that depends on frame 1 and frame 3.



But note that nothing is changing in this action sequence for 240 frames [about 10 seconds]

Thus in reality we have a sequence that goes as follows-

Frame 4: Nothing has changed.

Frame 5: Nothing has changed.

.

.

.

Frame 240: Finally the bad guy decides to make his move.

Now suppose, I told my encoder, I want only --bframes 2 – here is what could have happened frame by frame.

I -> B -> P -> B -> P -> etc.

But if I told my encoder that I am okay with bframes 10 – here is what happens :
I -> B -> B -> B -> 16 times then -> P then 10 more b frames.

So, to put it in one-line - if a --b-frames 10, means that after 10 consecutive b-frames - a frame that could have been encoded as a b-frame, will still end up as a p-frame - simple because the limit of consecutive b-frames considered is 10. It does not mean that b-frames are used when p-frames are explicitly determined as more benefitial.

So what are the pros and cons? A high b-frames would imply that your encoder keeps a lot more frames in mind while deciding if a frame must be b or a p. So it slows down your encoding process.

So what we do is – we determine what percentage of frames will benefit if we use a higher bframe strategy. Say we do a test-encode with –-bframes 16; the log will tell us how many frames actually benefit from looking at 16 frames ahead and behind. Echohead says that if a particular number is benefiting in less than 1% of the cases, you should choose that number.

Whereas, other encoders might say that you should always encode with –- bframes 16 because space saved in the final encode is space saved.

Verdict: Leave it at 16.

[/spoiler]
--b-adapt: [spoiler]
B-adapt is a minor setting that is always left at 2. Basically B-adapt is a setting that tells your encoder whether you want to place a P-frame or a B-frame in particular cases. B-adapt 1 will make this decision more quickly, leading to poorer judgements and lowered quality. B-adapt 0 will always pick B-frames – leading to worsened encode [but lowered encode size]. B-adapt 2; which is what encodes who aim for transparency use asks the encoder to take its time and make the best decision.

Verdict: Leave it at 2.
[/spoiler]
--b-bias: [spoiler]
B-bias is another minor setting that I leave unchanged at 0. B-bias positive till 100 will increase the likelihood of encoding a particular frame as B. And negative will increase the likelihood of encoding a particular frame as P.

Verdict: Leave it at 0
[/spoiler]
--b-pyramid: [spoiler]
B-pyramid is another minor setting that can be none, strict or normal. Bframes that are used as reference for other bframes get assigned more bits than your average bframe but less bits than a pframe. Choosing 'None'disallows a bframe from using another bframe as reference. 'Strict' basically allows only one b-frame to be used as reference frame in a set [referred to as minigop]. I leave this setting at normal to allow many b-frames to be used as reference b-frames.

Verdict: Leave it at normal.
[/spoiler]
Frames and Macroblocks: [spoiler]
Let us take a step back and understand how images are stored and rendered. When you deal with a 640X480 image - what does that mean? It means you are dealing with 3,07,200 pixels. So if you were storing that image by using a strategy of assigning a colour to each pixel, and say end up assigning 24 bits to store what colour a pixel is – you would end up using about 900 KB per picture. This would be a very inefficient strategy, because each pixel is not discrete/arbitrary; but will have a correlation with the adjacent pixels. Thus, as far as I understand, information in an image is not stored per pixel; but per macroblock. Look at macroblocks as an chunks/aggregation of pixels. Macroblocks themselves can depend on other macroblocks within the frame [or as we will see later in the next paras even other frames]

Going back to our original P vs B vs I discussion; an I-frame has independent and intradependent macroblocks. That is, a macroblock in that frame could depend on other macroblocks in the same frame or be totally independent.

But in a P-frame, a macroblock could depend on the macroblock from past-frame; and in a B-frame some macroblocks on macroblocks of a frame that can either be succeeding or previous. 
[/spoiler]
mbtree and Qcomp; pb and ip ratios: [spoiler]
Disclaimer: Well, mbtree is where I am going to take a HUGE leap of faith and try to explain what I think is right. But please correct me if I am wrong. I think that my understanding does have great empirical validity and practical value – so I present it even with the risk of it being technically wrong. I encoded a small shot with four settings to explain myself better.

qcomp: 0; Mbtree: Off:[spoiler]
x264 [info]: frame I:4 Avg QP:14.13 size: 69217
x264 [info]: frame P:34 Avg QP:17.18 size: 18025
x264 [info]: frame B:146 Avg QP:19.53 size: 4792
[/spoiler]
qcomp: 1; Mbtree: Off:[spoiler]
x264 [info]: frame I:4 Avg QP:13.47 size: 73761
x264 [info]: frame P:34 Avg QP:14.00 size: 32968
x264 [info]: frame B:146 Avg QP:15.98 size: 12360
[/spoiler]
Exactly, what does this data mean?

As you can see, when Qcomp was increased to 1, P and B frames encoded more data and were more space-intensive; whereas when Qcomp was made to 0, P and B frames had less data.

Qcomp determines your encoding strategy. Given the fact that the default qcomp is 0.6, when I say "Qcomp = 0.8" I am telling my encoder this "Hey, just because a frame is P or B, doesn't mean that they are to be given too much of a step-motherly treatment". When I say "Qcomp = 0.4", P-frames and B-frames deteriorate much more than the I-frame.

Thus - in our example, when Qcomp was 0, the P and B frames were very information-light.

Let us take a break here and understand what does mbtree does. For that two more tests

Qcomp: 0; Mbtree: On (the strongest mbtree):[spoiler]
x264 [info]: frame I:4 Avg QP:10.51 size:113605
x264 [info]: frame P:34 Avg QP:18.05 size: 24044
x264 [info]: frame B:146 Avg QP:25.76 size: 1188
[/spoiler]
Qcomp: 1; Mbtree: On:[spoiler]
x264 [info]: frame I:4 Avg QP:13.47 size: 73761
x264 [info]: frame P:34 Avg QP:14.00 size: 32968
x264 [info]: frame B:146 Avg QP:15.98 size: 12360
[/spoiler]
The most obvious thing you will probably notice is that at Qcomp 1; mbtree on or off doesn't make any difference. And also, when we use Mbtree, the I-frames consumed much more space, but the P-frames somewhat less space; and the B-frames a drastic drop in space consumed.

So what exactly happened here?

Mbtree is the answer to the following question - a lot of macroblocks evolve from a macroblock in the previous frame and evolve into macroblocks in the succceeding frame. So are we to look at macroblocks as stemming out from previous macroblocks and leading to succeeding macroblocks?

When I leave mbtree on – I make the reference frame highly detailed and I try to derive macroblocks in p and b frames as trees that emerge from the I-frame. So what are the implications of what we have discussed?

1) When a movie is very grainy and noisy [for instance, some of the 80s movies that are optimised for CRT viewing - we will benefit from high Qcomp]. Why? Because - the movie is very noisy! Thus, even though a b-frame is similar to p and b frames with respect to the actual movie - it is dissimilar with reference to noise. So if your encoder disrespect the b and p frames too much, your encode is going to look like shit. Similarly, a darker movie might not need a high qcomp, since quite a few macroblocks are non-luminated. A typical bluray movie will need qcomp from 0.65 to 0.75. A darker movie might have lower qcomp; and a brighter-grainier will need higher qcomp.

2) When you raise qcomp, your encode's size invariable increases

3) From (1) and (2) so ideally your qcomp should be high enough so that your p/b-frames get enough love; but low enough so that they don't document minor changes that are invisible to the eye. Example of a low Qcomp-scenario is a cartoon.

4) mbtree evaluates macroblocks as trees - when you have a very high qcomp, you are demanding that your b-frames and p-frames be treated respectfully. Thus, when you turn mbtree on at qcomps higher than 0.75, the mbtree is so weak that it will only look at extremely similar macroblocks as trees. Thus you will end up saving some bits, but not losing transparency.

5) The lower the Q-comp the stronger the impact of the mbtree. But read the next para carefully!

Warning: If you turn mbtree on for encodes which use qcomps less than 0.7, you must be very very sure that the source isn't grainy or complex or else you will end up destroying your encode. Turn it off with --no-mbtree; since mbtree is enabled by default.

Finally, IP ratio and PB ratio. As far as I understand, these are again controls that determine how much information must go inside P w.r.t I and P w.r.t B etc.
[/spoiler]
--aq-strength: [spoiler]
To me - this is another highly interesting variable. As we have discussed before, P and B frames have macroblocks that depend on macroblocks for other frames. Now the following questions arise – typically, all macroblocks aren’t same, so do you want them to treat them all similarly? Typically, the answer to this question is no. Say you are encoding a live action movie – in which a tanned, beautiful lady with extremely sharp features appears – here is what your AQ will determine for that shot:

The macroblocks that encode the eye, the nose, the lips etc. are highly detailed and are very dynamic – we will look at these as macroblocks that store sharp features. A low AQ preserves these edges and sharp features. Whereas, her clothing that has furls and waves and textures – what is stored there is gradients and contours. A high AQ preserves the texture/gradient. So if I had a low AQ, I will see her eyes as sharper and crisper, but look at her clothing - the finer details, waviness, folds in her altogether black dress will be compromised.

But, the above understanding grossly oversimplifies things. In reality - the quality of source, the graininess of the source, the noisiness of the source changes this game completely. Further, let us not forget the complexity introduced by qcomp and psy-rdo. Let us consider some examples. But before you read anything, bear in mind that these are empirical guidelines but not thumb rules. These should be used for coarse and not fine-tuning.

1) Consider a cartoony source that appears grain/noise-free and where you have chosen a q-comp that is low too, and maybe even enabled mbtree. There are some crisp features and some featureless macroblocks. In this scenario, you have chosen to invest more space/energy into I-frames. Thus, for a given bit-rate, a lower AQ encode (0.4 to 0.6) may look more transparent/better than a higher AQ encode (0.9 to 1.2) – since you have focussed your bits into encoding sharp features and edges.

2) Consider a grainy, high-quality source where you are, say, looking at qcomp of 0.65 or 0.7. An AQ between 0.7 to 0.9 could be helpful. Many high quality bluray sources may come under this category. A higher AQ will make your encode look smudged etc. A very low AQ could lead to loss of detail in clothing/hair etc.

Personally, I don’t know where I have benefitted from very high AQ like say beyond 1.1 or 1.2. They say that noise-free DVD films can use these settings. But I have worked only with shitty DVDs or DVDs optimised for CRTs and I have always found that the facial features and edges lose their detail if I use higher AQ [more than 1.1]. For noisy DVDs I have encoded thus far, I have had more luck with raising the qcomp and keeping the AQ low. For 70s and 80s movies that have shitty DVD transfers, qcomp of 0.75 to 0.85 is what worked for me; and for those I had to use lower AQs.
[/spoiler]
--psy-rdo: [spoiler]
Now this is a setting where theory and simulation won’t help you, but actual testing and experimentation is the way to go. Still - I will explain what little I have gathered from reading around and some experimentation.

Consider a set of source frames being encoded; here – in encodes, a lot of finer details and grains can be omitted to create a theoretically more faithful representation of the source [this is called structural similarity]. However, the resultant encode might not appear as sharp or as detailed as the source itself – because the encode is a simplified version of the source. Thus, even though the encode is similar to the source - it will look poor.

Here is where Psy-rdo comes handy. Psy-rdo, based on my rudimentary understanding, encodes to create macro-blocks that are less accurate representations of the source, but have similar complexity as the source. Thus – creating a perception of greater quality.

If you overuse psy-rdo, you will have too many artifacts and the encode will look bad. If you under-use or avoid psy-rdo, the encode will appear poor for the bit-rate. The right psy-rdo can only be determined through testing.

For relatively grain free sources, given the fact that the source is “simple”; you should try out lower psy-rdos [less than 0.6]. For more complex sources, you should try out all psys from 0.9 to 1.3; higher than 1.3 means you are abusing psy and are creating too many artifacts. Even if you manage to get a few transparent screenshots – I would call it a happy coincidence, and will say that your encode will definitely look bad overall. [Did I not tell you that I am an opinionated person?]

I don’t look at psy-rdo in isolation; rather I test it in tandem with AQ-strength. The reason why we do this must be self-explanatory – if you have understood the theory so far :)
[/spoiler]
psy-trellis: [spoiler]
Basically, in some movies some parts of a frame are grainier than other parts. In these cases Psy-trellis can be tweaked to gain transparency at a lowered-bit-rate. It is something that you have to test out after determining the psy – if you see a non-homogeneous grain-distribution.
[/spoiler]
--deblock: [spoiler]
Finally, one more setting that I want to discuss for now is deblock. Deblock basically determines what gets blocked. A deblock of -3:-3 is the trend. But if the source is very noisy and blocky – shitty DVD transfers; or VHS to DVD transfers [look up Turkish Rambo] – you will experience success with deblock -4:-4 to -6:-6. 
[/spoiler]
Most important setting: [spoiler]
 The most important setting to me and to many other users is this. --elitism 0. ;) Basically, x264 developers have a right to be proud. The ones that have read their documentation and made information understandable for the rest of us also have reasons to be proud. The rest of us have either plagiarized information, or learned things by hit and trial. 
[/spoiler]
[/spoiler]

[color=#007FFF][b][size=3]3.1.4.1 (Always) Settings[/size][/b][/color]

[b]--level 4.1[/b] for DXVA.
[b]--b-adapt 2[/b] uses the best algorithm (that x264 has) to decide how B frames are placed.
[b]--min-keyint[/b] should typically be the frame rate of your video, e.g. if you were encoding 23.976fps content, then you use [i]24[/i]. This is setting the minimum distance between I-frames.
[b]--vbv-bufsize 78125 --vbv-maxrate 62500[/b] for DXVA (The old guide used lower values to account for the possibility of writing the encode to BD for playback, this is no longer a consideration as other settings break this compatibility. The new values are the max level 4.1 can do, if your device breaks because of this the encode is not at fault, your device doesn't meet DXVA spec).
[b]--rc-lookahead[/b] [i]250[/i] if using mbtree, [i]60[/i] or higher else. This sets how many frames ahead x264 can look, which is critical for mbtree. You need lots of memory for this. (persoanlly I just leave this at 250 now as the impact on memory usage is 2GBs or so)
[b]--me[/b] [i]umh[/i] is the lowest you should go, if your CPU is fast enough, you might endure the slowdown from [i]tesa[/i]. [i]esa[/i] takes as much time as [i]tesa[/i] without the benifit if you want to slow down your encode to try and catch more movement vectors just use [i]tesa[/i], although the increase is not necessarily always worth it. This isn't really a setting you need to test, but on tough sources, you might squeeze some more performance out of x264 if you use tesa.
[b]--direct auto[/b] will automatically choose the prediction mode (spatial/temporal)
[b]--subme 10[/b] or [i]11[/i] (persoanlly I just set this to 11 the differnce in encode speed is within 3-4%)
[b]--trellis 2[/b] 
[b]--no-dct-decimate[/b] (possible slight increase in quality)
[b]--no-fast-pskip[/b] (possible slight increase in quality)

[color=#007FFF][b][size=3]3.1.4.2 (Tweak) Settings[/size][/b][/color]

[b]--bitrate / --crf[/b] Bitrate is in Kbps (Kilobits per second) and CRF takes a float, where lower is better quality. This is the most important setting you have... bitstarve an encode and it is guaranteed to look like crap. Use too many bits and you've got bloat (and if people wanted to download massive files, they would get a remux). Of course, the bitrate need can vary drastically depending on the source.
[b]--deblock[/b] [i]-3:-3[/i] to [i]3:3[/i], with negative values for live action and positive (including 0) values, typically. See [url=http://forum.doom9.org/showthread.php?t=109747]this doom9 post[/url] for what this does. (Most everyone just uses -3,-3 at all times now I believe this is a setting which can be of some use on really clean animation)
[b]--qcomp[/b] [i]0.6[/i] (default) to [i]0.8[/i] might prove useful. Don't set this too high or too low, or the overall quality of your encode will suffer. This setting has a heavy effect on mbtree. A higher qcomp value will make mbtree weaker. --qcomp 0 will cause a constant bitrate, while --qcomp 1 will cause a constant quantizer.
[b]--aq-mode[/b] [i]1 to 3[/i]: 1 distributes bits on a per frame basis, 2 tends to allocate more bits to the foreground and can distibute bits in a small range of frames, 3 is a modified version of 2 that attempts to allocate more bits to dark parts of the frames. The only way to know what's best for sure it to test.
[b]--aq-strength[/b] [i]0.5[/i] to [i]1.3[/i] are worth trying. Higher values might help with blocking. Lower values will tend to allocate more bits to the foreground, similar to --aq-mode 2. This setting drastically affects the bitrate when encoding with CRF, so it might be worthwhile to test this setting with 2 pass if you intend on using CRF for the final encode.
[b]--merange[/b] [i]24[/i] (the lowest that should ever be used) to [i]64[/i], setting this too high can hurt (more than 128), 32 or 48 will be fine for most encodes. In general, 32-48 for 1080p and 32 for 720p (when using umh) for movies with lots of motion this can help (e.g. action movies). Talking heads can get away with low values like 24. To achieve AHD GOLD, you need min of 32 for umh, min of 24 for tesa. The impact on encode speed is noticable but not horrible. I prefer to use 48 for 1080p and 32 for 720p when using umh or 32 for 1080p and 32 for 720p when using tesa.
[b]--no-mbtree[/b] I highly recommend testing with both mbtree enabled and disabled, as generally it will result in two very different encodes. mbtree basically attempts to degrade the quality of blocks rather than frames, so that only the unimportant parts of a frame get less bits. To do this, it needs to know how often a block is referenced later, which is why --rc-lookahead should be set to 250. Useful for things with static backgrounds, like anime. Or for things where you've used a high qcomp (.75 or above) and mbtree will have a lowered impact.
[b]--psy-rd[/b] [i]0.8[/i] to [i]1.15[/i]:[i]0[/i] to [i]0.15[/i] The first number is psy-rd strength, second is psy-trellis strength. This tries to keep x264 from making things blurry and instead keep the complexity.
[b]--bframes[/b] [i]6[/i] to [i]16[/i], This is setting the maximum amount of consecutive P frames that can be replaced with B frames. Test with 16 for your first test run, and set according to the x264 log: [quote][pre]x264 [info]: consecutive B-frames:  1.0%  0.0%  0.0%  0.0% 14.9% 23.8% 13.9% 15.8%  8.9%  9.9%  0.0% 11.9%  0.0%  0.0%  0.0%  0.0%  0.0%[/pre][/quote] Start counting with the first percentage as 0 and choose the highest number with more than 1%, which is 11 in this example. (or just leave this at 16 as allowing more bframes will not harm your encode and will aid in compression, I haven't lowered bframes in months now)
[b]--ref[/b] set the number of previous frames each P frame can use as references. Calculate the number you can use or if you're using the provided CLI inputs at the end of this it will be calculated for you by x264. Always use the max that you can.

The max --ref value can be calculated as follows:

For --level 4.1, according to the H.264 standard, the max DPB (Decoded Picture Buffer) size is 12,288 kilobytes.

Since each frame is stored in YV12 format, or 1.5 bytes per pixel, a 1920x1088 frame is 1920 * 1088 * 1.5 = 3133440 bytes = 3060 kilobytes.
12,288 / 3060 kilobytes = 4.01568627, so you can use a maximum of 4 reference frames.

Remember, [b]round both dimensions[/b] up to a mod16 value when doing the math, even if you're not encoding mod16!

Let's do the math for 1920x800.

1920 * 800 * 1.5 = 2304000 bytes = 2250 kilobytes.  12,288 / 2250 kilobytes = 5.45777778, so you can use a maximum of 5 reference frames.[/quote] Note that these conversions use base 2, so Kilobyte == 1024 bytes. If you get the math wrong, that's okay too - x264 will display a warning if you use too many, so you'll know if you need to change it.

[b]--zones[/b] zones have become a more common practice as of late, typically they are used to increase bitrate where ever you deband to stop the scene from rebanding when encoded. They can also be used to intelligently increase or decrease the bitrate of select scenes that don't need as much bitrate as the default allocation gives them. Typically it is harder to encode dark scenes so someone might trim and an encode to just it's light scenes (which are easier to compress) and then give more bitrate to the dark. If you have questions about this please ask.

[b]How should I test settings? (*by Moshrom*)[/b][spoiler]
Look at the bitrate your CRF value gave you, and switch to two-pass with that number. Then test q-comp (0.60-0.80 by 0.05 increments), aq (test mode 1/mode 2/mode 3 and between 0.50 and 1 by 0.05 increments), psy-rd (0.90-1.15 by 0.05 increments), and finally psy-trellis (0.00-0.10).

This is a super thorough approach that should give you a good result. There may be cases where other values outside these ranges may be necessary, but these are usually more than sufficient. You can test at 0.1 increments and then further at 0.05 increments to save time.

Once you've found the best values for these settings, retest CRF with them (it will definitely have changed) until you hit your initial bitrate again.

In addition, test --no-mbtree and --no-dct-decimate. Testing deblock -3-2, -2-2, -2-1 can be useful for SD sources.
[/spoiler]

[color=#007FFF][b][size=3]3.1.5 CLI Usage (optional for GUI users)[/size][/b][/color]

For Windows users, [quote][pre]avs2yuv in.avs -o - | x264 [options] -o out.mkv --demuxer y4m -[/pre][/quote]For Ubuntu and OS X users, [quote][pre]x264 [options] -o out.mkv in.avs[/pre][/quote]Those comfortable with shell scripting will benefit greatly here, as you can use a script to store and test settings.

My encodes all start with this input. avs2yuv "I:\temp\testing.avs" -o - | x264 --level 4.1 --preset veryslow --min-keyint 24 --vbv-bufsize 78125 --vbv-maxrate 62500 --rc-lookahead 250 --me umh --merange 48 --direct auto --subme 11 --no-dct-decimate --no-fast-pskip --deblock -3:-3 --qcomp .6 --aq-mode 1 --aq-strength 1 --ipratio 1.4 --pbratio 1.3 --no-mbtree --psy-rd 1 --bframes 16 --crf 15 -o 1080e.mkv --demuxer y4m - 2>&1

[color=#007FFF][b][size=3]3.1.6 Comparisons[/size][/b][/color]

Index your test encode or final encode, depending on where you are, and create a script for it, like [quote][pre]ffvideosource("encode.mkv")[/pre][/quote] [b]For Avxsynth (Ubuntu and OS X) users[/b], add to the top of both your source and encode script the following function: http://pastebin.com/Tdh080rF

This is a modified version of FFInfo, that works for AvxSynth users, which disables cfr/vfr time output (breaks AvxSynth), and enables a bonus text input. Add to the bottom of your scripts: [quote][pre]myFFInfo("Source or Encode")[/pre][/quote]depending on which, of course. If you need multiple lines, your call will look like [quote][pre]myFFInfo("line1
line2")[/pre][/quote] [b]For Avisynth (Windows) users[/b], download [url=http://pastebin.com/0Gqx9Hpg]this[/url] as [b]myffinfo.avsi[/b] and save it in [b]AviSynth 2.5\plugins[/b]. This script will be thus be automatically loaded whenever you try to make a call to [b]myFFInfo()[/b]. This function works the same as that for Avxsynth, with the exeception that newline escapes, '\n', are translated into actual newlines - use escapes instead of literal newlines.

[*][b]When frame counts and resolution match in AvsPmod, using the scroll wheel on your mouse will flip between the two (or more) scripts and frames[/b], this will help you compare the two (or more) images in rapid succession. Seeking in one script will also seek in matching scripts. You can also use the number pad. [b]Right Click > Save image as...[/b] to save a screenshot. Be sure to use PNG since it is lossless, and we do not want to introduce anymore compression. The file name will default to the frame number. You might find it beneficial to append [i]_source[/i] or [i]_encode[/i] (or similar) to the end of the frame number and use that as the file name.

[*]Use the slider to jump around the video and look for a frame that will be a good indication of overall quality. Choose a variety of frames, ranging from close-ups of faces to crowds and objects to scenery. 

[color=#007FFF][b][size=3]3.1.6 Transparency / Compression Discussion[/size][/b][/color]

Transparency is very subjective, what one person may consider transparent may not be to another. The goal of encoding is to produce an output that is indistinguishable to the source to the human eye, while reducing the size. Both parts are equally important. You do not want to produce an encode that has lost a lot of detail or grain, but you also do not want to create a bloated encode just to make it look more transparent. This compression is the main reason for encodes. The quality of the screen you use will impact your viewing experience, a TN panel will not repoduce the colors in the full range or as accually as an IPS screen.

*Actually, there are several other reasons why encodes are created, but they are out of the scope of this guide.
