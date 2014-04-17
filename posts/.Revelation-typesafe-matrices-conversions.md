---
title: Typesafe Matrices for OpenCV
description: Post in progress on implementing type safe matrices and the API planned for OpenCV
mathjax: on
---

Major problem when using OpenCV C++ API --
    Matrices are shaped containers of values
    Shape and type of values are untyped
    only known at runtime -- dynamically typed

Example: Colorspace conversions
    cvtColor just takes a numeric flag
    decides from this flag what conversion to run
    thought you were converting an HSV matrix to RGB when you had an RGB matrix all along? too bad.

It is a mistake to carry this approach over into the Haskell side

Instead we can expose the types of our matrices
    Row/Column size
    Channels/colorspace
    element type

Use DataKinds to restrict which types make valid Mats
    This would be problematic for code that wishes to have different behavior for differently typed matrices
    except that the matrices themselves *are* the runtime type witness that allows you to correctly dispatch on the channel or dimension type

The element type is not restricted in the same way, but if you intend to put in into a matrix or get it back out, it needs to be (Storable e, Num e)
    We technically don't need to restrict ourselves to Numeric types, but opencv's functions will probably break if you pass in a matrix of matrices or something.

Functions that don't affect the type of the matrix and don't care about the type coming in are trivial.

Functions that only work on one or a few types of matrices are no longer partial.

Functions that work on all matrices but need to dispatch on the type can do so via typeclasses
    The typeclassed functions will need to take a runtime witness for the lifted types (the matrice counts)

Example: Colorspace conversions in Revelation
