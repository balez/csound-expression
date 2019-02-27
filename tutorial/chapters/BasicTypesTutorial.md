Basic types
=====================================

Let's look at the basic types of the library.

Signals (Sig)
----------------------

We are going to make an audio signal. So the most frequently used type is a signal. It's called `Sig`.
The signal is a stream of numbers that is updated at a certain rate.
Actually it's a stream of small arrays of doubles. For every cycle the audio-engine
updates it. It can see only one frame at the given time.

Conceptually we can think that signal is a list of numbers.
A signal is an instance of type class `Num`, `Fractional` and `Floating`.
So we can treat signals like numbers. We can create them with numeric
constants, add them, multiply, subtract, divide, process with
trigonometric functions.

We assume that we are in ghci session and the module `Csound.Base` is loaded.

~~~
$ ghci
> import Csound.Base
~~~

So let's create a couple of signals:

~~~{.haskell}
> x = 1 :: Sig
> y = 2 :: Sig
> z = (x + y) * 0.5
~~~

In the older versions of ghci we need to write `let`  in
the interpreter to delcalre a variable or function in modern
versions it can be omitted. So if it's not working for you, just
include `let` keyword like this:

~~~{.haskell}
> let x = 1 :: Sig
~~~

Constants are pretty good but not that interesting as sounds.
A sound is a time varying signal. It should vary between -1 and 1.
It's assumed that 1 is the maximum amplitude. Everything beyond the 1
is clipped.

Let's study the [simple waveforms](http://public.wsu.edu/~jkrug/MUS364/audio/Waveforms.htm):

~~~{.haskell}
osc, saw, tri, sqr :: Sig -> Sig
~~~

They produce sine, sawtooth, triangle and square waves.
The output is band limited (no aliasing beyond [Nyquist](http://en.wikipedia.org/wiki/Nyquist_frequency)).
The waveform function takes in a frequency (and it's also a signal) and produces
a signal that contains a wave of a certain shape that is repeated at a given frequency (in Hz).

Let's hear the sound of the triangle wave at the rate of 220 Hz:

~~~{.haskell}
> dac $ tri 220
~~~

We can press `Ctrl+C` to stop the sound from playing. If we know the
time in advance we can set it with the function `setDur`:

~~~{.haskell}
> dac $ setDur 2 $ tri 220
~~~

Right now the sound plays only for 2 seconds. The `setDur` function
should be used only once. Right before the sending output to the `dac`.
Please don't use it many times for example to set the duration of notes,
it's only for setting the total duration of the whole track.

We can vary the frequency with slowly moving oscillator:

~~~{.haskell}
> dac $ tri (220 + 100 * osc 0.5)
~~~

If we use the `saw` in place of `tri` we can get a harsher
siren-like sound.

We can adjust the volume of the sound by multiplying it:

~~~{.haskell}
> dac $ mul 0.5 $ saw (220 + 100 * osc 0.5)
~~~

Here we used the special function `mul`. We could
just use the normal Haskell operator `*`. But `mul` is more
convenient. It can work not only for signals but for
tuples of signals (if we want a stereo playback)
or signals that contain side effects (wrapped in the monad `SE`).
So the `mul` is preferable.

Constant numbers (D)
----------------------------------------------

Let's study two another useful functions:

~~~{.haskell}
leg, xeg :: D -> D -> D -> D -> Sig
~~~

They are Linear and eXponential Envelope Generators.
They create [ADSR-envelopes](http://en.wikipedia.org/wiki/Synthesizer#ADSR_envelope).

ADSR envelope are signals built in four stages: the attack
stage where the signal value increases from 0 to 1, followed
by the decay stage where the signal value decreases from 1 to
the sustain level (a value between 0 and 1), the sustain
stage where the signal is constant at the sustain level, and
the release stage where the signal decreases from the sustain
level to zero.

They take in four arguments: the attack time, the
decay time, the sustain level and the release time.

The new type `D` in the signature is for constant doubles.
Think of it as a normal value of type `Double`. It's a `Double` that is
embedded in the Csound. In the implementation we don't calculate
these doubles but use them to generate the Csound code.

Let's create a signal whose pitch is gradually modified by a linear envelope.

~~~{.haskell}
> dac $ saw (50 + 150 * leg 2 2 0.5 1)
~~~

Notice that the signal doesn't reach the release phase. It's not a mistake!
The release happens when we release a key on the midi keyboard.
We don't use any midi here so the release never happens.

But we can try the virtual midi device:

~~~{.haskell}
> vdac $ midi $ onMsg $ \x -> saw (x + 150 * leg 2 2 0.5 1)
~~~

Right now don't bother about the functions `midi` and `onMsg`.
We are going to take a closer look at them in the chapter *User interaction*.
That's how we plug in the midi-devices.

The value of type `D` is just like a Haskell's `Double`. We can do all the
Double's operations on it. It's useful to know how to convert doubles to `D`'s
and how to convert `D`'s to signals:

~~~{.haskell}
double :: Double -> D
int    :: Int    -> D
sig    :: D      -> Sig
~~~

There are more generic functions:

~~~{.haskell}
linseg, expseg :: [D] -> Sig
~~~

They can construct the piecewise linear or exponential functions.

~~~{.haskell}
linseg [a, timeAB, b, timeBC, c, timeCD, d, ...]
~~~

The arguments are alternating values and time stamps that
describe how long it takes for the signal to move from one value to the next.
Values for `expseg` should be strictly positive.

There are two more generic functions useful for midi-instruments which add a release stage:

~~~{.haskell}
linsegr, expsegr :: [D] -> D -> D -> Sig
~~~

The last two arguments are the release time and the final value for release stage.

Another frequently used functions are

~~~{.haskell}
fadeIn  :: D -> Sig
fadeOut :: D -> Sig

fades   :: D -> D -> Sig
fades fadeInTime fadeOutTime = ...
~~~

They produce simpler envelopes. The `fadeIn` rises
in the given amount of seconds form 0 to 1. The `fadeOut`
does the opposite. It's 1 from the start and then it
fades out to zero in the given amount of seconds but only
after release. The `fades` combines both functions.


Strings (Str)
-----------------------------------

A friend of mine has made a wonderful track in Ableton.
I have a wav-file from her and want to beep-along with it.
I can use a `diskin2` opcode for it:

~~~{.haskell}
diskin2 :: Tuple a => Str -> a
diskin2 fileName = ...
~~~

It takes in a name of the file and
produces a tuple of signals. We should specify how many outputs
are in the record by precising the type of the tuple.
There are handy helpers for this:

~~~{.haskell}
ar1 :: Sig -> Sig
ar2 :: (Sig, Sig) -> (Sig, Sig)
ar3 :: (Sig, Sig, Sig) -> (Sig, Sig, Sig)
ar4 :: (Sig, Sig, Sig, Sig) -> (Sig, Sig, Sig, Sig)
~~~

Every function is an identity. It's here only to help the type inference.
Let's say we have a file `Noise.wav`. With a mono wav-file we can use:

~~~{.haskell}
> sample = ar1 $ diskin2 (text "Noise.wav")
~~~

The first argument of the `diskin2` is not a Haskell's `String`.
It's a Csound's string so it has a special type `Str`. It's just
like `D`'s  for `Double`'s. We used a converter function to
lift the Haskell string to Csound one:

~~~{.haskell}
text :: String -> Str
~~~
The function `text` converts Haskell strings to Csound strings.
The `Str` is an instance of `IsString` so if we are using
the extension `OverloadedStrings` we don't need to call the function `text`.

For a stereo wav-file "Composite.wav" we use:

~~~{.haskell}
> :set -XOverloadedStrings
> sample = toMono $ ar2 $ diskin2 "Composite.wav"
~~~

We don't care right now about the stereo so we have converted
everything to mono with function.

~~~{.haskell}
toMono :: (Sig, Sig) -> Sig
~~~

Ok, we are ready to play along with it:

~~~{.haskell}
> sample = toMono $ ar2 $ diskin2 (text "Composite.wav")
> meOnKeys = midi $ onMsg osc
> vdac $ mul 0.5 $ meOnKeys + pure sample
~~~

Notice how simple is the combining midi-devices output
with the regular signals. The function `midi` produces
a normal signal wrapped in `SE`-monad. We can use it anywhere.
We use standard function `pure` to wrap ordinary value to `SE`-type.
`meOnKeys` and `pure sample` have the same type and we can sum them up.

There are useful shortcuts that let us use a normal Haskell strings:

~~~{.haskell}
readSnd :: String -> (Sig, Sig)
loopSnd :: String -> (Sig, Sig)
loopSndBy :: D -> String -> (Sig, Sig)
readWav :: Sig -> String -> (Sig, Sig)
loopWav :: Sig -> String -> (Sig, Sig)
~~~

The functions with `read` play the sound files only once.
The functions with `loop` repeat over the sample over and over.
With `loopSndBy` we can specify the time length of the loop period.
The `readWav` and `loopWav` can read the file at a given speed.
A value of 1 corresponds to the normal speed. A value of -1 would mean playing in reverse.
Negative speed works only for `loopWav`.

So we can read our friend's record like this:

~~~{.haskell}
sample = loopSnd "Composite.wav"
~~~

If we want only a portion of the sound to be played we can use the
function:

~~~{.haskell}
takeSnd :: Sigs a => Sig -> a -> a
~~~

It truncates the input signal after a given time (in seconds)
and then follows with silence.

The first argument is a signal, but often it's set with a constant
or it's a constant value taken from a signal on the moment of invocation.
The second type is a bit unusual: `Sigs a => a`.
It means anything which is signal-like.
Instances of `Sigs a` not only include signals but all sorts of tuples of signals.
It enables us to use the same function with mono and stereo or
Dolby-surround audio signals.

It's interesting that we can loop not
only with samples but with regular signals too:

~~~{.haskell}
repeatSnd :: Sigs a => Sig -> a -> a
~~~

The first argument is the duration (in seconds) of the signal that we wish to loop over.
For instance:

~~~{.haskell}
> dac $ repeatSnd 3 $ leg 1 2 0 0 * osc 220
~~~


Tables (Tab)
-------------------------------------

We have studied the four main waveform functions: `osc`, `tri`, `saw`, `sqr`.
But what if we want to create our own waveform. How can we do it?

What if we want not a pure sine but two more partials. We want
a sum of sine partials and a first harmonic with the amplitude of 1
the second is with 0.5 and the third is with 0.125.

We can do it with `osc`:

~~~{.haskell}
> wave x = mul (1/3) $ osc x + 0.5 * osc (2 * x) + 0.125 * osc (3 * x)
> vdac $ midi $ onMsg $ mul (fades 0.1 0.5) . wave
~~~

But there is a more efficient way of doing it. Actually the oscillator reads
a table with a fixed waveform. It reads it with a given frequency and
we can hear it as a pitch. Right now our function contains three `osc`.
Each of them reads the same table. But the speed of reading is different.
It would be much better if we could write the static waveform with
three harmonics in it and read it with one oscillator. It would be much
more efficient. Think about waveforms with more partials.

We can achieve this with function:

~~~{.haskell}
oscBy :: Tab -> Sig -> Sig
~~~

It creates an oscillator with a custom waveform. The static waveform is encoded
as a value of type `Tab`, a one-dimensional table of doubles.
In Csound, such tables are called functional tables.
Generally functional tables are initialised using GEN-routines
which are high level procedures that compute the content of a
table.

Right now we want to create a sum of partials or harmonic series.
We can use the function sines:

~~~{.haskell}
sines :: [Double] -> Tab
~~~

Let's rewrite the example:

~~~{.haskell}
> wave x = oscBy (sines [1, 0.5, 0.125]) x
> vdac $ midi $ onMsg $ mul (fades 0.1 0.5) . wave
~~~

You can appreciate the simplicity of these expressions
compared to their implementation in Csound.

What if we want not 1, 2, and third partials but 1, 3, 7 and 11?
We can use the function:

~~~{.haskell}
sines2 :: [(PartialNumber, PartialStrength)] -> Tab
~~~

It works like this:

~~~{.haskell}
> wave x = oscBy (sines2 [(1, 1), (3, 0.5), (7, 0.125), (11, 0.1)]) x
~~~

### The table size

The table size was implicit in the previous examples.  The
default size 8196, but we may create tables of any size. The
bigger the table, the better is precision.

We can set the size explicitly with:

~~~{.haskell}
setSize :: Int -> Tab -> Tab
~~~

For efficiency reason, the size should be a power of 2. The
following functions are used to set the size of a given
table. Consult the section on GEN-routines from the [Csound reference manual](http://csound.com/docs/manual/index.html).

Alternatively, the following functions offer seven standard
sizes from low to high fidelity.

~~~{.haskell}
lllofi, llofi, lofi, midfi, hifi, hhifi, hhhifi :: Tab -> Tab
~~~

`lllofi` is the lowest fidelity and `hhhifi` is the highest fidelity.


### The guard point

If you are not familiar with Csound's conventions you are probably
not aware of the fact that for efficiency reasons Csound requires
that table size is equal to power of 2 or power of two plus one
which stands for guard point (you do need guard point if your intention
is to read the table once but you don't need the guard point if you
read the table in many cycles, then the guard point is the the first point of your table).

If we read the table once we have to set the guard point with function:

~~~{.haskell}
guardPoint :: Tab -> Tab
~~~

There is a short-cut called just `gp`. We should use it with `exps` or `lins`.

### Specific tables

There are a lot of GEN-routines [available](http://hackage.haskell.org/package/csound-expression/docs/Csound-Tab.html).
Let's briefly discuss the most useful ones.

We can write the specific numbers in the table if we want:

~~~{.haskell}
doubles :: [Double] -> Tab
~~~

Linear and exponential segments:

~~~{.haskell}
consts, lins, exps, cubes, splines :: [Double] -> Tab
~~~

Reads samples from files (the second argument is duration of an audio segment in seconds)

~~~{.haskell}
data WavChn = WavLeft | WavRight | WavAll
data Mp3Chn = Mp3Mono | Mp3Stereo | Mp3Left | Mp3Right | Mp3All

wavs :: String -> Double -> WavChn -> Tab
mp3s :: String -> Double -> Mp3Chn
~~~

Harmonic series:

~~~{.haskell}
type PartialStrength = Double
type PartialNumber   = Double
type PartialPhase    = Double
type PartialDC       = Double

sines  :: [PartialStrength] -> Tab
sines2 :: [(PartialNumber, PartialStrength)] -> Tab
sines3 :: [(PartialNumber, PartialStrength, PartialPhase)] -> Tab
sines4 :: [(PartialNumber, PartialStrength, PartialPhase, PartialDC)] -> Tab
~~~

Special cases for harmonic series:

~~~{.haskell}
sine, cosine, sigmoid :: Tab
~~~

There are other tables. We can find the complete list in the module [Csound.Tab](http://hackage.haskell.org/package/csound-expression/docs/Csound-Tab.html).
In Csound the tables are created by specific integer identifiers but in CE they are defined with names (hopefully self-descriptive). If you are used
to integer identifiers you can check out the names in the Appendix to the documentation of
the [Csound.Tab](http://hackage.haskell.org/package/csound-expression/docs/Csound-Tab.html) module.

Side effects (SE)
------------------------------------------------

The `SE`-type is for functions that work with side effects.
They can produce effectful value or can be used just for the
side effect.

For example every function that generates random numbers
uses the type `SE`.

To get the white, pink or red (Brownian) noise we can use:

~~~{.haskell}
white :: SE Sig
pink  :: SE Sig
brown :: SE Sig
~~~

Let's listen to the white noise:

~~~{.haskell}
> dac $ mul 0.5 $ white
~~~

We can get the random numbers with linear interpolation.
The output values lie in the range of -1 to 1:

~~~{.haskell}
rndi :: Sig -> SE Sig
~~~

The first argument is frequency of generated random numbers.
We can get the constant random numbers (it's like sample and hold
function with random numbers):

~~~{.haskell}
rndh :: Sig -> SE Sig
~~~

We can use the random number generators as LFO.
The `SE` is a `Functor`, `Applicative` and `Monad`.
Those are standard ways to manipulate values that are wrapped
in some sort of behaviour or are containers.
We rely on these properties to get the output.

With `Functor` we can map over wrapped value:

~~~{.haskell}
> instr lfo = 0.5 * saw (440 + lfo)
> dac $ mul 0.5 $ fmap instr (20 * rndi 5)
~~~

We use function `fmap`:

~~~{.haskell}
fmap :: Functor f => (a -> b) -> f a -> f b
~~~

There are unipolar variants: `urndh` and `urndi`.
The output ranges form 0 to 1 for them.

Note that the function `dac` is overloaded and can work not only on signals but
also on the signals that are wrapped in the type `SE`
and also on all sorts signal-like or renderable values.

Let's take a break and listen to the filtered pink noise:

~~~haskell
> dac $ mul 0.5 $ fmap (mlp (on 50 2500 $ tri 0.2) 0.3) $ pink
~~~

The function `on` is useful for mapping the range (-1, 1) to
a different interval. In the expression `on 50 2500 $ tri 0.2`
oscillation happens in the range `(50, 2500)`. There is another
useful function `uon`. It's like `on` but it maps from the range `(0, 1)`.

The essence of the `SE Sig` type lies in the usage of random values.
If `rndh` was a pure signal of type `Sig`, we wouldn't be able to distinguish between these two expressions:

~~~haskell
x1 =
  let a = rndh 1
  in a + a

x2 = rndh 1 + rndh 1
~~~

For `x1` we want only one random value but
for `x2` we want two random values.

The value is just a tiny piece of code (we don't evaluate expressions
but use them to generate Csound code).
The renderer performs common subexpression elimination.
So the examples above would be rendered in the same code.

In order to tell the renderer when we want two random values, we need to encapsulate side effects in a monad.
This is what the `SE` monad is for (it stands for Side Effects).

~~~haskell
x1 = do
  a <- rndh 1
  return $ a + a

x2 = do
  a1 <- rndh 1
  a2 <- rndh 1
  return $ a1 + a2
~~~

The SE monad was introduced to express the randomness.
But then it was useful to express many other things.
Procedures for instance. They don't produce signals
but do something useful:

~~~haskell
procedure :: SE ()
~~~

The `SE` is used for allocation of delay buffers in the following functions:

~~~haskell
deltap3 :: Sig -> SE Sig
delayr :: D -> SE Sig
delayw :: Sig -> SE ()
~~~

The `deltap3` is used to allocate the delay line. After allocation
we can read and write to delay lines with `delayr` and `delayw`.

The `SE` is used for allocation of local or global variables (see the type `SERef`
in the module `Csound.Control.SE`).

For convenience the `SE Sig` and `SE` of tuples of signals is instance of `Num`.
We can sum and multiply the signals wrapped in the `SE`. For example:

~~~haskell
> dac $ white + 0.5 * pink
> dac $ white + return (osc 440)
~~~


Mutable values
-------------------------------------------------

Mutable variables in csound-expression work just like normal
Haskell mutable variables. A mutable variable is a reference
to a memory cell which can be read and written to.

There are two types of mutable variables: local and global.
Local variables are visible only within one Csound instrument.
Global variables are visible everywhere.

Mutable variable are created with the following functions.

~~~{.haskell}
newRef          :: Tuple a => a -> SE (Ref a)
newGlobalRef    :: Tuple a => a -> SE (Ref a)
~~~

They take in an initial value and create a value of the type `Ref`
whose field are the write and read operations.

~~~{.haskell}
data Ref a = Ref
	{ writeRef :: a -> SE ()
	, readRef  :: SE a
	}
~~~


### Mutable values with control signals

By default `newRef` or `newGlobalRef` create placeholders for audio-rate signals.
But if we want them to hold control-rate signals we have to use special variants:

~~~{.haskell}
newCtrlRef          :: Tuple a => a -> SE (Ref a)
newGlobalCtrlRef    :: Tuple a => a -> SE (Ref a)
~~~

If signals are created with them they are control-rate signals.

Tuples (Tuple)
-------------------------------------------------

Some of the Csound functions are producing several outputs.
Then the output is represented with `Tuple`. It's a special
type class that contains all tuples of Csound values.

There is a special case. The type `Unit`. It's Csound's alias
for Haskell's `()`-type. It's here for implementation reasons.

We have already encountered the tuples when we have studied
the function `diskin2`.

~~~{.haskell}
diskin2 :: Tuple a => Str -> a
~~~

In Csound, functions can be overloaded. The concrete
definition is chosen according to the number and types of
arguments and results when using the function.

In order to do the same thing in Haskell, we use the type class `Tuples`.

However, the haskell compiler or interpreter may not be able
to infer the concrete tuple that we want. In order to make
the type unambiguous, we can provide a type
signature. Another possibility is to use the following identity functions:

~~~{.haskell}
ar1 :: Sig -> Sig
ar2 :: (Sig, Sig) -> (Sig, Sig)
ar3 :: (Sig, Sig, Sig) -> (Sig, Sig, Sig)
ar4 :: (Sig, Sig, Sig, Sig) -> (Sig, Sig, Sig, Sig)
~~~

For instance:
~~~haskell
> sample = ar2 $ diskin2 (text "fox.wav")
> sample
sample :: (Sig, Sig)
~~~

Alternatively, we can just specify the type of output.

In the interpreter:

~~~haskell
> sample = diskin2 (text "fox.wav") :: (Sig, Sig)
> :t sample
sample :: (Sig, Sig)
~~~

Or in the code:

~~~haskell
sample :: (Sig, Sig)
sample = diskin2 (text "fox.wav")
~~~

Many type synonyms are given for signal tuples. For instance:
~~~haskell
type Sig2 = (Sig, Sig)
type Sig3 = (Sig, Sig, Sig)
type Sig4 = (Sig, Sig, Sig, Sig)
~~~

----------------------------------------------------

* <= [Introduction](https://github.com/anton-k/csound-expression/blob/master/tutorial/chapters/Intro.md)

* => [Signals everywhere](https://github.com/anton-k/csound-expression/blob/master/tutorial/chapters/SignalTfm.md)

* [Home](https://github.com/anton-k/csound-expression/blob/master/tutorial/Index.md)
