Signals everywhere
==========================================

The signal type `Sig` is the key aspect of the library!
So let's make good friends with it!

We will learn how to transform signals in this chapter
and find out some interesting peculiarities of it.

Control-rate vs audio-rate signals
-------------------------------------------------

In CE we have a type `Sig` to represent both audio and control signals.
For ease of use some context is hidden from the user. But sometimes we need
to distinguish them.

Audio signals work on audio rate (typical values are 44.1 KHz or 48 KHz) while control signals
are updated on much lower rate (like 1/64 or 1/128 fraction of audio rate).
Using control signals can save a lot of CPU.

For instance we can use them with LFOs or envelope generators or
to modify with time any sort of parameter of synthesizer.

If you want to know the gory details of it.
Under the hood the audio- and control-rate signals are represented with different data structures.
Audio-rate signal is array of doubles and control rate is just a single double value.
But for the ease of use they are represented with the same type in CE.

The smart engine makes coversions behind the scenes. But if we want we can
give it a hint:

~~~haskell
kr :: Sig -> Sig   -- enforces control-rate
ar :: Sig -> Sig   -- enforces audio-rate
ir :: Sig -> D     -- takes a snapshot of the signal (init-time rate or constant)
~~~

### Mutable values with control signals

By default `newRef` or `newGlobalRef` create placeholders for audio-rate signals.
But if we want them to hold control-rate signals we have to use special variants:

~~~{.haskell}
newCtrlRef          :: Tuple a => a -> SE (Ref a)
newGlobalCtrlRef    :: Tuple a => a -> SE (Ref a)
~~~

If signals are created with them they are control-rate signals.


The Signal space (SigSpace)
---------------------------------------

Often, we want to apply functions on the signals that are
contained within a more complex structure. Example of
structures that occur in practice are tuples of signals, but
also the side effect monad `SE`, which we have already seen,
and signals associated with UI-widgets such as knobs and
sliders, that we will see later-on.

We provide an abstraction over such "signal containing"
types, which we call `SigSpace`. It's a type-class with a
method for applying a signal function to every signal
contained within the structure.

~~~{.haskell}
class Num a => SigSpace a where
  mapSig :: (Sig -> Sig) -> a -> a
~~~

We also defined a synonym `at` for `mapSig`.
Of course, `Sig` itself is an instance of `SigSpace` and defines `mapSig` as the identity.

The following example illustrates how to filter noise. The `linseg`
creates a straight line between the points `1500` and `250` that lasts for `5` seconds:

~~~haskell
> dac $ at (mlp (linseg [1500, 5, 250]) 0.1) $ white
~~~

The `SigSpace` type-class lets us apply signal transformations to values of
many different types. We've already seen such a function, `mul`:

~~~{.haskell}
mul :: SigSpace a => Sig -> a -> a
~~~

`mul` scales the amplitude of all the signals contained in a `SigSpace`.

Let us review other useful functions working with `SigSpace`.

~~~{.haskell}
cfd :: SigSpace a => Sig -> a -> a -> a
~~~

`cfd` does a crossfade between two signals. The first argument is a signal that should
vary between 0 to 1. `cfd` then interpolates between second
and third arguments according to the value of the first argument.

The same idea is generalised to interpolate between four signals:

~~~haskell
cfd4 :: SigSpace a => Sig -> Sig -> a -> a -> a -> a -> a
cfd4 x y asig1 asig2 asig3 asig4
~~~

We can imagine that we place four signals on the corners of the unipolar square.
we can move within the square with `x` and `y` signals. The closer we get to the
corner the more prominent becomes the signal that sits in the corner and other
three become quiter. The corner to signal map is:

* `(0, 0)` is for `asig1`

* `(1, 0)` is for `asig2`

* `(1, 1)` is for `asig3`

* `(0, 1)` is for `asig4`


Signals can be mixed using `wsum`. It takes a list of pairs
of a weight and `SigSpace` value, and computes the weighted
sums of the `SigSpace` values.

~~~{.haskell}
wsum :: SigSpace a => [(Sig, a)] -> a
~~~

`SigSpace` is analogous to the `Functor` class, its method
`mapSig` being similar to `fmap`.

~~~haskell
 mapSig :: (Sig -> Sig) -> a -> a
~~~

Likewise we define a `BindSig` class analogous to `Monad`
with a `bindSig` method that allows us to map effectful
signal functions.

~~~haskell
class Num a => BindSig a where
  bindSig :: (Sig -> SE Sig) -> a -> SE a
~~~

`jitter` is an example of an effectful signal function:

~~~haskell
jitter :: Sig -> Sig -> Sig -> SE Sig
jitter kamp kcpsMin kcpsMax
~~~

It creates random line segments with amplitude between `(-kamp, +kamp)`
with frequency of change in the interval `(kcpsMin, kcpsMax)`.

It can be used to introduce random variation in a sound which
helps make the music sound less artificial.  Random number
generation is an effectful computation hence, the result of
`jitter` is wrapped in the `SE` monad.

Let us apply `jitter` to modulate the cutoff frequency of
a Butterworth low-pass filter `blp`.

~~~haskell
> env :: SE Sig
> env = on 50 1500 $ jitter 1 0.5 2

> filt :: Sig -> SE Sig
> filt x = at (\cps -> blp cps x) env
~~~

Using `bindSig` we apply the filter to a signal.

~~~haskell
> dac $ bindSig filt $ (saw 220 + sqr 110)
~~~

To apply our jittery filter some white noise, we use the operator `=<<`.

~~~haskell
> dac $ bindSig filt =<< white
~~~

The operator `=<<`, which comes from the `Monad` type class.
It's standard way to apply effectful transformations to effectful values in the Haskell.
It's simplified type is:

~~~haskell
(=<<) :: (a -> SE b) -> SE a -> SE b
~~~

We can write the whole book on explanation of the `Monad`. But right now
just remember the signature. It allows us to plug effectful values
to effectful computations. It's just like `fmap` but with effectful input.

Generic At-class
------------------------------------

Wow so many conversions going on: `SigSpace`, `BindSig` or even combo of `Monad` with `BindSig`.
The head can go in rounds. This happens because Haskell is mmm.. well.. strongly
typed language and sometimes can be restrictive on types.

But it can be very awkward at times. imagine the expression:

~~~haskell
mapSig (f . g . h) expr
~~~

Here dot is Haskell way to compose functions. (we plug input of rightmost to the input of next and so on).
But now we change the `f` to `f'` which has side effects. And then we need to
change `mapSig` to `bindSig` and moreover if `expr` changes to effectful `expr'`
we need to add `=<<` in proper place.

~~~haskell
bindSig (e' . g . h) =<< expr'
~~~

This can be quite annoying when we want to quickly test some effects on input signals.
To solve this we can use the generic function `at`.
It calculates by the types of inputs the right conversion for it.
So it can be used instead of `mapSig`, `bindSig` and many others.
It's kind of swiss army knife to apply any signal processing function to anything.

The type is very generic. We use some clever hackery to make things right.

~~~haskell
at :: At a b c => (a -> b) -> c -> AtOut a b c
~~~

Just use it! It's convenient. Let's look at some examples:

In place of `mapSig`:

~~~haskell
dac $ at (mlp (linseg [1500, 5, 250]) 0.1) $ white
~~~

In place of `bindSig`:

~~~haskell
> env = on 50 1500 $ jitter 1 0.5 2
> filt x = at (\cps -> blp cps x) env

> dac $ at filt (saw 220 + sqr 110)
> dac $ at filt white
~~~

Notice no need for monadic operator when we switch from
pure waves to white noise!

The signal outputs (Sigs)
-------------------------------------

It's a tuple of signals. It's for mono, stereo and other sound-outputs.

~~~{.haskell}
class Tuple a => Sigs a
~~~

SigSpace for stereo transformations
----------------------------------

There are special variants of `SigSpace` and `BindSpace`
which are useful for stereo-effects. They process inputs with stereo transformations:

~~~haskell
class SigSpace2 a where
  mapSig2 :: (Sig2 -> Sig2) -> a -> a
~~~

and also

~~~haskell
class SigSpace2 a => BindSig2 a where
  bindSig2 :: (Sig2 -> SE Sig2) -> a -> SE a
~~~

And of course `at` unifies them all!.

----------------------------------------------------

* <= [Basic types](https://github.com/anton-k/csound-expression/blob/master/tutorial/chapters/BasicTypesTutorial.md)

* => [Rendering Csound files](https://github.com/anton-k/csound-expression/blob/master/tutorial/chapters/ProducingTheOutputTutorial.md)

* [Home](https://github.com/anton-k/csound-expression/blob/master/tutorial/Index.md)
