Rich error messages from GHC
============================

.. proposal-number:: Leave blank. This will be filled in when the proposal is
                     accepted.
.. ticket-url:: Leave blank. This will eventually be filled with the
                ticket URL which will track the progress of the
                implementation of the feature.
.. implemented:: Leave blank. This will be filled in with the first GHC version which
                 implements the described feature.
.. highlight:: haskell
.. header:: This proposal is `discussed at this pull request <https://github.com/ghc-proposals/ghc-proposals/pull/0>`_.
            **After creating the pull request, edit this file again, update the
            number in the link, and delete this bold sentence.**
.. sectnum::
.. contents::

Extend GHC's internal error message representation and expose it to tooling consumers.


Motivation
------------
Currently IDE tools (e.g. `Haskell IDE Engine
<https://github.com/haskell/haskell-ide-engine>`_) have to rely parsing GHC's
human-readable error messages to glean even a basic understanding of the nature
of the error. Not only is this parsing fragile, it also severely limits the
sorts of information the tool can gather about the error. This is unfortunate
since, as GHC ventures further into the territory of dependent types, error
messages grow in complexity and size.

Other languages (e.g. Idris, with their Emacs mode) have `demonstrated
<https://www.youtube.com/watch?v=m7BBCcIDXSg>`_ [1]_ that enriching error message
representations with embedded AST elements can open the door to useful
interactions between the user, IDE tools, and the compiler. Let's open this
door.

Note that this proposal is **not** about changing the default presentation of
error messages produced by the ``ghc`` executable. Rather, it merely discusses
the plumbing necessary to enable downstream consumers (e.g. REPL shells, editor
extensions, and language servers) to make these changes on their own.

Also see `GHC #8809 <https://gitlab.haskell.org/ghc/ghc/issues/8809>`_.

.. [5] While this proposal took a great deal of inspiration from the work in
       this area done by the Idris community, the proposal differs from their
       approach in a key detail. See the section `Why not scoped annotations?`_
       for further discussion on this.


Proposed Change Specification
-----------------------------
Error messages in GHC are currently represented as simple pretty-printer
documents (note: this is slightly simplified for the sake of discussion),
accompanied by a class to give a common name to the function that turns
all sorts of Haskell values into documents::

    -- | A pretty-printer document.
    data SDoc = ...

    class Outputable a where
      ppr :: a -> SDoc

    -- | An error message.
    type ErrMsg = SDoc

We propose to refactor this into ::

    -- | A pretty-printer document.
    data SDoc' a
      = ...
      | Pure a

   -- | 'fmap' transforms annotations
   instance Functor SDoc'
   -- | 'pure' creates a document from an annotation (using 'Pure'),
   --   @'(<*>)' = 'ap'@
   instance Applicative SDoc'
   -- | '(>>=)' performs substitution of annotations with new subdocuments in
   --   place of the 'Pure' nodes
   instance Monad SDoc'

   -- | A common name for annotation-agnostic pretty-printing functions
   class Outputable a where
     -- note how ppr's return type is polymorphic
     -- in the annotation type
     ppr :: a -> SDoc' b

   -- | A document containing no annotations whatsoever, can
   --   be used in code generation for example.
   type SDoc = SDoc' Void

   -- | Remove all annotations
   stripAnnotations :: SDoc' a -> SDoc

   -- | Turn all annotations into purely textual contents
   renderAnnotations :: (a -> SDoc) -> SDoc' a -> SDoc

   -- | Render annotations using the annotation type's Outputable
   --   instance.
   pprAnnotations :: Outputable a => SDoc' a -> SDoc

   -- | A value that can be embedded in an error message.
   data ErrorMessageItem = ...

   -- | An error message, which is a document with annotations
   --   described by the 'ErrorMessageItem' type, all attached to their
   --   typical GHC textual representation (an 'SDoc', no annotations)
   type ErrMsg = SDoc' (ErrorMessageItem, SDoc)

In this scheme ``SDoc'`` would be a free-monad-style pretty-printer document
(e.g. similar to that provided by ``wl-pprint-extras``).

Each producer of ``SDoc'`` (compiler errors, Haskell/Core/STG/Cmm/LLVM/Assembly
dumps, etc) would be free to pick its own annotation type and eventually turn
the said annotations into textual contents or hand the rich document as-is to
some other code. Or alternatively decide that it doesn't need any annotation
and work with ``SDoc`` values directly. With ``SDoc`` and ``ErrMsg``  being
type synonyms of ``SDoc'``, specialized to particular annotation types, we can
still use all the annotation-agnostic combinators for buiding up documents,
including all the ``Outputable`` instances we have in the compiler.

The ``Functor``, ``Applicative`` and ``Monad`` instances let us transform
the annotations and combine them into possibly larger ones, when documents
(and their annotations) are built from various small chunks that have their
own rich meaning.

The ``ErrorMessageItem`` type is a sum type including a variety of
elements frequently found in error messages that tooling users would find
useful to have available in structured form.

There are a number of things that might be included in this type but the
initial cases we propose here fall into a few categories which we will
address below.

Haskell AST elements
~~~~~~~~~~~~~~~~~~~~

These are the elements of the program we are compiling. For instance ::

    data ErrorMessageItem
      = ...
      | EIdentifier Id      -- An identifier
      | EExpr       HsExpr  -- A general expression
      | EType       HsType  -- A type

Error message idioms
~~~~~~~~~~~~~~~~~~~~

In addition, we can also capture common idioms found in error messages. Many of
these are already produced centrally by helpers in GHC's ``TcErrors`` module.
For instance, consider the case of the all-too-frequent expected-actual error ::

.. code-block:: none

    Test.hs:7:7: error:
        • Couldn't match expected type ‘Int’ with actual type ‘[Char]’
        ...

This could be represented as ::

    data ErrorMessageItem
      = ...
      | EExpectedActual { expectedType :: Type -- ^ what the typechecker expected
                        , actualType   :: Type -- ^ what the typechecker actually found
                        }

Likewise, the message,

.. code-block:: none

    hi.hs:5:5: error:
        • Variable not in scope: foldl'
        • Perhaps you meant one of these:
            ‘foldl’ (imported from Data.Foldable),
            ‘foldl1’ (imported from Prelude), ‘foldr’ (imported from Prelude)
          Perhaps you want to add ‘foldl'’ to the import list
          in the import of ‘Data.Foldable’ (hi.hs:3:1-28).

This could be represented as ::

    data ErrorMessageItem
      = ...
      | ENotInScope { badName               :: OccName
                    , suggestedAlternatives :: [Name]
                    }

Refactoring Actions
~~~~~~~~~~~~~~~~~~~

Additionally, we could further include more action-oriented items. For
instance, in numerous places GHC suggests enabling a language extension:

.. code-block:: none

    hi.hs:8:33: error:
        Illegal operator ‘+’ in type ‘n + 1’
          Use TypeOperators to allow operators in types

This could be represented as ::

    data ErrorMessageItem
      = ...
      | ESuggestExtension LanguageExtension

Likewise, suggestions of changes to ``import`` statements, e.g.

.. code-block:: none

    hi.hs:5:5: error:
        • Variable not in scope: foldl'
          ...
          Perhaps you want to add ‘foldl'’ to the import list
          in the import of ‘Data.Foldable’ (hi.hs:3:1-28).

can be encoded as ::

    data ErrorMessageItem
      = ...
      | ESuggestAddedImport SrcSpan Name  -- source span of import statement
                                          -- and suggested Name to import


An Example
~~~~~~~~~~

In general error messages will be built from plain pretty-printer documents
with embedded ``ErrorMessageItem``\s. For instance, consider the error

.. code-block:: none

    hi.hs:5:5: error:
        • Variable not in scope: foldl'
        • Perhaps you meant one of these:
            ‘foldl’ (imported from Data.Foldable),
            ‘foldl1’ (imported from Prelude), ‘foldr’ (imported from Prelude)
          Perhaps you want to add ‘foldl'’ to the import list
          in the import of ‘Data.Foldable’ (hi.hs:3:1-28).

This might be built by GHC as ::

    embed (EErrorHeader $span Nothing)
    <> embed (ENotInScope $foldl' [ $foldl, $foldl1 ])
    <> embed (ESuggestAddedImport $import_span $foldl' [ $foldl, $foldl1 ])

where ``$foo`` denotes the GHC AST item for ``foo`` and ``embed`` lifts an
``ErrorMessageItem`` into an ``SDoc'``::

    embed :: ErrorMessageItem -> SDoc' ErrorMessageItem
    embed = pure

Effect and Interactions
-----------------------
By introducing rich semantic content into error messages and exposing these
documents via the GHC API, we allow tooling authors significantly more
flexibility in presenting (and automatically fixing) compile-time errors.
We list a few compelling applications below (roughly in order of complexity):

* A REPL front-end might implement color-coded output, choosing a token's
  color by its syntactic class (e.g. type constructor, data constructor, or
  identifier), its name (e.g. all occurrences of ``foldl`` shown in red,
  occurrences of ``concat`` shown in blue), or some other criterion entirely.

* A REPL front-end or IDE tool might allow users the ability to interactively
  navigate a type in a type error and, for instance, allow the user to
  interactively expand type synonyms, show kind signatures, etc.

* An IDE tool might ask GHC to defer expensive analyses typically done
  during error message construction (e.g. `computing valid hole fits
  <https://gitlab.haskell.org/ghc/ghc/issues/16875#note_210045>`_) and instead
  query GHC for the analysis result asynchronously (or even only when
  requested by the user), shrinking the edit/typechecking iteration time.

* An IDE tool might use the action-items (e.g. ``ESuggestExtension`` and
  ``ESuggestAddedImport`` above) to present automated refactoring options to
  the user.


Costs and Drawbacks
-------------------

Judging from a prototype implementation undertaken a few years ago, the impact
of embedding structured data instead of producing pretty-printer documents is
quite minimal, but not trivial either. The idioms which we are trying to
represent are implemented in helper functions in ``TcErrors``, but we use
or mention ``SDoc`` explicitly in various subsystems of GHC, so a complete
implementation of the proposal would require updating type signatures that
mention ``SDoc``, to make them more generic and take an ``SDoc' a``, wherever
appropriate.

One unexpected challenge in implementing the prototype was the difficulty of
finding or adapting a pretty-printer library with the desired monadic
annotation semantics that does not break the formatting of GHC's error message
output. A previous attempt at using the ``wl-pprint-extras`` library found
that GHC's error messages generally include a great deal of superfluous
whitespace which is eliminated by the ``pretty`` library yet not by most other
libraries (see also this `prettyprinter issue
<https://github.com/quchen/prettyprinter/issues/34>`_).

The greatest challenge in this proposal is designing a vocabulary of
``ErrorMessageItem``\s that can be usefully and unambiguously interpreted by
error message consumers. We propose a few simple items in the design discussion
above, but we only scratch the surface of what could be encoded and what might
be useful. We hope that the discussion that arises from this proposal will shed
light on additional items. Moreover, we anticipate that the vocabulary will
grow in time as new tooling applications are found.

A smaller but very concrete challenge is figuring out how to give users
of the annotation mechanism (GHC API users, e.g IDE/tooling developers) a hook
into the processing of error documents when they're reported (and possibly
in other places where our prototype implementation had to "strip off"
annotations) because of the lack of such a hook. Our prototype just applies
the simplest annotation stripping function, which essentially replaces all
annotations with the ``SDoc`` they're paired with, while a user supplied
function of type ``a -> SDoc`` for a suitable annotation type would let GHC
adapt the final document, depending on the needs, under all circumstances.
One close solution is the ``log_action`` field in ``DynFlags``, but it
currently takes an ``SDoc``, and is probably not the only "document consumer"
that would have to be updated. Any specific choice of annotation type would
make it useless for "clients" that need another one (or none).

Alternatives
------------

Two close variations on the proposal's design have been examined:

* Make ``SDoc`` be ``SDoc' ErrorMessageItem``: this has the disadvantage of
  immediately making a bunch of types "wrong", if implemented. Indeed, a few
  code generators produce ``SDoc`` values when generating assembly, and
  claiming that such documents can possibly embed ``ErrorMessageItem``
  annotations seems confusing.

* Make ``SDoc`` be ``forall a. Annotation a => SDoc' a``: this forces the use
  of an additional extension in all the modules that mention ``SDoc``, but
  possibly offloads the rendering of annotations to typeclass instances.
  (One could consider ``Annotation = Outputable``.)


There are a few alternatives routes, too:

* Continuing representing error messages as plain pretty-printer documents.
  We think this would be a shame as it would IDE/tooling developers

* Represent error messages as fully structured data using a large sum
  type. Core GHC contributors have in the past opposed this approach on
  account of maintanence difficulty. We agree and further think that the
  proposal laid out above can capture most of the precision of a fully
  structured representation with a fraction of the maintanence overhead.

* Adopt the above plan, but using a "scoped annotations"-style instead of a
  free monad pretty-printer.  See the `Why not scoped annotations?`_ section
  below.

* Richard Eisenberg has `suggested
  <https://gitlab.haskell.org//ghc/ghc/issues/8809#note_101739>`_ a
  dynamically-typed variant of the above idea. That is, ``SDoc`` would be
  extended with a constructor: ::

      data SDoc where
          = ...
          | forall a. (Typeable a, Outputable a) => Embed a

  This gives us a slightly more flexible representation at the expense of
  easy of consumption. In particular, it will be much harder for consumers
  to know what sort of things it should expect in a document.

.. _scoped-annotations:

Why not scoped annotations?
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Idris has a slightly different document representation from what we propose
here. Specifically, it relies on what we will refer to as "scoped annotations".
Under this model the ``SDoc`` type is similarly parametrized with an annotation
type but the ``embed`` combinator is replaced by ``annotate`` ::

    annotate :: a -> SDoc a -> SDoc a

That is, an annotation "covers" a subdocument. While convenient for some
applications, we think that this model is restrictive and potentially confusing
for consumers.

Specifically, with an ``annotate``-style document the consumer must consider the
possibility that there is information in the sub-document that is *not*
conveyed in the annotation. For instance, we might produce a document like: ::

   let aVar :: Var
       aVar = ...
   in annotate aVar (text "the variable" <+> ppr aVar <+> text "is not in scope")

How should a consumer present this document to the user? They have three options:

* They could throw away the sub-document, but this would lose critical
  information about the error (namely that the named variable is not in scope).
* They could display *just* the subdocument, but annotation has bought us
  nothing over the status quo.
* They could display the submodule but modify it slightly based on the
  annotation (e.g. rendering it as a hyperlink, changing its text styling,
  etc).

Because of this potential for information loss when discarding the subdocument,
the ``annotate``-style pretty-printer model severely limits
the sorts of presentations that a consumer can choose: they are forced to
*somehow* display the sub-document, regardless of whether it contributes any
new information to the user.

By contrast, with an ``embed``-style document it is clear that the embedded
value represents a piece of the document which the consumer is free to
render in any way it sees fit. All of the information relevant to the message
is guaranteed to be in the embedded value.

However, some applications (e.g. kythe_) are more natural to write
with scoped-style annotations. For this cases it is possible to emulate scoped
annotations with ``embed``-style document, by attaching the document and the
annotation together, as part of a "bigger", compound annotation:

.. _kythe: https://github.com/mpickering/core-kythe

    -- using our embed-style SDoc to store both annotations as well
    -- as the sub-documents that gets annotated with those values
    newtyped ScopedSDoc a = ScopedSDoc
      { getScopedSDoc :: SDoc' (a, ScopedSDoc a) }

    -- scoped annotation function
    scopedAnn :: a -> ScopedSDoc a -> ScopedSDoc a
    scopedAnn a d = Scoped $ pure (a, d)

Likewise, if we were using a scoped annotation friendly representation, say
``SDoc2``, we would be able to emulate non-scoped annotations by scoping
our annotations over empty documents:

    -- assuming SDoc2 has 'annotate :: a -> SDoc2 a -> SDoc2 a'
    ann :: a -> SDoc2 a
    ann a = annotate a empty


Unresolved Questions
--------------------

As described in the "Costs and Drawbacks" section above, a number of questions
regarding the design of the ``ErrorMessageItem`` type remain open.



Implementation Plan
-------------------

Well-Typed LLP will implement this proposal with financial support from
Richard Eisenberg, under NSF grant number 1704041. Having better support
for error messages will smooth the way toward dependent types (the main
objective of that NSF grant).

